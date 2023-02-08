# NVA+ARS for AVS Internet connectivity

3 options for [AVS internet connectivity] (https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-design-public-internet-access):

1. **via an AzFW or NVA already hosted in Azure**

2. Via [public IP at NSX-T level](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-public-ip-nsx-edge) (allowing to deploy a 3P NVA in the AVS environment)

3. Via [AVS managed SNAT solution](https://learn.microsoft.com/en-us/azure/azure-vmware/enable-managed-snat-for-workloads) (SNAT through an AVS NAT GW) 


This repo is about option 1.

<img width="1164" alt="image" src="https://user-images.githubusercontent.com/110976272/217469277-4172cd03-7fe8-4c2b-ae0d-75ff37a875a1.png">

:arrow_right: Disabling *GW route propagation* on a subnet removes the routes programmed by ARS on that subnet: the default route to the FW needs to be enforced with a UDR on the Spokes

:arrow_right: VNET peering routes are preferred over ARS propagated routes (even if ARS routes are more specific): UDRs on the GW subnet are required to force the traffic to the Spoke VNETs through the FW
