---
title: Logic App (Standard) and API Connections
layout: post
tags:
  - Logic App
  - API Connection
  - ARM Template
---

I have recently had the pleasure ([You keep using that word. I do not think it means what you think it means.][pb]) of deploying Logic App workflows on a Logic App (Standard) instance.
For those not familiar with Logic App (Standard), they are the single-tenant instance of Logic Apps.
They provide the ability to host your workflows within a virtual network, something that cannot be done with a consumption Logic App.
Under the hood, standard Logic Apps are a completely different beast to consumption Logic Apps.
Consumption apps can only have a single Workflow in the app (which makes sense when you consider you also pay by the execution), while standard apps are deployed into an App Service plan and can therefore have multiple workflows in a single Logic App.

Logic Apps are designed to be the glue between multiple systems.
The intent is that the workflow author can connect to multiple systems, and process data in a low-code manner by integrating many apps and services.
These connections are created and configured through the use of _[managed connectors][mc]_.
And this is where our fun begins…

According to Microsoft, "before you can use most managed connectors, you must first create a connection from your workflow and authenticate your identity."[^create-connection]
What Microsoft doesn't tell you is how to create the connections.
Not via the portal.
Not via code.
There is not a shred of documentation on how to create these connections.

OK, so that last statement is a little harsh.
There is [documentation][webconn-docs] on creating `Microsoft.Web/Connections` resources via ARM templates, but that is just generic information and does not cover how to create specific connection.
(I can understand why — the resource is a generic resource and it would not be possible to document all the possible connectors as part of the resource documentation.)
There are some [Logic App deployment samples][la-samples] on GitHub, but what the repository doesn't tell you is that they are only applicable to consumption Logic Apps.
Which leads me to another complaint: Microsoft uses the same Logic App name for two systems that are fundamentally different platforms, and Microsoft Support have admitted that they are diverging with every feature release.

Creating connections via the Azure Portal is simple.
You create a Logic App workflow, add in a data source (either as an action or a trigger) and the portal will offer to create the API Connection on your behalf.
And you will be presented with all the options for configuring the connection.
![Creating an API Connection through the Azure Portal](/solideogloria-tech.github.io/images/api-connections/portal-create.png)

But via code, there is nothing.
For most resources, if you manually deploy it you can export the ARM template required to redeploy them from the portal.
But when you export an ARM template containing API connections, the core information you need to redeploy is not there.
No authentication options.
No endpoint names.
And no hint of what the property names should be.

So how does one deploy a Logic App via code?
At this point it seems the simplest way is to contact Microsoft support and ask them to provide the information required.
But we don't always have 2–3 days to wait for an ARM template.
So to help, I've [created a repository][api-repo] to host ARM templates to deploy these API connections.

But what if you need a connection that is not part of the repository?
The best answer I have is to use the developer tools in your browser to capture the creation request from the portal, and to export that template.
To achieve this follow these steps:

1. Either create a new workflow, or open an existing workflow.
2. Add a new trigger or action (or you can modify an existing one).
3. Click on `Add connection` or `Change connection` (which you see will depend on whether you already have a connection to the selected service).
4. Fill in the connection details as requested.
5. **Before creating the connection, make sure you have the developer tools open (typically by pressing `F12`).**
6. Click the `Create` button.
7. In the `Network` tab of developer tools, look for a `PUT` request to an endpoint named for the type of resource you are deploying.
   ![Network request to create a new API connection](/solideogloria-tech.github.io/images/api-connections/network-request.png)
8. Open the `Payload` tab in the request and copy the ARM JSON into an editor and update as required to template the connection.

But creating the API connections is only half of the battle, because then you need to tell the Logic App how to use it.
This is done through a `connections.json` file that is deployed along with your Logic App workflow code.
This needs to be created and updated with information that is specific to the API Connection resource that is deployed.
But that is the subject for another post.

[api-repo]: https://github.com/oWretch/logicapp-connections
[la-samples]: https://github.com/Azure-Samples/azure-logic-apps-deployment-samples
[mc]: https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview#managed-connector
[pb]: https://www.youtube.com/watch?v=dTRKCXC0JFg
[webconn-docs]: https://docs.microsoft.com/en-us/azure/templates/microsoft.web/connections?tabs=json

[^create-connection]: Taken from the [Logic Apps documentation on Managed Connectors][mc]
