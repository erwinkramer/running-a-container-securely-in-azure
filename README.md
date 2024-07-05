# Running a container securely in Azure

[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]
![GitHub commit activity](https://img.shields.io/github/commit-activity/m/erwinkramer/running-a-container-securely-in-azure)

Choosing the right tool for the job is hard, especially in the Azure container landscape. Microsoft already provides some great documentation, like [Choose an Azure container service](https://learn.microsoft.com/en-us/azure/architecture/guide/choose-azure-container-service) and [General architectural considerations for choosing an Azure container service](https://learn.microsoft.com/en-us/azure/architecture/guide/container-service-general-considerations). Interestingly, they make a comparison between Azure Kubernetes Service (AKS), Web App for Containers and Container Apps, while leaving out Azure Container Instances.

In this article, the focus is running a single container image, exposed on a private network with private ingress, on Azure, with as little overhead as possible.

This case rules out additional tools like Application Gateways or any other ingress controllers outside of the resource, as that would not make it a fair comparison. We can also rule out Azure Kubernetes Service (AKS), as it is simply a managed Kubernetes service, which is short for: you get a whole cluster of VMs.

We're adding Azure Container Instances to the comparison instead as it is suitable of running a single container image.

## Secure traffic considerations

What defines secure traffic? It simply is the ability to verify certificates, tokens and network origin of requests, all while having an identity and reliable endpoint on it's own. This boils down to the following comparison:

| *feature* | Web App for Containers | Container Instances | Azure Container Apps |
|:---|:---|:---|:---|
| Private endpoint | ✅ | ❌ | ❌ |
| Inbound Private IP behavior | ✅, static | ⚠️, [dynamic on restart or redepoy](https://github.com/MicrosoftDocs/azure-docs/issues/65128) | ⚠️, static -  [VNet-scope ingress requires domain name usage with complex DNS setup](https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#dns:~:text=If%20you%20plan%20to%20use%20VNet%2Dscope%20ingress%20in%20an%20internal%20environment%2C%20configure%20your%20domains%20in%20one%20of%20the%20following%20ways%3A) |
| FQDN | ✅, [azurewebsites.net](https://learn.microsoft.com/nl-nl/azure/app-service/overview-private-endpoint#dns), [static with subdomain takeover risk](https://learn.microsoft.com/en-us/azure/security/fundamentals/subdomain-takeover) or [dynamic on redepoy](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/public-preview-creating-web-app-with-a-unique-default-hostname/ba-p/4156353) | ✅, [azurecontainer.io](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-quickstart-portal#create-a-container-instance:~:text=page%2C%20specify%20a-,DNS%20name%20label,-for%20your%20container) - [static + protected](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-quickstart-portal#create-a-container-instance:~:text=DNS%20name%20label%20scope%20reuse) | ⚠️, [azurecontainerapps.io](https://learn.microsoft.com/en-us/azure/container-apps/connect-apps?tabs=bash#location) - [dynamic on Container Apps environment redeploy](https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#:~:text=UNIQUE_IDENTIFIER%3E.%3CREGION_NAME%3E) |
| HTTP Ports exposed | ⚠️, [443, 80](https://learn.microsoft.com/en-us/azure/app-service/networking-features#app-service-ports) | ✅, [up to 5 ports](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-resource-and-quota-limits#:~:text=20-,Ports%20per%20IP,-5) | ✅, [up to 6 ports](https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview#additional-tcp-ports:~:text=There%27s%20a%20maximum%20of%205%20additional%20ports%20per%20app) |
| HTTPS Certificate - Microsoft domain | ✅ | ⚠️, via [Caddy -  reverse proxy](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-group-automatic-ssl) or [other sidecar container](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-group-ssl) | ✅ |
| HTTPS Certificate - custom domain | ⚠️, [free managed certificate by DigiCert](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=apex#create-a-free-managed-certificate) - [no private DNS support](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=apex#create-a-free-managed-certificate:~:text=Doesn%27t%20support%20private%20DNS.) | ✅, free managed certificate by Let's Encrypt via [Caddy -  reverse proxy](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-group-automatic-ssl) or [other sidecar container](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-group-ssl) - [only sidecar has to be publicly accessible](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-group-automatic-ssl#:~:text=only%20the%20Caddy%20container%20gets%20exposed%20on%20ports%2080/TCP%20and%20443/TCP) | ⚠️, [free managed certificate](https://learn.microsoft.com/en-us/azure/container-apps/custom-domains-managed-certificates?pivots=azure-portal) - [ingress has to be publicly accessible](https://learn.microsoft.com/en-us/azure/container-apps/custom-domains-managed-certificates?pivots=azure-portal#:~:text=enabled%20and%20is-,publicly%20accessible,-.) |
| Ingress token validation | ✅, [Easy Auth](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization?WT.mc_id=dotnet-00000-cephilli#identity-providers)| ❌ | ✅, [Easy Auth](https://learn.microsoft.com/en-us/azure/container-apps/authentication) |
| Egress identity | ✅, [Managed identity](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity?tabs=portal%2Chttp) |  ✅, [Managed identity](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-managed-identity) | ✅, [Managed identity](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity?tabs=portal%2Cdotnet) |

## Wrapping up

Let's summarize the downsides of each offering. Container Instances are not an option if you;

- require private inbound connections, since the dynamic nature of the private IP is not practical. FQDN's are static, however, they only apply to the public IP.
- require a Microsoft managed certificate without much complexity, it can be done but involves using a sidecar container and API configuration.
- require easy validation of authorization tokens, as it doesn't come with Easy Auth or anything similar

Container Apps are not an option if you;

- require to call by IP and not by FQDN, it is not possible to use IP routing via the Container Apps environment to the Container App itself if you require VNet-scoped traffic.
- don't like the complex DNS setup it requires. It involves setting up dns zones for each Container Apps environment. To add to the complexity, each time a Container Apps environment is being built from the ground up, a new name has to be used, as there is randomness added to the FQDN.
- require a managed custom domain certificate while securing your ingress endpoint, it has to be accessible by the certification authority.

Web App for Containers is not an option if you;

- require secure connections from any other port other than 443, since you are not allowed to configure this at all.
- require a managed custom domain certificate if secured by private DNS.

Web App for Containers seems like the offering that ticks most requirements in this particular scenario, you have easy out of the box HTTPS support, secure and easy networking via private endpoints with a fixed endpoint and Easy Auth support. However, every scenario is different and should be closely examined per requirements.

## Web App for Containers sample

To see how easy it is setting up Web App for Containers in IaC, please see my fork of the `darklynx/request-baskets` image, which includes a [bicep deploy template with Azure Deployment Environments support](https://github.com/erwinkramer/request-baskets/tree/master/bicep).

## License

This work is licensed under a
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa]

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg
