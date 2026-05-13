# On network vs off network traffic flow

## Introduction

A common hurdle for network engineers is grasping the difference between public and private networking in Azure. Many cloud-native services do not live "on the network" or "on a VNet"—they were originally designed to be accessed via public endpoints over the public internet. Features such as Private Endpoints and Virtual Network Injection have helped blend these traffic flows, but you still need to understand the nuances to ensure traffic reaches the right destination and is secured the way you intend.

> [!NOTE]
> Public internet traffic may be across the Microsoft network backbone and/or may never leave an Azure datacenter, but it is still utilizing public IP addresses and it routable from the internet. The path that traffic will take is highly dependent upon source, destination, and routing rules between.

## On network (i.e., Private networking)

Simply put, virtual machines and VM-based PaaS services are hosted on a virtual network (VNet). These services are "on network". In practice, cloud-based "on network" hosting is not all that different from an on-premises or co-located datacenter—the control plane is different, but the networking is fundamentally the same.

Azure VNets should follow RFC1918 rules just like an on-premises datacenter. This allows traffic to flow easily between the two. In fact, if you connect your on-premises datacenter to an Azure VNet via S2S VPN or Express Route, you can consider the combined network as "on network".

Additionally, in some architectures you may have disconnected, independent networks. Since these networks do not connect, each would need its own ingress, egress, routing rules, DNS, and other supporting services.

### Egress

Egress from an Azure Virtual network can take a few different routes. In an enterprise environment, the traffic will usually flow through an outbound firewall (with a public IP address) to reach the Internet. There are some exceptions to this such as using Service Endpoints, but the base case is to send traffic through the firewall.

> [!NOTE]
> Service endpoints route traffic directly to the *public* interface of an Azure service, ignoring other routing rules and bypassing any outbound firewalls.

You can choose your network topology and routes that you would like traffic to take to an outbound firewall. If you have a robust outbound security stack in your on-premises datacenter; you may want to "force tunnel" all internet bound traffic from Azure down the S2S VPN/Express Route and then have it hair pin out to the Internet. This is generally discouraged because it is inefficient and costly, but could be a first step in a cloud build-out.

A more efficient way is to build that same security stack in Azure and let Azure traffic exit to the internet directly from Microsoft datacenters. On-premises traffic may still egress from an existing on-premises firewall. Depending upon long term WAN, data center and zero-trust client strategies, the On-premises egress could eventually be retired in favor of what is hosted in Azure.

### Ingress

Similar to on-premises, ingress into an Azure VNet requires a public IP address attached to a public facing device. This is usually a firewall of some sort. It could be a traditional L4 firewall or it could be a L7 web application firewall (e.g., Azure Application Gateway) which acts as a reverse proxy.

Those ingress points can (depending upon routing and network ACLs) direct inbound traffic to anywhere else on your private network. So you could place an Azure Application Gateway in front of an on-premises web application or API.

It is not uncommon to have multiple ingress points. You may have only one L4 firewall ingress, but you may also have multiple Azure Application Gateways (i.e., one for each public facing application).

Again, this ingress would be to expose an on-network application to the public internet.

## Off network (i.e., Public networking)

Many cloud services are multi-tenant PaaS and SaaS offerings. They do not operate on the customer's network and do not have RFC 1918 IP addresses that you can point to as a destination or use as a source (with the exception of VNet injection and Private Endpoints). Instead, they rely on *shared* public IPv4 and IPv6 addresses, and use L7 host headers to direct traffic to the correct tenant. Even if you know the public IP address of the PaaS service (which is not guaranteed to be static), the service will not respond without the correct host header in the request.

### Service as the Destination

Azure services vary somewhat in capabilities they support but most work very similarly. When sending traffic to a PaaS service there are generally network ACL controls on the public endpoint as well as private endpoint capabilities.

#### Public Endpoint

Traffic from the source (through firewalls or other network technologies) must be allowed to reach the public endpoint on the internet using its FQDN. The public endpoint can then enforce its own ACLs, which can restrict access to specific *public* IP addresses, Azure Virtual Networks (via service endpoints), or specific Azure resources.

#### Private Endpoint

After setting up private endpoint infrastructure, you *can* access the PaaS Service via a private IP address. However, you must still use the FQDN so that the service can read the host header. This is why DNS plays such a big role in private endpoints.

Standard private endpoints are one way traffic *to* the PaaS service. The service is not allowed to originate traffic into your network.

The public endpoint can be completely disabled and services which are on-network will be able to reach the PaaS service over the private endpoint.

> [!NOTE]
> Private Endpoints only work for source systems that are on-network! Other off-network PaaS/SaaS services **cannot** access a private endpoint. If you need to connect one PaaS/SaaS service to another, that traffic generally needs to flow over a public endpoint.


### Service as the Source

Often, a PaaS/SaaS service is the *source* of a traffic flow (e.g., an App Service connecting to a SQL database). That outbound connection originates from a shared, non-persistent public IP address—even when the source service has a private endpoint attached to it. Remember: private endpoints are inbound only; they cannot originate traffic into your network.

This makes it difficult to use the target service's network ACLs to control inbound traffic.

This lack of network control point can be difficult to overcome, but here are some techniques to think about.

#### Virtual Network Injection

Some PaaS services can be "injected" into an Azure VNet. Instead of running on a shared multi-tenant system, the resources are provisioned into an Azure VNet that you control. The PaaS service effectively moves from off-network to on-network. The backend IP addresses that are the source of calls to other services (say, Azure SQL) will then follow your routing rules and resolve private endpoints normally.

#### Managed Private Endpoints

Some PaaS services (primarily data ETL services) offer a capability called Managed Private Endpoint. The setup is fully transparent to the customer and does not require you to build private endpoint infrastructure or DNS resolution. It allows a service (e.g., Azure Data Factory) to connect to the private endpoint of another service (e.g., Azure Storage). The managed private endpoint is only usable by the single source service instance that owns it—other systems on your network would still require a traditional private endpoint.

#### Reverse Proxy

A reverse proxy can be built on any number of technologies, but the principle is the same: a publicly facing L7 service with a narrow security profile proxies traffic to an on-network endpoint. Some examples include:

- Azure Application Gateway
- Microsoft On-premises Data Gateway (this can be installed on an Azure VM, in which case it's better thought of as an "on-network data gateway")
- Azure Virtual Network Data Gateway (the spiritual successor to the On-premises Data Gateway, but must be hosted in an Azure VNet)
- 3rd Party (non-Microsoft)

In all cases, the end-to-end flow needs to be understood to ensure the proxy technology supports it.

#### Mitigating Controls

In some cases, there is no good technical solution for PaaS/SaaS to PaaS/SaaS communication over private networking, and you may need to allow public access from a large set of internet IP addresses. When that happens, identify the mitigating controls (both preventative and detective) you can put in place to accept the risk. Mitigating controls might include:

- Strong Authentication
- Enhanced logging and alerting
- Data level protections such as scrubbing and/or DRM

## Interplay between networking flows

## Network Security Perimeters