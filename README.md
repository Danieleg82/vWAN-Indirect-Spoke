# Virtual WAN and Indirect-spoke model: keeping or re-architecting?


If you are basing the design of your current Azure connectivity topology on Virtual WAN (vWAN) and in detail on the so-called **“Indirect spoke”** model, you’re probably asking yourself if today this is still a valid/modern model to use 😊

Let’s do a step back first…

Once upon a time you decided that vWAN was the right connectivity service for your organization, but a direct integration of security solution (Firewalls) inside the vHUB was not fitting your needs…hence you decided to adopt the so-called INDIRECT SPOKE model:

**1)	A DIRECT spoke model**

<Pic of vWAN direct spoke model>

**2)	An INDIRECT spoke model**

<Pic of vWAN indirect model>

In the INDIRECT spoke model, you basically “move” the FW solution outside the boundaries of the vHUB, and you connect your spoke VNETs to the vHUB “indirectly”, through a Transit HUB VNET.

What moved you using Indirect-spoke?
What was the reason behind this choice?

In the recent past, some of the common reasons for moving you toward the usage of INDIRECT spoke model could have been the following:
-	The kind of Firewall solution you like could not be integrated inside vWAN HUB
-	The kind of Firewall solution you wanted to use could be integrated within vWAN HUB, but could not scale horizontally the same way as its VM-based solution could, or some specific feature was not present in the integrated version.
-	You had necessity to perform inter-region traffic inspection through firewall, and **Routing Intent** (https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-routing-policies) feature was not there yet available 
-	You were going to break the limitation of 1k BGP routes advertisable from Azure to Onpremise through ExpressRoute, due to presence of massive amount of spoke VNETs to be directly connected to your vWAN HUB
The INDIRECT spoke model could offer possibility of FW filtering in Azure for all the traffic data-paths, except for Branch2Branch traffic.

<pic of INDIRECT pre RI>

The INDIRECT spoke model helped with all the above challenges, but this kind of setup also has some disadvantages, like:
-	The loss of integration between the address ranges of your spoke VNETs and the connectivity HUB, which introduces the necessity of leveraging routes’ aggregation + static routes in vWAN, or BGP endpoints
-	The need of Route Tables (UDRs) within your spoke VNETs used to redirect traffic toward your transit VNET’s virtual appliances.
-	Extra costs for VNET peerings


Today, after the full availability of the **Routing Intent** feature and the possibility to integrate a lot of the most common Firewall brands inside vWAN HUBs (https://learn.microsoft.com/en-us/azure/virtual-wan/about-nva-hub#partners) are you still running an optimized network topology?
Can you always safely move to an integrated solution?

<pic of post RI DIRECT model>

Well…as it often happens…there’s no easy answer to this question, and the world is not black or white 😊
Let’s try to summarize all the possible scenarios where moving toward DIRECT spoke / integrated model is possible and optimal for you!

# IS IT TIME TO RECONSIDER MY TOPOLOGY?

Without any doubt, moving toward DIRECT spoke connectivity model brings a lot of simplifications.
-	Your firewall solution is integrated with the vWAN control plane
-	The IP ranges of your spoke VNETs are automatically propagated to your branches and between HUBs, hence you don’t need to leverage on routes’ aggregations (static routes) nor you need to configure BGP between the vHUBs and any appliance.
-	You reduce VNET peering costs
-	With **Routing Intent** feature, you can leverage automatic Inter-HUB and Branch-2-Branch firewall filtering
…but this doesn’t mean that such migration is always possible, nor always recommended, depending on your needs.

## SCENARIO 1: MY FAVOURITE FIREWALL BRAND CANNOT BE INTEGRATED WITH vWAN

<Pic 3rd party firewalls in spoke>

If – for any reason – you need to use a firewall solution that is not in the list of the ones which you can integrate inside vHUB (https://learn.microsoft.com/en-us/azure/virtual-wan/about-nva-hub#partners) , the INDIRECT spoke model is likely still plausible and valid for you.

*BENEFITS*:
-	You can use the FW solution compatible with your needs
-	You can scale your FW (or generally speaking, NVA) solution the way you prefer, either horizontally ( = more VMs in your cluster)  or vertically ( = changing your VMs SKU when possible)
-	You can leverage the flexibility of the TRANSIT VNET to build different FW clusters for different purposes – if needed (i.e. E/W cluster + N/S cluster), including connectivity appliances (i.e. 3rd party SDWAN / IPSEC gateways)

*THINGs TO KEEP IN MIND*:
-	If you are **not** using _vWAN BGP endpoints_ (https://learn.microsoft.com/en-us/azure/virtual-wan/scenario-bgp-peering-hub) you need to be able to summarize/aggregate the IP ranges of the spoke VNETs connected to your transit HUB as static routes in the Default Route table of the vWAN HUB. In such case, keep in mind that **Routing Intent** will not be available at all on your vHUB: in fact, **Routing Intent is incompatible with any kind of static route configured in the Default Route Table of a vHUB** (https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-routing-policies#knownlimitations )
-	If you are using vWAN BGP endpoints, you either need an **integration with Azure Route Server (ARS)*** [see section below], or you still must be able to summarize/aggregate the IP ranges of the spoke VNETs connected to your transit HUB as BGP advertisement to the vHUB itself. With the BGP integration, if you don’t use any static route in vHUB, the **Routing Intent** feature is still an option you can use for Branch2Branch traffic filtering. With BGP integration, your firewall cluster can be **active-passive only**, since vHUB today (Nov. 2023) doesn’t still support BGP _CUSTOM NEXT HOP_ feature.
-	You will always need Route tables (UDRs) in your spoke VNETs, unless you deploy the **integration with Azure Route Server (ARS)*** [see section below]

## SCENARIO 2: MY FIREWALL BRAND CAN BE INTEGRATED, BUT I WANT FULL CONTROL ON IT

<Pic 3rd party firewalls in HUB>

When you decide to leverage a FW solution which is integrated inside a vHUB, from one side you have all the facilitations of the integration, but from the other side you do not have the kind of control on the FW solution you could have with a VM-based deployment of the same in a spoke VNET. (see https://learn.microsoft.com/en-us/azure/virtual-wan/about-nva-hub )
If you need the full control on your FW cluster solution, the INDIRECT spoke model is likely still plausible and valid for you.
_BENEFITs:_
- You can scale your FW (or generally speaking, NVA) solution the way you prefer, either horizontally ( = more VMs in your cluster)  or vertically ( = changing your VMs SKU when possible)
-	You can leverage the flexibility of the TRANSIT VNET to build different FW clusters for different purposes – if needed (i.e. E/W cluster + N/S cluster), including connectivity appliances (i.e. 3rd party SDWAN / IPSEC gateways)

_THINGs TO KEEP IN MIND:_

Same as SCENARIO 1


## SCENARIO 3: I USE EXPRESSROUTE AND I PLAN TO INTEGRATE HUNDREDS OF SPOKE VNETs WITH MY CONNECTIVITY LAYER

If you’re an ExpressRoute user, the amount of routes advertised from Azure toward onprem is your enemy.
Today, the limit is 1k routes.
Every address space of every spoke VNET connected to vHUB represents one advertised route.
In a scenario where your Azure environment is going to evolve toward hundreds (or more) of spoke VNETs connected to your vHUB (or vHUBs), you risk getting closer to the 1k advertised routes’ limit.
This is even worse – of course – for scenarios where you have vWAN branch2branch enabled and you’re receiving as well BGP routes from branches, together with inter-hub routes.
<Pic of hundreds spokes vHUB>
In vWAN we have a feature  (currently in Preview) called **Route Maps** (https://learn.microsoft.com/en-us/azure/virtual-wan/route-maps-how-to)  which will alleviate such issue when it will go generally available, but today we can’t still rely on that.
In similar situations, the INDIRECT spoke model is likely still plausible and valid for you.

_BENEFITs:_
-	The vHUB (hence ExpressRoute circuit) will have no visibility over the network ranges of the spoke VNETs connected to your transit HUB, hence you won’t risk to exceed 1k routes
-	You will be able to advertise over ExpressRoute only the aggregates of routes representing the super-net of your spoke networks (levering either BGP endpoint or static routes in vWAN)
-	If you use BGP endpoint (no static routes in vWAN) this solution is compatible with the usage of an integrated vWAN FW solution and Routing Intent


_THINGs TO KEEP IN MIND:_

Same as SCENARIO 1


## SUMMARY MATRIX
The following matrix represents a possible approach toward DIRECT or INDIRECT spoke models depending on some common combinations of requirements.
Note that what reported is valid today (Nov. 2023) and may be obsolete tomorrow…as per any discussion around cloud products 😊

<>
 
# vWAN INTEGRATION WITH AZURE ROUTE SERVER

Somebody told you that the integration between **Azure Route Server** and **vWAN** is impossible?
Well…technically speaking, this is correct…but we have some workarounds that could make this possible, and this will simplify a lot the routing scenarios of INDIRECT spoke models.
Look at this:

<Pic of vWAN + ARS>

In this scenario:
1.	We create a dedicated VNET containing ARS.
2.	We peer such VNET with all the spoke VNETs (with the GW transit option enabled).
3.	We as well peer such VNET with the intermediate Transit VNET.
4.	We create a BGP peering between your virtual appliance in the Transit VNET and the ARS.
5.	We create a BGP peering between your virtual appliance in the Transit VNET and the vHUB
What happens after?
-	All the address prefixes of your spoke VNETs will be advertised to the NVA, which will readvertise to the vHUB
- All the address prefixes coming from vHUB will be advertised to the NVA, which will readvertise to the ARS

Result of all this?
The result is that you will no longer need UDRs on your spoke VNETs: all the IP ranges coming from vHUB will automatically have the IP of your NVA (or the load balancer in front of it, if it’s a cluster) as nexthop.
Similarly, from the point of view of the vHUB, all the ranges of the spokes will automatically have the NVA IP (or load balancer IP) as nexhop
As discussed previously, today you will be able to leverage this kind of setup only considering Active/Standby clusters of NVAs, since the BGP-endpoint technology between NVA and vHUB doesn’t support BGP custom-next-hops definitions… but this limitation is supposed to disappear within 2024.

Basing on this, we can say that – even in scenarios where you had to stay sticky to the INDIRECT spoke model -  you still have possibility to simplify your routing with BGP and you’re not forced to the use of Static Routes in the vHUB’s route table and routes’ aggregation.
