# H&S topologies with ARS for AVS transit and internet breakout

The goal of this article is to list the architectures and designs addressing the following AVS connectivity patterns in a Hub & Spoke topology:
- AVS <=> On-Prem
- AVS <=> Spokes
- AVS to Internet

As represented on the screenshot below, there are 3 options to provide [internet breakout to AVS](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-design-public-internet-access):

1. **via an Azure Firewall or FW NVA already hosted in Azure**

2. Via [public IP at NSX-T level](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-public-ip-nsx-edge) (allowing to deploy a 3P NVA in the AVS environment)

3. Via [AVS managed SNAT solution](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-managed-snat-for-workloads) (SNAT through an AVS NAT GW) 

![avs-internet-connectivity-settings](https://user-images.githubusercontent.com/110976272/224350509-83e7d75d-5880-4385-8831-87b1b05827a9.png)

This repo is about option 1 and how to leverage an Azure Firewall or a FW NVA already used in an Azure VNet environment and extend it to AVS inspection and AVS internet connectivity.

Most of these designs are already documented across multiple sources and have been referenced accordingly in this article.

[1. SINGLE HUB VNET ARCHITECTURE](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#1-single-hub-vnet-architecture)

&emsp;[1.1. SINGLE HUB VNET WITH FW NVA](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#11-single-hub-vnet-with-fw-nva)

&emsp;[1.2. SINGLE HUB VNET WITH AZURE FIREWALL](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#12-single-hub-vnet-with-azure-firewall)

&emsp;[1.3. SINGLE HUB VNET WITHOUT GLOBAL REACH](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#13-single-hub-vnet-without-global-reach)

&emsp;&emsp;&emsp;- [FW NVA (DESIGN 1.1 bis)](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#fw-nva-design-design-11-bis)

&emsp;&emsp;&emsp;- [AZURE FW (DESIGN 1.2 bis)](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#azure-fw-design-design-12-bis)

[2. HUB VNET AND AVS TRANSIT VNET ARCHITECTURE (WITHOUT GLOBAL REACH)](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#2-hub-vnet--avs-transit-vnet-architecture-without-global-reach)

&emsp;[2.1. HUB VNET AND AVS TRANSIT VNET WITH NVAs](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#21-hub-vnet--avs-transit-vnet-and-nvas)

&emsp;[2.2. HUB VNET AND AVS TRANSIT VNET WITH AZURE FIREWALL](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#22-hub-vnet-avs-transit-vnet-and-azure-firewall)
##
# 1. Single Hub VNet architecture

With this architecture, all the flows are sent through the FW for inspection: Spoke <=> Spoke, Spoke <=> On-Prem, Spoke <=> AVS, On-Prem <=> AVS (when Global Reach is not used).

Key deployment elements:

:arrow_right: In the following 1.1 and 1.2 designs, the ARS **only** purpose is to push the default route to AVS.

:arrow_right: The Spoke subnets are configured with *GW route propagation* disabled, hence not receiving the routes programmed by ARS. The default route to the FW needs to be enforced with a UDR on the Spokes.

:arrow_right: VNet peering routes are preferred over ARS propagated routes (even if ARS routes are more specific): UDRs on the GW subnet are required to force the traffic to the Spoke VNets through the FW.

In [section 1.3.](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#13-single-hub-vnet-without-global-reach) we will see how this design can be adapted to offer On-Prem <=> AVS transit and filtering capabilities when [Global Reach is not available](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-global-reach#availability) or not aligned with the customer requirements (On-Prem <=> AVS traffic required to be inspected by a FW in an Azure VNet).

## 1.1. Single Hub VNet with FW NVA

:warning: Make sure to disable GW route propagation on the internet facing NIC of the FW NVA, to avoid a routing loop (to be addressed in a next article).

<img width="900" alt="image" src="https://user-images.githubusercontent.com/110976272/224343018-be509a53-f25e-4c8c-b022-52921d867896.png">

## 1.2. Single Hub VNet with Azure Firewall

As the Azure Firewall doesn't speak BGP, a routing NVA is added to advertise the default route to the ARS for further propagation. This routing NVA is here only to generate the default route and won't be in the data path.

:arrow_right: This design leverages the [Next-hop IP feature](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) supported by the ARS and that allows the NVA to set the Next-Hop of the route advertised to be the AzFW.

<img width="900" alt="image" src="https://user-images.githubusercontent.com/110976272/224343211-9dd4e486-22eb-47c3-a3c2-f8f9a5454ac0.png">

## 1.3. Single Hub VNet without Global Reach

Without Global Reach, as explained in this [video](https://www.youtube.com/watch?v=x32SNdEaf-Q) by Adam, On-Prem <=> AVS transit can be achieved via the FW in the Hub VNet by configuring additional static routes and UDRs (highlighted on the diagrams below):

- NVA: static routes towards the ARS facing NIC for the AVS ranges to be propagated On-Prem by the ARS. These static routes MUST be **supernets** of the specific AVS ranges to avoid a routing loop.
- (The On-Prem reachability from AVS is covered by the default route, or an On-Prem supernet could be statically configured on the NVA and advertised to the ARS.)
- UDRs on the GW subnet to force both the AVS and On-Prem traffic through the FW. These UDRs MUST **specifically** match the On-Prem and AVS ranges.*

\* *The max number of UDRs in an Azure Route table is 400, ie: this design doesn't scale for scenarios where the (# of advertised On-Prem prefixes) > 400 - (# of AVS ranges) - (# of Spoke ranges).*

:arrow_right: Here the role of the ARS is to both push the default route to AVS and enable the On-Prem <=> AVS transit.

### FW NVA design (design 1.1 bis):

> The [internet breakout](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-network-design-considerations#default-route-to-azure-vmware-solution-for-internet-traffic-inspection) and [transit with a supernet design topology](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-network-design-considerations#connectivity-between-azure-vmware-solution-and-an-on-premises-network) designs are publicly documented on the referenced links.

<img width="900" alt="image" src="https://user-images.githubusercontent.com/110976272/224705028-9100ca37-aad3-45f4-9141-1f9d519334b0.png">

### Azure FW design (design 1.2 bis)

> Details of this design can be found in this [video](https://youtu.be/8CPghVFIR9Q?t=335) shared by Adam.

<img width="900" alt="image" src="https://user-images.githubusercontent.com/110976272/224705248-487701f0-26da-49d9-a138-846e4afa4043.png">

# 2. Hub VNet + AVS transit VNet architecture (without Global Reach)

This approach is an alternative to [design 1.1 bis](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#fw-nva-design-design-11-bis) and [design 1.2 bis](https://github.com/cynthiatreger/hs-and-ars-for-avs-and-internet-connectivity/blob/main/README.md#azure-fw-design-design-12-bis) (when Global Reach is either not supported 
or not adapted to the customer requirements). 

The 4 drivers for this architeture are the following:
- unblocking scenarios in which there are thousands of On-Prem prefixes
- keeping the knowledge of the AVS/On-Prem route granularity, for example during the AVS migration phase
- avoiding configuration maintenance complexity
- having the ability if needed/preferred to bypass FW inspection for Spokes <=> On-Prem connectivity*, while mandatory in the single hub design.

\* *If Spoke <=> On-Prem filtering is not needed, disabling *GW route propagation* on the Spoke subnets is no longer required.*

## 2.1. HUB VNet + AVS transit VNet and NVAs

> The public documentation for the transit design part is available on this [link](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-network-design-considerations#transit-spoke-virtual-network-topology).

<img width="1128" alt="image" src="https://user-images.githubusercontent.com/110976272/224393137-6ddcdbd7-d3fa-48b6-bf1b-3527c84be694.png">

| resources | actions |
| - | - |
| FW NVA | 1/ originates and advertises the default route, 2/ forwards the AVS ranges to ARS1, 3/ forwards the On-Prem prefixes and H&S VNet ranges to the AVS NVA |
|AVS NVA | 1/ advertises the AVS ranges to the FW NVA, 2/ forwards the On-Prem and H&S VNet ranges to ARS2 | 
| ARS1 | 1/ propagates the default route learnt from the FW NVA + the AVS ranges forwarded by the AVS NVA via the FW NVA to the On-Prem over ER and to the Spoke VNets, 2/ advertises the OnPrem + the hub and spoke VNet ranges to the FW NVA |
|ARS2 | 1/ propagates the default route + the On-Prem + the H&S VNet ranges to AVS, 2/ advertises the AVS ranges to the AVS NVA |

:arrow_right: Internet connectivity can be provided either by the FW NIC facing the AVS NVA or a 3rd dedicated NIC on the FW NVA. GW route propagation should be disabled for that NIC.

2 options for the FW NVA to AVS NVA transit:
1. **UDRs**
    - on the FW NVA NIC to AVS VNet: AVS ranges to the AVS NVA NIC, 
    - on the AVS NVA NIC to the Hub VNet: 0/0 to the FW NVA NIC
    - disable GW route propagation on these 2 NICs
2. **BGP over IPSec/VxLAN** if the granularity of the AVS and/or OnPrem routes are required

:arrow_right: If stateful, the NVA instances should be configured as Active/Standby to avoid asymmetric routing. 

For reasons already discussed in a previous [article](https://github.com/cynthiatreger/az-routing-guide-ep5-nva-routing-2-0#532-chained-nvas-ars-and-vxlan) the eBGP session between the FW NVA and the AVS NVA is established within a VxLAN or IPSec tunnel:
- the FW NVA advertises the default route, the On-Prem prefixes and the Hub & spoke ranges to the AVS NVA.
- The AVS NVA advertises the AVS ranges to the FW NVA.

## 2.2. HUB VNet, AVS transit VNet and Azure Firewall

> This design is documented in detail [here](https://github.com/Azure/Enterprise-Scale-for-AVS/tree/main/BrownField/Networking/Step-By-Step-Guides/Expressroute%20connectivity%20for%20AVS%20without%20Global%20Reach) and is useful when the FW doesn't speak BGP, like the Azure Firewall.

<img width="1126" alt="image" src="https://user-images.githubusercontent.com/110976272/224393452-f73b8167-43d1-4a8f-861e-4a8358192118.png">

| resources | actions |
| - | - |
| AVS NVA | 1/ originates and advertises the default route, 2/ learns the AVS ranges from ARS2 and forwards them to ARS1, 3/ for these routes, [the Next-Hop is updated](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) to be the Azure Firewall, 4/ forwards the On-Prem and H&S VNet ranges to ARS2 (the Next-Hop remains unchanged and will be the AVS NVA NIC facing ARS2). |
| ARS1 | 1/ propagates the default route + the AVS ranges learnt from the AVS NVA to the On-Prem over ER and to the Spoke VNets (with Next-Hop = AzFW), 2/ advertises the OnPrem + the H&S VNet ranges to the FW NVA |
|ARS2 | 1/ propagates the default route + the On-Prem + the H&S VNet ranges to AVS, 2/ advertises the AVS ranges to the AVS NVA |

:arrow_right: **A 0/0 UDR pointing to the AzFW** for internet **and On-Prem** connectivity is required on the AVS NVA NIC facing ARS1 (ARS1 advertises the On-Prem prefixes to the AVS NVA via BGP but these prefixes are not enforced on the AVS NVA NIC because GW Transit is disabled between the Hub VNnet and AVS transit VNet). 

:arrow_right: In addition, GW route propagation must be disabled on the AVS NVA NIC to ARS1 to avoid the routing loop that would be programmed by ARS2.

:arrow_right: UDRs towards the AVS ranges must be configured on the Hub VNet GW subnet to force the On-Prem traffic to AVS through the AzFW.
