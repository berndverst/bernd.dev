+++
author = "Bernd Verst"
title = "From Cloud Security Alert to Open Source Bugfix"
date = 2020-04-08T10:06:00-07:00
description = "My journey from mitigating a security alert to addressing the root cause in several open source projects."
tags = [
    "azure",
    "kubernetes",
    "kubeflow",
    "open source",
]
categories = [
    "cloud computing",
    "security",
    "troubleshooting"
]
images = ["/img/securityalerts/godeeper.jpg"]
+++

Perhaps you've seen vulnerability reports in your CI/CD pipeline or tools like NPM. Cloud infrastructure has these too and I was surprised to get an alert. Naturally, I had to investigate to see where I went wrong... (and of course mitigate the problem).
<!--more-->
## The security alert
It all started a few weeks ago when I received an email from a colleague with the following content.

{{< alert danger >}}
**Secure transfer to storage accounts should be enabled**  
Subscription ID: devrel-berndverst-demo-test  
Resource: /subscriptions/XXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXXX/ resourcegroups/kubeflowrelease/providers/ microsoft.storage/storageaccounts/ > fairingXXXXXXXXXXXXXXXXX  
[1-Click mitigation link](https://portal.azure.com/#blade/Microsoft_Azure_Security/SecurityMenuBlade/5/menuIdToOpen/5/externalSourceName/S360/menuItemBladeParameters/%7B'searchBarFilter'%3A%20'Secure%20transfer%20to%20storage%20accounts%20should%20be%20enabled'%7D?WT.mc_id=bernddev-blog-beverst)

*Note: several similar results have been omitted*
{{< /alert >}}

This was my gut reaction:  
> 😱*This **cannot** be. What did I do wrong? Did I mess something up?* 😰

My colleague had used [Azure Security Center](https://docs.microsoft.com/azure/security-center/security-center-intro?WT.mc_id=bernddev-blog-beverst) to generate a report of security issues. Since I had never received such a message before (note I could have set up [email alerts](https://docs.microsoft.com/azure/security-center/security-center-provide-security-contact-details?WT.mc_id=bernddev-blog-beverst) in Azure Security Center myself) I was a bit skeptical and first verified the authenticity of the email: Email domain, Sender Police Framework (SPF) check, Domain Keys Identified Mail (DKIM) all indicated the email was legitimate.

The **1-Click mitigation** link (which I verified went to the Azure web portal domain) took me directly to the **recommendations** section in the Azure Security Center for my Azure account and jumped to the relevant entries. The [Security Center recommendations](https://docs.microsoft.com/azure/security-center/recommendations-reference?WT.mc_id=bernddev-blog-beverst) documentation also had more information on the alert I had received.

## What I did wrong (supposedly)

{{< alert warning >}}
Azure Blob Storage accounts should be configured to only serve traffic over **https**.
{{< /alert >}}

The 1-step mitigation in Azure Security Center resolved the issue, but I could also have followed [these instructions](https://docs.microsoft.com/azure/storage/common/storage-require-secure-transfer?WT.mc_id=bernddev-blog-beverst) in the docs.

> 🤔 Why would I ever not want secure blob transfers? This does not sound like me.

## Taking a closer look

I noticed that all affected blob storage accounts had **programmatically generated names**. Furthermore they all resided in my Azure resource group `kubeflowrelease` and contained the string `fairing` in the account names.

Days earlier I had ported an [end to end tutorial for Kubeflow](https://github.com/kubeflow/examples/tree/master/mnist#azure) using the MNIST training set to Azure. Of course in the process I deployed [Kubeflow](https://www.kubeflow.org) to my Kubernetes cluster and went through the tutorial I wrote.

{{< alert info >}}
The Kubeflow project is dedicated to making deployments of machine learning (ML) workflows on Kubernetes simple, portable and scalable.
{{< /alert >}}


## The culprit: Kubeflow
> 💡Kubeflow somehow created the storage accounts in question

The Kubeflow Fairing helper library [created the storage account without forcing secure transport](https://github.com/kubeflow/fairing/blob/c7faf298221d8f28ae2e4794e85ac3d967c6d1e8/fairing/cloud/azure.py#L74).



#### How to correctly create secure storage accounts

Looking at the [relevant documentation](https://docs.microsoft.com/azure/storage/common/storage-require-secure-transfer?WT.mc_id=bernddev-blog-beverst) I discovered something very interesting:

{{< alert info >}}
By default, the Secure transfer required property is enabled when you create a storage account in Azure portal. However, it is disabled when you create a storage account with SDK.
{{< /alert >}}

**In other words, the SDKs default to doing that for which the Azure Security Center alerted me, creating insecure storage accounts.**

> ❓Kubeflow Fairing is written in Python. I wonder how to create secure storage accounts with the Azure Python SDK.

The Python SDK sample code at [github.com/Azure-Samples/storage-python-manage](https://github.com/Azure-Samples/storage-python-manage/blob/31c5e908cb5f5dc0393523e83eea3ab2f422d332/README.md) does not explicitly set the secure option. **The sample code is bad and uses uses the insecure default**. I made [this pull request](https://github.com/kubeflow/fairing/pull/477) to fix this.

{{< gist berndverst b1ec0fbb7ab2edbb846cbf43a4338349 azure-sample-storage-account-create.py >}}

Fixing the sample code **led to several unrelated issues with the tests**, all of which I fixed:
- The Azure Python SDK version was not locked and the API output had changed, no longer matching the mocked responses
- Travis was running tests for a version of Python that was no longer supported
- Travis was not running tests for current versions of Python 3.7 and 3.8.

It took me hours, but [Azure-Samples/storage-python-manage#11](https://github.com/Azure-Samples/storage-python-manage/pull/11) fixed all of the above.

#### What did Kubeflow Fairing do?

Taking a close look we can see that Kubeflow Fairing basically [copied from the Azure Python sample code](https://github.com/kubeflow/fairing/blob/c7faf298221d8f28ae2e4794e85ac3d967c6d1e8/fairing/cloud/azure.py#L74).
> *I would have done the same.* Good thing that is fixed now!

I made [this pull request](https://github.com/kubeflow/fairing/pull/477) to fix the issue in Kubeflow Fairing.

{{< gist berndverst b1ec0fbb7ab2edbb846cbf43a4338349 kubeflow-azure-storage-account-creation.py >}}

## Success?

{{< alert success >}}
- ✅ Kubeflow Fairing now creates secure storage accounts.
- ✅ Anyone finding the Azure Python SDK samples for storage account creation will create secure accounts.
{{< /alert >}}

{{< alert info >}}
**Solution**:  
A new version of the Azure Storage Resource Provider API should **default to creating secure storage accounts** if the `supportsHttpsTrafficOnly` parameter has not been provided.
{{< /alert >}}

Anyone who updates the Azure CLI or Storage SDK will **automatically inherit secure defaults**. Those who have been relying on the insecure behavior (this should be few people) will experience a breaking change, but they of course are **not required to upgrade SDKs or API versions** and can certainly **explicitly set the parameter**.

{{< alert success >}}
Exact what I suggested here was indeed done since [API version 2019-04-01](https://azuresdkdocs.blob.core.windows.net/$web/python/azure-mgmt-storage/8.0.0/azure.mgmt.storage.v2019_04_01.models.html#azure.mgmt.storage.v2019_04_01.models.StorageAccountCreateParameters). All current SDKs will be using this API version or a newer one which defaults to the secure setting.
{{< /alert >}}

### So why the Kubeflow Fairing issue afterall?

Kubeflow Fairing was using the deprecated `azure` library. Instead `azure-mgmt-storage` should have been used. **The edits to the Azure Python sample documentation weren't strictly necessary.** But at least I fixed the tests.

![We must go deeper](/img/securityalerts/godeeper.jpg)