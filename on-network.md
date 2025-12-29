# On network vs off network traffic flow

## Introduction

A common hurdle for a network engineer to grasp in the difference between public and private networking. Many cloud native services do not live "on the network" or "on a VNet". These services were primarily designed to be accessed over public endpoints over the public internet. Advanced features such as Private Endpoints and Virtual Network Injection have helped blend the different traffic flows, you must understand the nuances to ensure that the traffic gets where you want it to get and secured in a manner that you intend.

> [!NOTE]
> Public internet traffic may be across the Microsoft network backbone and/or may never leave an Azure datacenter, but it is still utilizing public IP addresses and it routable from the internet. The path that traffic will take is highly dependent upon source, destination, and routing rules between.

## On network (i.e., Private networking)

Simply put virtual machines and VM based PaaS services are hosted on a virtual network (VNet). These services are "on network". In reality, cloud based "on network" hosting is not all that different from an on-premises or co-lo based datacenter. The control plane is different, but the networking is fundamentally the same.

Azure VNets should follow RFC1918 rules just like an on-premises datacenter. This allows traffic to flow easily between the two. In fact, if you connect your on-premises datacenter to an Azure VNet via S2S VPN or Express Route, you can consider the combined network as "on network".

Additionally, in some architectures you could have disconnected, independent networks. Since these networks do not connect each would need their own ingress, egress, routing rules, DNS, and other supporting services.

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

Many cloud services are multi-tenant PaaS and SaaS offering. They do not operate on the customers network. They do not have RFC 1918 IP addresses that you can point to as a destination or utilize as a source (with the exception of VNet injection and Private Endpoints). They rely on *shared* Public IPv4 and IPv6 addresses. The public endpoint relies on L7 host headers to properly direct traffic to the desired service. If if you know the public IP address (which is not guaranteed to remain static) of the PaaS service, the service will not respond without accurate host headers in the request.

### Destination

Azure services vary somewhat in capabilities they support but most work very similarly. When sending traffic to a PaaS service there are generally network ACL controls on the public endpoint as well as private endpoint capabilities.

#### Public Endpoint

Using the FQDN traffic must be allowed from the source (through firewalls or other network technologies) to reach the public endpoint on the internet. The public endpoint can then have its own ACLs which can only allow specific *public* IP address, Azure Virtual Networks (service endpoints), or specific Azure resources.

#### Private Endpoint

After setting up private endpoint infrastructure, you *can* access the PaaS Service via a private IP address. However, you must still use the FQDN so that the service can read the host header. This is why DNS plays such a big role in private endpoints.

Standard private endpoints are one way traffic *to* the PaaS service. The service is not allowed to originate traffic into your network.

The public endpoint can be completely disabled and services which are on-network will be able to reach the PaaS service over the private endpoint.

> [!NOTE]
> Private Endpoints only work source systems that are on-network! Other off-network PaaS/SaaS services **cannot** access a private endpoint. If you need to connect a PaaS/SaaS service to another PaaS/SaaS service, this generally needs to be done over a public endpoint.


### Source

Often times PaaS/SaaS services are the source of a traffic flow (e.g., an App Service connecting to a SQL database). But remember that private endpoints cannot originate traffic and send it into your network. That network connection will originate from a shared non-persistent public IP address. Thus you cannot use the target service's network ACLs to control inbound traffic.

This lack of network control point can be difficult to overcome, but here are some techniques to think about.

#### Virtual Network Injection

Some PaaS Services can be "injected" into an Azure VNet. Instead of running on shared multi-tenant system, resources will be provisioned on an Azure VNet that you control. The PaaS service moves from off-network to on-network. The backend IP addresses which are the source of calls to the other service (say Azure SQL), will follow routing rules and resolve private endpoints.

#### Managed Private Endpoints

Some PaaS Services (primarily around data ETL) have a capability called Managed Private Endpoint. This setup is completely transparent to the customer, and does not require private endpoint infrastructure and DNS resolution. This capability allow a service (e.g., Azure Data Factory) to connect a private endpoint of another service (e.g., Azure Storage). This managed private endpoint is only usable by the single instance of the source service, and not services on your network.

#### Reverse Proxy

A Reverse Proxy can exist based upon any number of technologies, but the principal is the same--you have a publicly facing L7 service with a narrow security profile that proxies traffic to an on-network endpoint. Some examples include:

- Azure Application Gateway
- Microsoft On-premises Data Gateway (Note that this could be installed on an Azure VM. It is better though of as an "on-network data gateway)
- Azure Virtual Network Data Gateway (The spiritual successor to the On-premises Data Gateway, but must be hosted in an Azure VNet)
- 3rd Party (non-Microsoft)

In all cases, the end-to-end flow needs to be understood to ensure the proxy technology supports it.

#### Mitigating Controls

In some cases there might not be a good technical solution to allow PaaS/SaaS to PaaS/SaaS communication over private networking. You may need to allow public access from a large number of internet IP addresses. In these cases, you should look at what mitigating controls (preventative as well as detective) that you can implement to accept the risks of such a configuration. Mitigating controls might include:

- Strong Authentication
- Enhanced logging and alerting
- Data level protections such as scrubbing and/or DRM

## Interplay between networking flows