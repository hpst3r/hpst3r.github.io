---
title: "Registering Azure resource providers"
date: 2025-10-26T16:30:00-00:00
draft: false
---


You must enable resource providers to use parts of Azure with a subscription. This helps protect you from compromise by making it more difficult for a malicious user with lesser privileges to activate new services in a subscription, according to Microsoft.

This is usually done automagically if you have the adequate permissions (to perform the `/register/action` operation) and try to create a resource that requires a currently-inactive provider, but you can certainly go ahead and toggle these manually.

In my case, this subscription seems a little broken and automatic registration isn't working. So let's do it manually! Why not.

### Azure Portal

I want to activate the Azure Container Instances provider in a new Azure pay-as-you-go subscription ("Azure subscription 1"). So, I'll navigate to my subscription (Subscriptions > 'Azure subscription 1') in the Azure Portal, then expand Settings in the left-side navigation pane, click 'Resource providers', and run a search for 'container'. Breathe out. Lots of clicks.

Then, you can click the provider, and click Register.

{{< figure src="images/0-az-portal.png" >}}

Like I said, lots of clicking. Lame. How else can we do this?

### Terraform

I don't think you'd usually need to register a provider if you're using Terraform as, by default, your first `terraform plan` will try to register any supported providers. I'll register a provider with Terraform anyway - just for fun.

Install the Azure CLI (`az`) and Terraform, e.g.:

```sh
brew install az terraform
```

Then, define your Terraform provider (the AzureRM provider), disable automatic resource provider registrations (or it'll initialize every supported provider as soon as you run your first `terraform plan`, as mentioned just above), specify the subscription ID, then, finally, define your resource (in this case, a resource provider registration resource.. as funny as that sounds):

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
  resource_provider_registrations = "none"
  subscription_id = ""
}

resource "azurerm_resource_provider_registration" "ContainerInstance" {
  name = "Microsoft.ContainerInstance"
}
```

Run a `terraform init` to prepare the environment (e.g., download the provider), authenticate the Azure CLI with `az login` and select a subscription, then run a `terraform plan` to preview your changes:

```txt
wporter@wm3 terraform % terraform plan -out register_aci_provider

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_resource_provider_registration.ContainerInstance will be created
  + resource "azurerm_resource_provider_registration" "ContainerInstance" {
      + id   = (known after apply)
      + name = "Microsoft.ContainerInstance"
    }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Saved the plan to: register_aci_provider

To perform exactly these actions, run the following command to apply:
    terraform apply "register_aci_provider"
```

If you're happy with what Terraform tells you, run a `terraform apply` to configure the subscription to your desired state:

```txt
wporter@wm3 terraform % terraform apply register_aci_provider
azurerm_resource_provider_registration.ContainerInstance: Creating...
azurerm_resource_provider_registration.ContainerInstance: Still creating... [10s elapsed]
azurerm_resource_provider_registration.ContainerInstance: Still creating... [1m40s elapsed]
azurerm_resource_provider_registration.ContainerInstance: Creation complete after 1m45s [id=/subscriptions/0/providers/Microsoft.ContainerInstance]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### Azure CLI

We're already logged in to the Azure CLI, so let's do this next. If you weren't logged in, you'd need to install it and authenticate to your Azure account:

```txt
brew install az || winget install Microsoft.AzureCLI
az login
```

You can work with resource providers with the `az provider` set of commands. For example, to list all providers and their properties, run `az provider list`.

For example, to see the registration status of a provider, you can find the `registrationState` property in the output of an `az provider show`:

```txt
wporter@wm3 ~ % az provider show -n Microsoft.ContainerInstance | grep registrationState
  "registrationState": "Registered",
```

To unregister or register a provider, use the `az provider (register|unregister)` commands. For example, let's unregister the provider we just registered with Terraform:

```txt
wporter@wm3 ~ % az provider unregister -n Microsoft.ContainerInstance --wait
wporter@wm3 ~ % az provider show -n Microsoft.ContainerInstance | grep -i registrationState
  "registrationState": "Unregistered",
```

### PowerShell

I'll assume you already have PowerShell 7. If you don't have the Az module installed, install it:

```PowerShell
Install-Module Az -Scope CurrentUser
```

Connect to your Azure account, and choose a subscription:

```PowerShell
Connect-AzAccount
```

To see the registration state of a provider in said subscription, you can use the `Get-AzResourceProvider` cmdlet. This will return each provider a number of times (one for each resource type), which is very annoying. Just another example of how this PowerShell module really sucks.

```txt
PS /Users/wporter> Get-AzResourceProvider -ProviderNamespace Microsoft.ContainerInstance | Select RegistrationState

RegistrationState
-----------------
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
Unregistered
```

To register the provider.. you guessed it! `Register-AzResourceProvider`!

You can shorten the `ProviderNamespace` argument to `Pro` in most cases. None of these cmdlets do positional arguments, which is also annoying and makes things extremely verbose.

```txt
PS /Users/wporter> Register-AzResourceProvider -ProviderNamespace Microsoft.ContainerInstance

ProviderNamespace : Microsoft.ContainerInstance
RegistrationState : Registering
ResourceTypes     : {containerGroups, serviceAssociationLinks, locations, locations/capabilities…}
Locations         : {Australia Central, Australia Central 2, Australia East, Australia Southeast…}
```

```txt
PS /Users/wporter> Get-AzResourceProvider -ProviderNamespace Microsoft.ContainerInstance | Select RegistrationState | Select -First 1

RegistrationState
-----------------
Registered
```

Anyway. Kind of pointless, but what else would one do on a Sunday afternoon but toggle free stuff in Azure?

Now, on to the next one!
