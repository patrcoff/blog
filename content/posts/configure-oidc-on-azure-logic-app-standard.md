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

This sounds like something which should be fairly trivial given that OIDC is an MS implementation of OAuth and is natively supported for many resource types however there were several stumbling blocks faced when figuring out how to do this.

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