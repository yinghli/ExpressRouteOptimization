## ExpressRouteOptimization

Multiple ExpressRoute Circuit optimization. </br>

Now Azure support multiple ExpressRoute PoP at same region for site level resilience and support multiple ExpressRoute Circuit connect to same VNET from same location. Customers is considering below topology for HA. </br>

For example, at Azure Southeast Region, there are two ExpressRoute peering location. One is at Equinix and the other is at GlobalSwitch. </br>

![](https://github.com/yinghli/ExpressRouteOptimization/blob/master/NewTopology.jpg)

In this topology, customer had VNET at Southeast Asia Region and setup GatewaySubnet and provision an ExpressRoute Gateway. Customer setup ExpressRoute circuit at peering location Singapore and another circuit2 at peering location Singapore2. After that, customer can link ExpressRoute gateway to different circuit for high redundancy. </br>

In this design, either physical link down or ExpressRoute site down will not impact customer end to end network connectivity. </br>

From routing perspective, by default, customer VM effective route table will see each IP prefix have four ECMP next-hop from MSEE. And customer edge router will see each Azure VNET address space route have four next-hop from MSEE. This is because both ExpressRoute circuit have two connection to MSEE and each of them running separate BGP session with customer edge router and ExpressRoute gateway. Each MSEE will use BGP ASN 12076 to setup BGP neighbor. </br>

## From Azure to On-prem route optimization. 
Longest prefix match is always working. If customer advertise 10.0.0.0/24 to ER circuit1 and advertise 10.0.0.0/23 to ER circut2. For packet destination is 10.0.0.1, Azure will choose 10.0.0.0/24 next hop as longest prefix match. </br>

On Azure side, customer can adjust each ER connection BGP weight on ExpressRoute gateway. Higher weight ER connection will be chosen as preferred path. This will influence traffic from Azure to On-Prem direction. </br>
```
$connection = Get-AzVirtualNetworkGatewayConnection -Name "MyVirtualNetworkConnection" -ResourceGroupName "MyRG"
$connection.RoutingWeight = 100
Set-AzVirtualNetworkGatewayConnection -VirtualNetworkGatewayConnection $connection
```
Customer can also optimize this by leveraging BGP attribute. Azure support AS-PATH prepending. For private peering, customer can prepend any ASN to influence Azure side route selection. For same IP prefix, shorter AS-PATH will be chosen as preferred path. </br>

In this example, customer will advertise two prefixes to Azure. 3.3.3.3/32 will prepend 64512 and 64513 ASN. 3.3.3.4/32 will be advertised as normal. </br>
```
router bgp 65000
 bgp log-neighbor-changes
 network 3.3.3.3 mask 255.255.255.255
 network 3.3.3.4 mask 255.255.255.255
 neighbor 13.1.1.1 remote-as 12076
 neighbor 13.1.1.1 route-map prependAS out
!
ip prefix-list toAzure seq 5 permit 3.3.3.3/32
!
route-map prependAS permit 10
 match ip address prefix-list toAzure
 set as-path prepend 64512 64513
!
route-map prependAS permit 20
```
On Azure side, route 3.3.3.3/32 will have two candidate next hop. One AS-PATH is  <12076, 65000> , the other is <12076, 65000, 64512, 64513>. So route with AS-PATH <12076, 65000> will be chosen. </br>

3.3.3.4/32 have two equal cost next hop, both AS-PATH is <12076, 65000>. </br>

## From On-prem to Azure route optimization. 
By default, Microsoft will advertise same IP prefix on each BGP session, customer can setup all BGP attribute at their end to determine which path is preferred. Like Weight, Local Preference, AS-PATH, Community and so on. </br>

In the previous topology, if customer wants to use ER circuit to access 10.1.0.0/16 and use ER circuit2 to access 10.2.0.0/16, they can put 10.1.0.0/16 higher BGP Local Preference attribute in ER circuit and 10.2.0.0/16 higher Local Preference attribute in ER circuit2. </br>

```router bgp 65000
 bgp log-neighbor-changes
 neighbor 24.1.1.2 remote-as 12076
 neighbor 24.1.1.2 route-map ModifyLP in
!
ip prefix-list LP seq 5 permit 10.1.0.0/16
!
route-map ModifyLP permit 10
 match ip address prefix-list LP
 set local-preference 400
!
route-map ModifyLP permit 20
!
```
