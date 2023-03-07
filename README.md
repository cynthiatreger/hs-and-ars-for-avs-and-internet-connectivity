# Transit and Internet connectivity for AVS with ARS

This article aims at summarising the sets of designs available for AVS connectivity filtered by a Firewall (3P FW or Azure Firewall), also providing internet breakout, in a Hub and Spoke topology.

There are 3 options for [AVS internet connectivity](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-design-public-internet-access):

1. **via an AzFW or NVA already hosted in Azure**

2. Via [public IP at NSX-T level](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-public-ip-nsx-edge) (allowing to deploy a 3P NVA in the AVS environment)

3. Via [AVS managed SNAT solution](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-managed-snat-for-workloads) (SNAT through an AVS NAT GW) 

This repo is about option 1.

[1. SINGLE HUB VNET DESIGN](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#1-single-hub-vnet-design)

&emsp;[1.1. SINGLE HUB VNET and FW NVA](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#11-single-hub-vnet-and-fw-nva)

&emsp;[1.2. SINGLE HUB VNET and Azure Firewall](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#12-single-hub-vnet-and-azure-firewall)

&emsp;[1.3. Transit without Global Reach](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#13-single-hub-vnet-without-global-reach)

[2. HUB VNET + AVS TRANSIT VNET DESIGN](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#2-hub-vnet--avs-transit-vnet-design)

&emsp;[2.1. HUB VNET, AVS TRANSIT VNET and NVAs](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#21-hub-vnet--avs-transit-vnet-and-nvas)

&emsp;[2.2. HUB VNET, AVS TRANSIT VNET and Azure Firewall](https://github.com/cynthiatreger/transit-and-internet-for-avs-with-ars#22-hub-vnet-avs-transit-vnet-and-azure-firewall)
##
# 1. Single Hub VNET design

On-Prem to AVS is managed via Global Reach when available*, with an On-Prem FW for filtering/inspection if needed. 

\* *In section 1.3. we will see how this current desig ncan be adapted to provide [On-Prem to AVS transit](https://www.youtube.com/watch?v=x32SNdEaf-Q)*

All the other flows are sent through the FW: Spoke to Spoke, On-Prem to Spokes and Spoke to AVS:

:arrow_right: When Global Reach is used for On-Prem to AVS transit, the ARS **only** purpose is to push the default route to AVS.

:arrow_right: Disabling *GW route propagation* on a subnet removes the routes programmed by ARS on that subnet: the default route to the FW needs to be enforced with a UDR on the Spokes.

:arrow_right: VNET peering routes are preferred over ARS propagated routes (even if ARS routes are more specific): UDRs on the GW subnet are required to force the traffic to the Spoke VNETs through the FW.

## 1.1. Single Hub VNET and FW NVA

:warning: Make sure to disable GW route propagation on the internet facing NIC of the FW, to avoid a routing loop.

<img width="876" alt="image" src="https://user-images.githubusercontent.com/110976272/223509584-28517c8d-7ece-49e3-b7b3-a9ff9d3040a2.png">

## 1.2. Single Hub VNET and Azure Firewall

As the Azure Firewall doesn't speak BGP, a routing NVA is added to advertise the default route to the ARS for further propagation. This routing NVA is here only to generate the default route and won't be in the data path

This design leverages the [Next-hop IP feature](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) supported by the ARS and that allows the NVA to set the Next-Hop of the route advertised to be the AzFW.

This design is also detailed in this [Adam video](https://youtu.be/8CPghVFIR9Q?t=335).

<img width="936" alt="image" src="https://user-images.githubusercontent.com/110976272/223523069-7475ff01-b69c-4d9c-9914-ccaf4b5670f7.png">

## 1.3. Single Hub VNET without Global Reach

When Global Reach is not available, Transit can be achieved via the FW in the Hub VNet by configuring additional static routes and UDRs:

- NVA: static routes (could be a supernet) for the AVS ranges to be propagated On-Prem by the ARS.
- (The On-Prem reachability from AVS is covered by the default route)
- Specific UDRs on the GW subnet to force both the AVS and On-Prem traffic through the FW.

This design is also detailed in this [Adam video](https://youtu.be/8CPghVFIR9Q?t=335).

### FW NVA design:

<img width="873" alt="image" src="https://user-images.githubusercontent.com/110976272/223527462-e6b085ff-e988-431f-b4d4-c54d9e908d09.png">

### Azure FW design:

<img width="866" alt="image" src="https://user-images.githubusercontent.com/110976272/223563152-1afdd727-1907-4ecb-9471-f0b08e0b13d1.png">

# 2. Hub VNET + AVS Transit VNET design

With this design, FW inspection can be adjusted. Ex: On-Prem <-> Spokes can go direct while Spokes <-> AVS is filtered.

This design provides On-Prem to AVS transit capalitites by default. However, when Global Reach is available, Global Reach will be preferred.

:heavy_plus_sign: Limited need of UDRs/No UDRs. If Spoke <-> On-Prem filtering is not needed, disabling *GW route propagation* on the Spoke subnets is no longer required.

:heavy_minus_sign: Additional infrastructure may be required: a dedicated AVS transit VNET, a 2nd ERGW, a 2nd ARS and a 2nd NVA.

## 2.1. HUB VNET + AVS TRANSIT VNET and NVAs

<img width="1125" alt="image" src="https://user-images.githubusercontent.com/110976272/223567625-f1d9e591-b658-4b9f-8ad1-2f34f426e468.png">

| resource | actions |
| - | - |
| FW NVA | 1/ originate and advertise the default route, 2/ forward the AVS ranges to ARS1, 3/ forward the On-Prem and H&S VNET ranges to the AVS NVA |
|AVS NVA |1/ advertise the AVS ranges to the FW NVA, 2/ forward the On-Prem and H&S VNET ranges to ARS2 | 
| ARS1 | 1/ propagate the default route learnt from the FW NVA + the AVS ranges forwarded by the AVS NVA via the FW NVA to the On-Prem over ER and to the Spoke VNETs, 2/ advertise the OnPrem + the hub and spoke VNET ranges to the FW NVA |
|ARS2 | 1/ propagate the default route + the On-Prem + the H&S VNET ranges to AVS, 2/ advertise the AVS ranges to the AVS NVA |

:arrow_right: Internet connectivity can be provided either by the FW NIC facing the AVS NVA or a 3rd dedicated NIC on the FW NVA. GW route propagation should be disabled for that NIC.

2 options for the FW NVA to AVS NVA transit:
1. UDRs 
    - on the FW NVA NIC to AVS VNET: AVS ranges to the AVS NVA NIC, 
    - on the AVS NVA NIC to the Hub VNet: 0/0 to the FW NVA NIC
    - disable GW route propagation on these 2 NICs
2. BGP over IPSec/VxLAN if the granularity of the AVS and/or OnPrem routes are required

:arrow_right: If stateful, the NVA instances should be configured as Active/Standby to avoid asymmetric routing. 

For reasons already discussed in a previous [article](https://github.com/cynthiatreger/az-routing-guide-ep5-nva-routing-2-0#532-chained-nvas-ars-and-vxlan) the eBGP session between the FW NVA and the AVS NVA is established within a VxLAN or IPSec tunnel:
- the FW NVA advertises the default route, the On-Prem prefixes and the Hub & spoke ranges to the AVS NVA
- The AVS NVA advertises the AVS ranges to the FW NVA

## 2.2. HUB VNET, AVS TRANSIT VNET and Azure Firewall

This design is documented in detail [here](https://github.com/Azure/Enterprise-Scale-for-AVS/tree/main/BrownField/Networking/Step-By-Step-Guides/Expressroute%20connectivity%20for%20AVS%20without%20Global%20Reach) and is useful when the FW doesn't speak BGP, like the Azure Firewall.

<img width="1125" alt="image" src="https://user-images.githubusercontent.com/110976272/223567770-a6966d8f-a826-4d89-9dbd-ad5242386fb0.png">

| resource | actions |
| - | - |
| AVS NVA | 1/ originate and advertise the default route, 2/ learn the AVS ranges from ARS2 and forward them to ARS1, 3/ for these routes, [the Next-Hop is updated](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) to be the Azure Firewall, 4/ forward the On-Prem and H&S VNET ranges to ARS2 (the Next-Hop remains unchanged and will be the AVS NVA NIC facing the ARS). |
| ARS1 | 1/ propagate the default route + the AVS ranges learnt from the AVS NVA to the On-Prem over ER and to the Spoke VNETs (with Next-Hop = AzFW), 2/ advertise the OnPrem + the H&S VNET ranges to the FW NVA |
|ARS2 | 1/ propagate the default route + the On-Prem + the H&S VNET ranges to AVS, 2/ advertise the AVS ranges to the AVS NVA |

:arrow_right: **a 0/0 UDR pointing to the AzFW** for the internet connectivity and GW route propagation disabled are required on the AVS NVA NIC to ARS1.

:arrow_right: UDRs towards the AVS ranges must be configured on the Hub VNet GW subnet to force the On-Prem traffic to AVS through the AzFW.
