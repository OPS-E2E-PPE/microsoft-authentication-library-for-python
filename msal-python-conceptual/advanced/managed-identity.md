---
title: Using Managed Identity
description: Learn how to use Managed Identity with Microsoft Authentication Library (MSAL) for Python.
author: localden

ms.service: msal
ms.subservice: msal-python
ms.topic: conceptual
ms.date: 06/25/2024
ms.author: ddelimarsky
ms.reviewer: rayluo
---

# Using Managed Identity

[Managed identity](/entra/identity/managed-identities-azure-resources/overview) enables developers to remove the need to store, rotate, and otherwise handle credentials for their applications running in the Microsoft Azure cloud. Microsoft Authentication Library (MSAL) for Python supports using authentication with managed identity on supported Azure workloads.

>[!NOTE]
>Managed Identity in MSAL Python is supported since version [1.29.0](https://pypi.org/project/msal/1.29.0/) of the [`msal` package](https://pypi.org/project/msal/).

MSAL Python supports acquiring tokens through the managed identity service when used with applications running inside Azure infrastructure, such as:

- [Azure App Service](https://azure.microsoft.com/products/app-service/) (API version `2019-08-01` and above)
- [Azure VMs](https://azure.microsoft.com/free/virtual-machines/)
- [Azure Arc](/azure/azure-arc/overview)
- [Azure Cloud Shell](/azure/cloud-shell/overview)
- [Azure Service Fabric](/azure/service-fabric/service-fabric-overview)

For a complete list, refer to [Azure services that can use managed identities to access other services](/azure/active-directory/managed-identities-azure-resources/managed-identities-status).

## Which SDK to use - Azure SDK or MSAL?

MSAL libraries provide lower level APIs that are closer to the OAuth2 and OIDC protocols.

Both MSAL Python and Azure SDK allow to acquire tokens via managed identity. Internally, Azure SDK uses MSAL Python, and it provides a higher-level API via its [`DefaultAzureCredential`](/python/api/azure-identity/azure.identity.defaultazurecredential?view=azure-python) and [`ManagedIdentityCredential`](/python/api/azure-identity/azure.identity.ManagedIdentityCredential?view=azure-python) abstractions.

If your application already uses one of the SDKs, continue using the same SDK.

- Use Azure SDK if you are writing a new application and plan to call other Azure resources, as this SDK provides a better developer experience by allowing the app to run on private developer machines where managed identity doesn't exist.
- Use MSAL if you need to call other downstream web APIs like Microsoft Graph or your own web API.

## How to use managed identities

There are two types of managed identities available to developers - **system-assigned** and **user-assigned**. You can learn more about the differences in the [Managed identity types](/azure/active-directory/managed-identities-azure-resources/overview#managed-identity-types) article. MSAL Python supports acquiring tokens with both. MSAL Python logging allows to keep track of requests and related metadata.

Prior to using managed identities from MSAL Python, developers must enable them for the resources they want to use through Azure CLI or the Azure Portal.

## Examples

In both system- and user-assigned identities, developers need to use <xref:msal.managed_identity.ManagedIdentityClient> to access managed identities.

### System-assigned managed identities
