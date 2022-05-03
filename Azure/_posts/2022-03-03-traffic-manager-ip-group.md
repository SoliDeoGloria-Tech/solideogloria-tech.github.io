---
title: Allow Traffic Manager Probes Through Azure Firewall
layout: post
tags:
  - Azure Firewall
  - Logic App
  - Traffic Manager
---

[Traffic Manager][tm] is an essential component of any resilient deployment within Azure.
Whether you have a multi-region behemoth, or simply want a simple way to activate DR instances should the primary go down, Traffic Manager has a configuration for you.
One key component of Traffic Manager is its probes—by frequently checking the status of your application, Traffic Manager can make intelligent decisions about where to direct the traffic.

As with all services, there are a specific set of IP addresses from which the probes will originate.
Microsoft even helpfully provide a [Service Tag][svc_tag] `AzureTrafficManager` which is kept up-to-date with the latest IP addresses used by Traffic Manager probes.
They even tell us that this Service Tag is supported for use in Azure Firewall.[^st_fw]
Except… that is not the whole story.

See, the typical usage for Traffic Manager probes with Azure Firewall occurs when you run a non-web service (because your web services are hosted behind [Application Gateway][appgw], aren't they?) and need to restrict access to the service.
This seems like the perfect use for the Service Tag—just add it to the DNAT rule and profit!!
But when you try to, this is the lovely message that greets you.

![Azure Portal error when using the Azure Traffic Manager Service Tag in a DNAT rule](/solideogloria-tech.github.io/images/tm-logic-app/service-tag-error.png)

What's the point in Azure Firewall supporting a Service Tag such as `AzureTrafficManager` but not in a DNAT rule?

All is not lost however.
Azure Firewall also supports using [IP Groups][ip-group] in rules.
Time to whip out my favourite[^fav_tech] Azure technology: Logic Apps.

![Traffic Manager IP Update Logic App Designer view](/solideogloria-tech.github.io/images/tm-logic-app/logic-app-designer.png)

The Logic App downloads the list of Traffic Manager IP addresses from Microsoft, and adds them to an IP Group.
This IP group can then be used in the source field of the DNAT rule in Azure Firewall.
The source code, including deployment templates is [available on GitHub][repo] for use.

_One thing to note is that the first run of the Logic App which is triggered during deployment will fail as the Logic App won't have permissions to the IP group._

This solution is less than ideal.
Because of the restriction of using Service Tags in DNAT rules we are forced to use two additional services which have a (minimal) cost associated with them.
Hopefully Microsoft will address this in a future update.

[appgw]: https://docs.microsoft.com/en-us/azure/application-gateway/overview
[la_connections]: /azure/logic-apps-and-api-connections.html
[repo]: https://github.com/oWretch/traffic-manager-ip-group-logic-app
[st_fw_docs]: https://docs.microsoft.com/en-us/azure/firewall/service-tags
[svc_tag]: https://docs.microsoft.com/en-us/azure/virtual-network/service-tags-overview#available-service-tags
[tm]: https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview

[^fav_tech]: Yes, I have a love–hate relationship with Logic Apps. They have so much power, and are excellent for tasks such as this, but the [connection nightmare][la_connections], plus lack of documentation makes them hard to deal with in enterprise environments.
[^st_fw]: Ok, as was pointed out to me, [the documentation][st_fw_docs] does specify that Service Tags are only supported in the destination fields of network rules. But still…
