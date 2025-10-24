# Private Endpoints

Private Endpoints can be somewhat difficult to understand until one day it just clicks. Part of the challenge is that DNS is 90% of the solution (and thus 90% of the related problems), but not all traditional DNS thinking applies.

## Organization

Each Private Endpoint service type has a corresponding "privatelink" Private DNS Zone. For each service that you use, you should create one (and generally only one) Private DNS Zone for your network. If you have multiple networks (likely independent routing and certainly independent DNS resolution), you may need a second set of these Zones.

These zones should live in a Shared Services location and be managed by a Cloud Ops team. It makes sense to group them into a single resource group.

![List of private zones](/images/list-of-private-zones.png)

When creating Private Endpoints through the Azure Portal, it makes it very easy to create additional private DNS zones in non-centralized locations. AVOID THIS. Re-use already created zones whenever they exist or create new ones in a central location.

## Implementation

Your first private endpoint is the most work because you need to stand up and configure the DNS infrastructure to support private endpoints. They get significantly easier after that. Here are the high-level implementation steps:

|Task|Frequency|
|----|---------|
|Build DNS Infrastructure|Once|
|Create Private DNS zone and conditional forwarding|Once per private endpoint service type|
|Connect Private Endpoint to DNS zone|Once per private endpoint|

## DNS Infrastructure

The secret to private endpoint DNS resolution is that it requires *magic*. Specifically, the DNS resolution needs to take place in an Azure VNet which is using the "Azure provided DNS service".

![DNS service](/images/dns-service.png)

This service uses a *magic IP* of 168.63.129.16. But you can't simply forward your DNS requests to this IP address and expect it to work. DNS responses **will vary** based upon which Azure VNet the request is coming from!

Think about it: every Azure customer in the world is using this IP address, but you only want resolution for your endpoints, not everybody else's. This requires that you **link** your Private DNS Zones to your VNet(s). Depending on how you are configuring DNS outside of Private Endpoints, you may only need to link private DNS zones to a single VNet where your DNS servers live.

![VNet linking](/images/vnet-link.png)

> [!NOTE]
> Auto-Registration applies only to virtual machines and will create A records for virtual machines in the associated VNet. You should never need to use this option for Private Endpoints. This would only be used with a custom private DNS zone specific to your company. If you are using a different DNS solution (like Active Directory), you likely don't need this option ever.

> [!NOTE]
> Fallback to Internet is a feature that solves a unique situation where a third party has a private endpoint enabled, but wants you to access its public endpoint. This feature tells **your** DNS resolution to check for the public endpoint if it is not found in **your** private endpoint zones.

You may put your own DNS solution into the DNS resolution VNet. This could be BIND, Active Directory DNS, or anything really. The key point is that the resolution needs to take place IN that VNet.

Consider if you are using Active Directory as your primary DNS service. You have domain controllers on-premises as well as in Azure. DNS configuration is synced across all domain controllers. Within Active Directory DNS, you configure specific upstream DNS servers that meet your security requirements (e.g., Cloudflare or OpenDNS). You don't want to have your domain controllers use the Azure provided DNS service, thus they cannot do private endpoint resolution. Additionally, if a DNS resolution took place on your on-premises domain controller, private endpoint resolution wouldn't work anyway.

Enter Azure DNS Private Resolver. This is a simplistic PaaS DNS service that you can conditionally forward private endpoint DNS resolution to. You would create a specific VNet for the Private Resolver and configure it to use the Azure provided DNS service. All other VNets in your environment would likely be configured to point to your domain controllers. You should conditionally forward any private endpoint FQDNs that you plan to use to the Inbound Endpoint of the Private Resolver. This forces all resolution of private endpoint related zones to take place in the correctly configured VNet. 

![Private Endpoint Networking](/images/PrivateEndpointNetworking.png)

>[!NOTE]
>The red lines represent network routing. The other colored lines represent configurations, not network paths.


To sum up, you need:

- A DNS service in a VNet that is configured to use the Azure provided DNS service (Azure DNS Private Resolver is recommended)
- Private DNS zones linked to the above VNet
- Private endpoints configured to register with the appropriate private DNS zone.
- You'll likely need to conditionally forward private endpoint zones from your primary DNS solution to the above mentioned DNS service.

## Public DNS Records

Private Link-able services (is that even a word?) have designated public DNS zones. We need to essentially override that resolution. It's usually best to use the defined DNS zones and avoid custom zones (i.e., storageaccount123.company.com), because most services don't support custom FQDNs.

See the public DNS resolution of a blob storage account before a private endpoint is created:

    hbteststroageaccount.blob.core.windows.net
    Server:  XXXX
    Address:  XXX.XXX.XXX.XXX

    Non-authoritative answer:
    Name:    blob.lon26prdstr09c.store.core.windows.net
    Address:  20.209.30.1
    Aliases:  hbteststroageaccount.blob.core.windows.net

And after the private endpoint is created:

    hbteststroageaccount.blob.core.windows.net
    Server:  XXXX
    Address:  XXX.XXX.XXX.XXX

    Non-authoritative answer:
    Name:    hbteststroageaccount.privatelink.blob.core.windows.net
    Address:  192.168.132.197
    Aliases:  hbteststroageaccount.blob.core.windows.net

Two things to note:
- A private IP address is returned (assuming all DNS infrastructure is set up correctly)
- The canonical name has changed to a .privatelink. FQDN

> [!NOTE]
> It's a common misconception that you should begin using the .privatelink. FQDN after you create the private endpoint. In reality, if DNS is properly configured, you should continue to use the original FQDN. Even when implementing conditional forwarding to the private resolver—forward the original FQDN, and **not** the .privatelink. domain.

## DNS Resolution - The Play

If all is configured correctly, the DNS resolution flow will go something like this:

>**Client:** Hey Domain Controller, you are my DNS server. I'm looking for *hbteststroageaccount.blob.core.windows.net*

>**Domain Controller:** Oh hey Client, I actually can't answer that, I need to go ask the Azure Private Resolver.

>**Domain Controller:** Yo Private Resolver, Client is looking for a private endpoint. I need you to do that magic thing that you do.

>**Private Resolver:** Sure thing, one moment.

>**Private Resolver:** Mr. Azure Provided DNS, sir. Its me, Private Resolver from Company ABC's VNet. I'm looking for a private endpoint, and here are my linked Private DNS Zones...its probably in there somewhere.

>**Azure Provided DNS:** Well lets take a look, shall we. I see that the public DNS record has been updated so that the canonical name is a .privatelink. DNS name. And if I just look through your linked private DNS Zones...ah yes...here it is. You have an A RECORD for it right here. You can respond with 10.100.1.25.

>**Private Resolver:** Thanks Azure Provided DNS, you are the best...sir.

>**Azure Provided DNS:** Remember, if I don't find it in your linked private DNS zones, I can't give you the public endpoint's address unless you enabled Fallback to Internet!

>**Private Resolver:** Ok, Domain Controller, I got that address for you. 10.100.1.25.

>**Domain Controller:** Alright Client, the private address is 10.100.1.25.

>**Client:** Thanks Domain Controller, now lets just hope that Firewall lets me through!