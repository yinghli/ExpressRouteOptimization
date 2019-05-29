## ExpressRouteOptimization

Multiple ExpressRoute Circuit optimization. </br>

Now Azure support multiple ExpressRoute PoP at same region for site level resilience and support multiple ExpressRoute Circuit connect to same VNET from same location. Customers is considering below topology for HA. </br>

For example, at Azure Southeast Region, there are two ExpressRoute peering location. One is at Equinix and the other is at GlobalSwitch. </br>

![](https://github.com/yinghli/ExpressRouteOptimization/blob/master/Topology.jpg)

In this design, customer had VNET at Southeast Asia Region and setup GatewaySubnet and provision an ExpressRoute Gateway. Customer setup ExpressRoute circuit at peering location Singapore and another circuit2 at peering location Singapore2. After that, customer can link ExpressRoute gateway to different circuit for high redundancy. </br>
In this design, either physical link down or ExpressRoute site down will not impact customer end to end network connectivity. </br>
From routing perspective, by default, customer VM effective route table will see each IP prefix have four ECMP next-hop from MSEE. And customer edge router will see the same each Azure VNET address space route have four next-hop from MSEE. This is because both ExpressRoute circuit have two MSEE and each of them running separate BGP session with customer edge router and ExpressRoute gateway. Each MSEE will use BGP ASN 12076 to setup BGP neighbor. </br>

# From Azure to On-prem route optimization. 
