# Private Endpoints

Private Endpoints can be somewhat difficult to understand until one day it just clicks. Part of the challenge is that DNS is 90% of the solution (and thus 90% of the related problems), but not all traditional DNS thinking applies.

> [!TIP]
> For comprehensive information about Private Endpoints, see the [Azure Private Endpoint DNS configuration guide](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns).

## What is a Private Endpoint?

A Private Endpoint is a network interface (NIC) in *your* VNet that gets assigned a private IP address from your address space and is wired through Azure Private Link to a specific instance of a PaaS service (a specific storage account, a specific Key Vault, a specific SQL server, etc.). Once it exists, traffic to that PaaS instance can flow over your private network instead of the public internet.

A few things to keep in mind from the start:

- A Private Endpoint targets one **instance** of a service, not the service as a whole. You don't get a private endpoint to "Azure Storage"—you get one to *your* storage account.
- Many services have multiple **sub-resources** (Azure Storage exposes blob, file, queue, table, and dfs; Cosmos DB exposes Sql, MongoDB, etc.). Each sub-resource you want to reach privately needs its own Private Endpoint.
- A Private Endpoint is one-way (inbound to the PaaS service). The PaaS service cannot use it to originate traffic into your network.
- The private IP is reachable like any other VNet IP—across peering, ExpressRoute, and VPN—as long as routing and firewall rules allow it.

## Organization

Each Private Endpoint service type has a corresponding "privatelink" private DNS zone. For each service that you use, you should create one (and generally only one) private DNS zone for your network. If you have multiple networks (likely independent routing and certainly independent DNS resolution), you may need a second set of these zones.

> [!TIP]
> For a complete list of private DNS zone values for different Azure services, see [Azure Private Endpoint private DNS zone values](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns).

These zones should live in a Shared Services location and be managed by a Cloud Ops team. It makes sense to group them into a single resource group.

![List of private zones](/images/list-of-private-zones.png)

When creating Private Endpoints through the Azure Portal, it makes it very easy to create additional private DNS zones in non-centralized locations. AVOID THIS. Re-use already created zones whenever they exist or create new ones in a central location.

## Implementation

Your first private endpoint is the most work because you need to stand up and configure the DNS infrastructure to support private endpoints. They get significantly easier after that. Here are the high-level implementation steps:

|Task|Frequency|
|----|---------|
|Build DNS Infrastructure|Once|
|Create private DNS zone and conditional forwarding|Once per private endpoint service type|
|Connect Private Endpoint to DNS zone|Once per private endpoint|

## DNS Infrastructure

The secret to private endpoint DNS resolution is that it requires *magic*. Specifically, the DNS resolution needs to take place in an Azure VNet which is using the "Azure provided DNS service".

![DNS service](/images/dns-service.png)

This service uses a *magic IP* of 168.63.129.16. But you can't simply forward your DNS requests to this IP address and expect it to work. DNS responses **will vary** based on which Azure VNet the request is coming from!

> [!TIP]
> To learn more about this special IP address, see [What is IP address 168.63.129.16?](https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16)

Think about it: every Azure customer in the world is using this IP address, but you only want resolution for your endpoints, not everybody else's. This requires that you **link** your private DNS zones to your VNet(s). Depending on how you are configuring DNS outside of Private Endpoints, you may only need to link private DNS zones to a single VNet where your DNS servers live.

> [!TIP]
> For detailed information about virtual network links, see [What is a virtual network link?](https://learn.microsoft.com/en-us/azure/dns/private-dns-virtual-network-links)

![VNet linking](/images/vnet-link.png)

> [!NOTE]
> Auto-Registration applies only to virtual machines and will create A RECORDS for virtual machines in the associated VNet. You should never need to use this option for Private Endpoints. This would only be used with a custom private DNS zone specific to your company. If you are using a different DNS solution (like Active Directory), you likely don't need this option ever.

> [!NOTE]
> Fallback to Internet is a feature that solves a unique situation where a third party has a private endpoint enabled, but wants you to access its public endpoint. This feature tells **your** DNS resolution to check for the public endpoint if it is not found in **your** private endpoint zones.

You may put your own DNS solution into the DNS resolution VNet. This could be BIND, Active Directory DNS, or anything really. The key point is that the resolution needs to take place IN that VNet.

Consider if you are using Active Directory as your primary DNS service. You have domain controllers on-premises as well as in Azure. DNS configuration is synced across all domain controllers. Within Active Directory DNS, you configure specific upstream DNS servers that meet your security requirements (e.g., Cloudflare or OpenDNS). You don't want to have your domain controllers use the Azure provided DNS service, thus they cannot do private endpoint resolution. Additionally, if a DNS resolution took place on your on-premises domain controller, private endpoint resolution wouldn't work anyway.

Enter Azure DNS Private Resolver. This is a simple PaaS DNS service that you can conditionally forward private endpoint DNS resolution to. You would create a dedicated VNet for the Private Resolver and configure it to use the Azure provided DNS service. All other VNets in your environment would continue to point at your domain controllers. You then conditionally forward any private endpoint FQDNs you plan to use from Active Directory DNS to the Inbound Endpoint of the Private Resolver. This forces resolution of private endpoint zones to take place in the correctly configured VNet.

Two important reasons drive this design:

1. **Why conditionally forward from Active Directory at all?** You want your default DNS resolution path to keep using your preferred external resolver (Cloudflare, OpenDNS, etc.) for everything that *isn't* a private endpoint. Conditional forwarding lets you carve out just the private endpoint FQDNs and hand them off to Azure DNS for resolution, while leaving every other lookup alone.

2. **Why forward to a Private Resolver inbound endpoint instead of directly to the Azure DNS magic IP (168.63.129.16)?** Because Active Directory DNS is replicated between your on-premises and Azure domain controllers, the same conditional forwarder will end up running on *both* sides. If the forwarder pointed straight at 168.63.129.16, an on-premises domain controller (which is not in an Azure VNet) would send the query to the magic IP from outside Azure, where it cannot be answered correctly. The result: private endpoints would likely resolve fine for clients in Azure, but fail for clients on-premises. Forwarding to the Private Resolver's inbound endpoint (a routable private IP reachable from both Azure and on-premises) ensures the actual resolution always happens inside the correctly configured Azure VNet, no matter where the original request originated.

> [!TIP]
> To get started with Azure DNS Private Resolver, see [What is Azure DNS Private Resolver?](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview) and [Quickstart: Create an Azure DNS Private Resolver using the Azure portal](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-get-started-portal). 

![Private Endpoint Networking](/images/PrivateEndpointNetworking.png)

>[!NOTE]
>The red lines represent network routing. The other colored lines represent configurations, not network paths.


To sum up, you need:

- A DNS service in a VNet that is configured to use the Azure provided DNS service (Azure DNS Private Resolver is recommended)
- Private DNS zones linked to the above VNet
- Private endpoints configured to register with the appropriate private DNS zone.
- You'll likely need to conditionally forward private endpoint zones from your primary DNS solution to the above mentioned DNS service.

> [!TIP]
> For guidance on configuring conditional forwarders in Active Directory DNS, see [Administer DNS and create conditional forwarders in a Microsoft Entra Domain Services managed domain](https://learn.microsoft.com/en-us/entra/identity/domain-services/manage-dns#create-conditional-forwarders) and [Add-DnsServerConditionalForwarderZone](https://learn.microsoft.com/en-us/powershell/module/dnsserver/add-dnsserverconditionalforwarderzone).

## Public DNS Records

Each Private Link-capable service has a designated public DNS zone, and we essentially need to override that resolution. It's usually best to stick with the default DNS zones rather than custom zones (e.g., storageaccount123.company.com), because most services don't support custom FQDNs.

See the public DNS resolution of a blob storage account before a private endpoint is created:

    hbteststorageaccount.blob.core.windows.net
    Server:  XXXX
    Address:  XXX.XXX.XXX.XXX

    Non-authoritative answer:
    Name:    blob.lon26prdstr09c.store.core.windows.net
    Address:  20.209.30.1
    Aliases:  hbteststorageaccount.blob.core.windows.net

And after the private endpoint is created:

    hbteststorageaccount.blob.core.windows.net
    Server:  XXXX
    Address:  XXX.XXX.XXX.XXX

    Non-authoritative answer:
    Name:    hbteststorageaccount.privatelink.blob.core.windows.net
    Address:  10.100.1.25
    Aliases:  hbteststorageaccount.blob.core.windows.net

Two things to note:
- A private IP address is returned (assuming all DNS infrastructure is set up correctly)
- The canonical name has changed to a .privatelink. FQDN

> [!NOTE]
> It's a common misconception that you should begin using the .privatelink. FQDN after you create the private endpoint. In reality, if DNS is properly configured, you should continue to use the original FQDN. Even when implementing conditional forwarding to the private resolver—forward the original FQDN, and **not** the .privatelink. domain.

> [!TIP]
> For more detailed DNS configuration guidance, see [Configure DNS forwarding for Azure Files using VMs or Azure DNS Private Resolver](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-networking-dns), [Resolve Azure and on-premises domains](https://learn.microsoft.com/en-us/azure/dns/private-resolver-hybrid-dns) and [Azure Private Endpoint DNS integration scenarios](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration).

## Network Security and Routing

A Private Endpoint is just a NIC with a private IP, but its traffic behaves a bit differently than a normal VM NIC—and that's where people get tripped up.

### NSGs and UDRs

Historically, traffic to a Private Endpoint **bypassed Network Security Groups (NSGs)** and User-Defined Routes (UDRs) on the subnet hosting the endpoint. That behavior has changed: Azure now supports a per-subnet setting (`privateEndpointNetworkPolicies`) that lets NSGs and route tables apply to Private Endpoint traffic. If you're using a hub-and-spoke topology and want to enforce NSGs (or steer Private Endpoint traffic through a firewall via UDR), enable that setting on the subnet hosting your endpoints.

> [!TIP]
> See [Manage network policies for Private Endpoints](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy) for the current behavior and how to enable NSGs/UDRs.

### Reachability across peering, ExpressRoute, and VPN

A Private Endpoint's IP is reachable the same way any other VNet IP is reachable. That means:

- Peered VNets (including spokes in a hub-and-spoke topology) can reach a Private Endpoint as long as peering allows it and routing/firewall rules permit the flow.
- On-premises clients connected via ExpressRoute or Site-to-Site VPN can also reach a Private Endpoint, provided routes are advertised and firewalls let the traffic through.

The catch is almost always DNS: even with full network reachability, on-premises clients won't resolve the private IP unless you've extended your private endpoint DNS strategy to them (typically via the Private Resolver pattern described above).

## Cost Considerations

Private Endpoints are not free, and the costs are easy to overlook when you're focused on getting traffic flowing:

- Each Private Endpoint has an hourly charge plus a per-GB inbound and outbound data processing charge.
- Azure DNS Private Resolver bills hourly per inbound/outbound endpoint and per million DNS queries.
- Private DNS zones themselves are inexpensive, but a sprawl of duplicated zones (the anti-pattern called out in the [Organization](#organization) section) inflates management overhead more than cost.

> [!TIP]
> See [Azure Private Link pricing](https://azure.microsoft.com/pricing/details/private-link/) and [Azure DNS pricing](https://azure.microsoft.com/pricing/details/dns/) for current rates.

## Operational Gotchas

A few things that bite teams in production:

- **Stale DNS records on deletion.** If you delete a Private Endpoint that was created with the "Integrate with private DNS zone" option, the A record in the private DNS zone may not be cleaned up automatically (especially if the zone was managed inconsistently). Audit your zones periodically and remove orphaned records.
- **Re-creating a Private Endpoint to the same service can cause IP changes.** If clients have cached the old IP (or you've hard-coded it somewhere), expect breakage.
- **One Private Endpoint per sub-resource.** Don't assume a single endpoint to a storage account covers everything—blob and file each need their own.
- **Region matters.** A Private Endpoint and the PaaS instance it targets do not have to be in the same region, but cross-region traffic incurs bandwidth charges and adds latency.

## DNS Resolution Walkthrough

If everything is wired up correctly, here's how a single resolution flows from a client all the way to a Private Endpoint IP. Imagine a client in an Azure VNet (or on-premises) trying to reach `hbteststorageaccount.blob.core.windows.net`:

1. **Client → Domain Controller.** The client asks its configured DNS server—an Active Directory domain controller—to resolve the storage account FQDN.
2. **Domain Controller → Private Resolver.** The domain controller has a conditional forwarder for `privatelink.blob.core.windows.net` (and other private endpoint zones) pointing at the Azure DNS Private Resolver's inbound endpoint. It forwards the query there.
3. **Private Resolver → Azure-provided DNS (168.63.129.16).** Because the Private Resolver lives in an Azure VNet configured to use the Azure-provided DNS service, its query to the magic IP is answered in the context of that VNet, with full visibility into any private DNS zones linked to it.
4. **Azure-provided DNS lookup.** Azure DNS sees that the public record for the storage account is a CNAME to `hbteststorageaccount.privatelink.blob.core.windows.net`. It then checks the linked private DNS zone, finds the matching A record (created when the Private Endpoint was provisioned), and returns the private IP—e.g., `10.100.1.25`.
5. **Response trip back.** The Private Resolver returns the answer to the domain controller, which returns it to the client.
6. **Client connects.** The client opens its connection to `10.100.1.25` over the private network. From here, normal routing, NSGs, and firewall rules apply—DNS got you the IP, but it's still up to your network design to actually let the packets through.

A couple of subtleties worth highlighting:

- If the private DNS zone is missing the A record (or isn't linked to the resolver's VNet), Azure DNS returns the *public* IP via the CNAME chain. Your traffic will then try to leave through the public path—often blocked by firewalls, often confusing to diagnose.
- The "Fallback to Internet" feature on a private DNS zone is what allows that public-IP fallback when a record is missing. It's useful in a few specific cases (third-party Private Link services), but for your own endpoints it usually masks misconfiguration rather than fixing it.