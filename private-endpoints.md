# Private Endpoints

Private Endpoints can be somewhat difficult to understand until one day it just clicks. Part of the challenge is that DNS is 90% of the solution (and thus 90% of the related problems), but not all traditional DNS thinking applies.

## Organization

Each Private Endpoint service type has a corresponding "privatelink" Private DNS Zone. For each service that you use, you should create one (and generally only one) Private DNS Zone for your network. If you have multiple networks (likely independent routing and certainly independent DNS resolution), you may need a second set of these Zones.

These zones should live in a Shared Services location and be managed by a Cloud Ops team. It makes sense to group them into single resource group.

![List of private zones](/images/list-of-private-zones.png)

When creating Private Endpoints through the Azure Portal it makes it very easy to create additional private DNS zones in non-centralized locations. AVOID THIS. Re-use already created zones when ever they exist or create new ones in a central location.

## Implementation

Your first private endpoint is the most work because you need to stand up and configure the DNS infrastructure to support private endpoints. They get significantly easier after that. Here are the high level implementation steps

|Task|Frequency|
|----|---------|
|Build DNS Infrastructure|Once|
|Create Private DNS zone|Once per private endpoint service type|
|Connect Private Endpoint to DNS zone|Once per private endpoint|

## DNS Infrastructure

The secret to private endpoint DNS resolution is that it requires *magic*. Specifically, the DNS resolution needs to take place in an Azure VNet which is using the "Azure provided DNS service".

![DNS service](/images/dns-service.png)

This service uses a *magic IP* of 168.63.129.16. But you can't simply forward your DNS requests to this IP address and expect it to work. DNS responses **will vary** based upon which Azure VNet the request is coming from!

Think about it, every Azure customer in the world is using this IP address, but you only want resolution for your endpoints, not everybody else's. This requires that you **link** your Private DNS Zones to your VNet(s). Depending upon how you are configuring DNS outside of Private Endpoints, you may only need to link private DNS zones to a single VNet where you DNS servers live.

![VNet linking](/images/vnet-link.png)

> [!NOTE]
> Auto-Registration applies only to virtual machines and will create A RECORDS for virtual machines in the associated VNet. You should never need to use this option for Private Endpoints. This would only be used with a custom private DNS zone specific to your company. If you are using an different DNS solution (like Active Directory), you likely do not need this option ever.

> [!NOTE]
> Fallback to Internet is a feature that solves a unique situation where a third party has a private endpoint enabled, but wants you to access its public endpoint. This feature tells **your** DNS resolution to check for the public endpoint if it is not found in **your** private endpoint zones.

You may put your own DNS solution into the DNS resolution VNet. This could be BIND, Active Directory DNS, or anything really. The key point is that the resolution need to take place in that VNet. So if you have a distributed DNS architecture (like Active Directory), where the resolution could take place somewhere else (like an on premises server)--you need an independent DNS resolver. Enter Azure DNS Private Resolver. This is an simplistic PaaS DNS service that you can forward private endpoint DNS resolution to. This forces all resolution of private endpoint related zones to take place in the correctly configured VNet.

To sum up, you need:

- A DNS service in a VNet that is configured to use the Azure provided DNS service
- Private DNS zones linked to the above VNet
- Likely to conditionally forward private endpoint zones from your primary DNS solution to that DNS service.

## DNS Resolution

Private Linkable (is that a word) services have designated public DNS zones. We need to essentially override that resolution. It is usually best to use the defined DNS zones and avoid a custom zone (i.e., storageaccount123.company.com), because most services do not support custom FQDNs.

See the public DNS resolution of a blob store before a private endpoint is created:

    Default Server:  XXXX
    Address:  XXX.XXX.XXX.XXX

    hbteststroageaccount.blob.core.windows.net
    Server:  XXXX
    Address:  XXX.XXX.XXX.XXX

    Non-authoritative answer:
    Name:    blob.lon26prdstr09c.store.core.windows.net
    Address:  20.209.30.1
    Aliases:  hbteststroageaccount.blob.core.windows.net

And after the private endpoint is created: