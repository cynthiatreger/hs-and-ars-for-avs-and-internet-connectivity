# NVA + ARS for AVS Internet connectivity

3 options for [AVS internet connectivity](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-design-public-internet-access):

1. **via an AzFW or NVA already hosted in Azure**

2. Via [public IP at NSX-T level](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-public-ip-nsx-edge) (allowing to deploy a 3P NVA in the AVS environment)

3. Via [AVS managed SNAT solution](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-managed-snat-for-workloads) (SNAT through an AVS NAT GW) 

This repo is about option 1.

## 1. Single Hub VNET for AVS and On-Prem connectivity

On-Prem to AVS is managed via Global Reach, with possibly an On-Prem FW for filtering/inspection.

All the other flows are sent through the FW: Spoke to Spoke, On-Prem to Spokes and Spoke to AVS. 

<img width="1156" alt="image" src="https://user-images.githubusercontent.com/110976272/222504152-7c9e27c0-bd4b-4488-a46c-ef21af09f46e.png">

:arrow_right: Disabling *GW route propagation* on a subnet removes the routes programmed by ARS on that subnet: the default route to the FW needs to be enforced with a UDR on the Spokes

:arrow_right: VNET peering routes are preferred over ARS propagated routes (even if ARS routes are more specific): UDRs on the GW subnet are required to force the traffic to the Spoke VNETs through the FW

:warning: Make sure to disable GW route propagation on the internet facing NIC of the FW, to avoid a routing loop.

## 2. Hub & Spoke + dedicated AVS Transit VNET for AVS connectivity

With this design FW inspection can be adjusted. Ex: On-Prem <-> Spokes can go direct, Spokes <-> AVS is filtered.

:heavy_plus_sign: No UDRs, No Global Reach.

:heavy_minus_sign: Additional infrastructure required: a dedicated AVS transit VNET, a 2nd ERGW, a 2nd ARS and a 2nd NVA.

<img width="1068" alt="image" src="https://user-images.githubusercontent.com/110976272/222540830-bc3fe588-c6d1-44cb-859f-f75559c7c02d.png">

ARS1 will:
1. propagate the default route learnt from the FW NVA + the AVS range forwarded by the AVS NVA
    - to the On-Prem over ER
    - to the Spoke VNETs
2. advertise the OnPrem + the hub and spoke VNET ranges to the FW NVA

ARS2 will:
1. propagate the default route + the On-Prem + the H&S VNET ranges to AVS
2. advertise the AVS range to the AVS NVA

For reasons already discussed in another [article](https://github.com/cynthiatreger/az-routing-guide-ep5-nva-routing-2-0#532-chained-nvas-ars-and-vxlan) the eBGP session between the FW NVA and the AVS NVA is established within a VxLAN or IPSec tunnel:
1. the FW NVA advertises the default route, the On-Prem prefixes and the Hub & spoke ranges to the AVS NVA
2. The AVS NVA advertises the ABS range to the FW NVA

:arrow_right: The FW being stateful, its instances should be configured as Active/Standby to avoid asymmetric routing. If deployed as Active/Active indtances, the [Next-Hop IP feature](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) should be used on the peering with ARS1 to set the next hop to reach both NVA instances as the IP address of their frontend internal load balancer.

:warning: Make sure to disable GW route propagation on the internet facing NIC of the FW, to avoid a routing loop.
