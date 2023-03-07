# Transit & Internet connectivity for AVS: ARS + FW NVA or ARS + AzFW

There are 3 options for [AVS internet connectivity](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-design-public-internet-access):

1. **via an AzFW or NVA already hosted in Azure**

2. Via [public IP at NSX-T level](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-public-ip-nsx-edge) (allowing to deploy a 3P NVA in the AVS environment)

3. Via [AVS managed SNAT solution](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-managed-snat-for-workloads) (SNAT through an AVS NAT GW) 

This repo is about summarising the 2 sets of designs available for option 1 in a H&S topology.

[1. SINGLE HUB VNET DESIGN]

&emsp;[1.1. SINGLE HUB VNET & FW NVA]

&emsp;[1.2. SINGLE HUB VNET & AzFW]

[2. HUB VNET + AVS TRANSIT VNET DESIGN]

&emsp;[2.1. HUB VNET, AVS TRANSIT VNET & NVAs]

&emsp;[2.2. HUB VNET, AVS TRANSIT VNET & AzFW]
##
# 1. Single Hub VNET design

On-Prem to AVS is managed via Global Reach when available*, with an On-Prem FW for filtering/inspection if needed.

All the other flows are sent through the FW: Spoke to Spoke, On-Prem to Spokes and Spoke to AVS. 

\* *As explained by Adam, this design can offer [On-Prem to AVS transit](https://www.youtube.com/watch?v=x32SNdEaf-Q) should GR not be available*

:arrow_right: In this design, the ARS **only** purpose is to push the default route to AVS.

:arrow_right: Disabling *GW route propagation* on a subnet removes the routes programmed by ARS on that subnet: the default route to the FW needs to be enforced with a UDR on the Spokes.

:arrow_right: VNET peering routes are preferred over ARS propagated routes (even if ARS routes are more specific): UDRs on the GW subnet are required to force the traffic to the Spoke VNETs through the FW.

## 1.1. Single Hub VNET & FW NVA

:warning: Make sure to disable GW route propagation on the internet facing NIC of the FW, to avoid a routing loop.

<img width="911" alt="image" src="https://user-images.githubusercontent.com/110976272/223410598-48f07499-b564-4baa-8488-12fe112b57d8.png">

## 1.2. Single Hub VNET & Azure FW

As the Azure Firewall doesn't speak BGP, a routing NVA* is added to advertise the default route to the ARS for further propagation. This design leverages the [Next-hop IP feature](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) supported by the ARS and that allows the NVA to set the Next-Hop of the route advertised to be the AzFW.

\* *This routing NVA is here only to generate the default route and won't be in the data path.*

This design is also detailed in this [Adam video](https://youtu.be/8CPghVFIR9Q?t=335).

<img width="911" alt="image" src="https://user-images.githubusercontent.com/110976272/223411045-f0d10635-8010-41f8-b2b2-cf622aa04461.png">

# 2. Hub VNET + AVS Transit VNET design

With this design, FW inspection can be adjusted. Ex: On-Prem <-> Spokes can go direct, Spokes <-> AVS is filtered.

Just like in the scenarios above, On-Prem to AVS transit can be achieved in case Global Reach is not available.

:heavy_plus_sign: Limited need of UDRs/No UDRs. Depending on the traffic flitering required, disabling *GW route propagation* on the Spoke subnets is no longer required.

:heavy_minus_sign: Additional infrastructure required: a dedicated AVS transit VNET, a 2nd ERGW, a 2nd ARS and eventually a 2nd NVA.

## 2.1. HUB VNET + AVS TRANSIT VNET & NVAs

<img width="1127" alt="image" src="https://user-images.githubusercontent.com/110976272/223508772-23bbbb98-d6ff-404a-90d9-3a4440954c65.png">

ARS1 will:
1. propagate the default route learnt from the FW NVA + the AVS ranges forwarded by the AVS NVA via the FW NVA:
    - to the On-Prem over ER
    - to the Spoke VNETs
2. advertise the OnPrem + the hub and spoke VNET ranges to the FW NVA

ARS2 will:
1. propagate the default route + the On-Prem + the H&S VNET ranges to AVS
2. advertise the AVS ranges to the AVS NVA

2 options for the FW NVA to AVS NVA transit:
1. UDRs (FW NVA: AVS ranges to AVS NVA NIC, AVS NVA: 0/0 to FW NVA NIC)
3. BGP over IPSec/VxLAN if the granularity of the AVS and/or OnPrem routes are required

:arrow_right: If stateful, the NVA instances should be configured as Active/Standby to avoid asymmetric routing. 

For reasons already discussed in another [article](https://github.com/cynthiatreger/az-routing-guide-ep5-nva-routing-2-0#532-chained-nvas-ars-and-vxlan) the eBGP session between the FW NVA and the AVS NVA is established within a VxLAN or IPSec tunnel:
- the FW NVA advertises the default route, the On-Prem prefixes and the Hub & spoke ranges to the AVS NVA
- The AVS NVA advertises the AVS ranges to the FW NVA

:warning: GW route propagation should be disabled in the following cases:
- should UDRs be used instead of BGP, on the FW NVA and AVS NVA NICs (subnets) used for inter-hub connectivity
- on the NIC used for internet connectivity: either the FW NIC facing the AVS NVA or a 3rd dedicated NIC

## 2.2. HUB VNET, AVS TRANSIT VNET & Azure Firewall

This design is documented in detail [here](https://github.com/Azure/Enterprise-Scale-for-AVS/tree/main/BrownField/Networking/Step-By-Step-Guides/Expressroute%20connectivity%20for%20AVS%20without%20Global%20Reach) and is useful when the FW doesn't speak BGP, like the Azure Firewall.

<img width="1132" alt="image" src="https://user-images.githubusercontent.com/110976272/223506157-87c702de-4a70-453f-83ae-a98178b93c46.png">

The AVS NVA will:
1. originate and advertise the default route
2. forward the AVS ranges to ARS1
3. for these routes: [update the Next-Hop](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip) to be the Azure Firewall
4. forward the On-Prem and H&S VNET ranges to ARS2 (the Next-Hop remains unchanged and will be the AVS NVA NIC facing the ARS.

:warning: GW route propagation disabled and **a 0/0 UDR pointing to the AzFW** are required on the AVS NVA NIC to ARS1.

ARS1 will:
1. propagate the default route + the AVS ranges learnt from the AVS NVA:
    - to the On-Prem over ER
    - to the Spoke VNETs (with Next-Hop = AzFW the Next-Hop IP featur 
2. advertise the OnPrem + the hub and spoke VNET ranges to the AVS NVA

ARS2 will:
1. propagate the default route + the On-Prem + the H&S VNET ranges to AVS
2. advertise the AVS ranges to the AVS NVA

:arrow_right: that UDRs towards the AVS ranges will be required on the Hub VNet GW subnet to force the On-Prem traffic to AVS through the AzFW.

:warning: make sure to disable GW route propagation on the AVS NVA NIC to ARS1.
GW route propagation should be disabled in the following cases:
- should UDRs be used instead of BGP, on the FW NVA and AVS NVA NICs (subnets) used for inter-hub connectivity
- on the NIC used for internet connectivity: either the FW NIC facing the AVS NVA or a 3rd dedicated NIC

Spoke and OnPrem ranges received via eBGP from ARS2
(not required because of the static default route here, but mandatory in the case Internet breakout is available via AVS
