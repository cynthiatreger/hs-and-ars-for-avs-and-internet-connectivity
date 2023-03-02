# NVA+ARS for AVS Internet connectivity

3 options for [AVS internet connectivity] (https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-design-public-internet-access):

1. **via an AzFW or NVA already hosted in Azure**

2. Via [public IP at NSX-T level](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-public-ip-nsx-edge) (allowing to deploy a 3P NVA in the AVS environment)

3. Via [AVS managed SNAT solution](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-managed-snat-for-workloads) (SNAT through an AVS NAT GW) 

This repo is about option 1.

# 1. Single Hub VNET for AVS and On-Prem connectivity

On-Prem to AVS is managed via Global Reach.

All the other flows are sent through the FW: Spoke to Spoke, On-Prem to Spokes and Spoke to AVS. 

<img width="1156" alt="image" src="https://user-images.githubusercontent.com/110976272/222504152-7c9e27c0-bd4b-4488-a46c-ef21af09f46e.png">

:arrow_right: Disabling *GW route propagation* on a subnet removes the routes programmed by ARS on that subnet: the default route to the FW needs to be enforced with a UDR on the Spokes

:arrow_right: VNET peering routes are preferred over ARS propagated routes (even if ARS routes are more specific): UDRs on the GW subnet are required to force the traffic to the Spoke VNETs through the FW

:warning: Make sure to disable GW route propagation on the internet facing NIC of the FW, to avoid a routing loop.

# 2. Hub & Spoke & dedicated AVS Transit VNET for AVS connectivity

With this design FW inspection can be adjusted. Ex: On-Prem <-> Spokes can go direct, Spokes <-> AVS is filtered.

:heavy_plus_sign:: No UDRs.

:heavy_minus_sign:: additional costs of a dedicated AVS transit VNET, a 2nd ERGW, 2nd ARS, 2nd NVA.

