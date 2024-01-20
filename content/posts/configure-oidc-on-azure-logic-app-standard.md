+++
title = 'Configure Oidc on Azure Logic App Standard'
date = 2024-01-18T09:47:43Z
draft = true
+++

> [!NOTE] - to be removed
> The below section outlines reference materials which are to be properly referenced within the post. I am leaving them here during drafting for simplicity. Most importantly ensure to properly reference and attribute to the non-MS resources such as hybridbrothers.

- [blog post showing clearly how to do it for consumption](https://hybridbrothers.com/using-managed-identities-in-logic-app-http-triggers/#allowing-multiple-identities)
- [MS tech community guide (officially linked from ms docs)](https://techcommunity.microsoft.com/t5/azure-integration-services-blog/trigger-workflows-in-standard-logic-apps-with-easy-auth/ba-p/3207378)
- [GH issue with mention re azapi tf usage](https://github.com/hashicorp/terraform-provider-azurerm/issues/12928)
- [MS docs](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app?tabs=azure-portal#enable-microsoft-entra-id-open-authentication-microsoft-entra-id-oauth)

Problem statement: My client requires a Logic App Standard which will have a workflow triggered by HTTP request with a JSON payload. There is a requirement that the Logic App sit behind APIM and that the caller be authenticated and authorised via OIDC/OAuth using an identity so that only the caller can trigger the workflow.

This sounds like something which should be fairly trivial given that OIDC is an MS extension of OAuth and is natively supported for many resource types however there were several stumbling blocks faced when figuring out how to do this.

- my own ignorance/inexperience with regards to OAuth and the related concepts and terminology
- differences between Logic Apps consumption vs standard
- the MS docs tendancy to talk by default about consumption as well as external pages referenced by the official docs often not stating which tier was being discussed.
- the lack of a way to 'turn on' the authv2 feature for the logic app natively via portal
- the lack of any way to interact with the authv2 feature for the logic app via terraform
- confusion about what clientId, allowedAudiences etc were to be
- different information from different sources
- considerations of best practice (i.e. allowing http auth headers into workflow outputs)
- calling the azapi terraform resource
- deconstructing the example json payload of the manual az rest api call example
- lastly, mention about how to run the az api locally as too many posts skip that and it's not actually that easy to find in the ms docs, bunch of circular links etc

## What About IP Restrictions?

Some of you might be thinking why not simply restrict the IPs which are allowed to trigger the workflow to the APIM IP(s) only, but this only goes _so far_ in meeting the client's goals. They want to ensure categorically that _only the specific system_ making the original HTTP request to APIM can trigger the workflow, restricting the IP ranges allowed to reach the Logic App only ensures than the trigger came _from APIM_, not which specific origin service sent the request. 

Further, by default, the 'When a HTTP request is recieved' trigger generates an endpoint URL which **includes the _SAS token_** for the workflow. This is a secret and should not be readily shared between teams, commited to source control etc. It's really not a highly secure way to authenticate and authorise triggering of the workflow but the standard workflow development flow does not make this obvious. Anyone with this URL could trigger the workflow which is definitely not ideal!

This issue is discussed in detail in [this](https://hybridbrothers.com/using-managed-identities-in-logic-app-http-triggers/#allowing-multiple-identities) post by hybridbrothers, which I have based a significant portion of this article on. However, that post discusses Logic App _Consumption_ and not Logic Apps _Standard_ - crucially, there are differences in the two Logic App types which make implementing OAuth more complicated, or at least less obvious, for Standard Workflows. Robbe does a great job of explaining the 'how' and 'why' of implementing managed identity authentication for Logic App Consumption and I recommend reading his post, but if you too need to secure a standard workflow HTTP trigger with Managed identity or similar OIDC/OAuth implementation, then read on for the Logic App Standard specific details.

Reading Robbe's post, we understand:

- the inherent insecurity of using the SAS token including endpoint URL
- how to 'disable' SAS authentication for the workflow
- how to 'enable' OAuth on Logic Apps Consumption and create a policy which only allows certain identities to be authorised to call the endpoint. - this is what we can't do in Logic Apps Standard in via the same means.
- information about clientId, issuer, audience and other OAuth/Managed Identity concepts (though not in huge detail)
- how to set up another logic app to use as the trigger caller who is to be authenticated

> Some of this is going to be re-covered in this post for completeness and clarity but I want to make a point of thanking Robbe for his post outlining much of this information and doing a lot of the leg-work which I cannot claim to have done all myeslf.


The first problem is Logic App Standard doesn't expose the Authentication options for the App Service resource which Logic App Standard runs on behind the scenes. This is true via the portal but crucially also via the Azurerm Terraform provider. As we're working in an Enterprise environment, we must deploy all of our infrastructure for our client via IAC so whatever solutions we come up with, must be possible to define in code.