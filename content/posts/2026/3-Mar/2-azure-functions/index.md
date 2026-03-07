---
title: "Intro to Azure Functions"
date: 2026-03-07T17:45:00-00:00
draft: false
---

## Introduction

Azure Functions is a "serverless" PaaS offering like AWS Lambda. Its main draw is allowing you to run event-driven code, rather than containers or virtual machines (though you can still run containers if you'd like). This "serverless" platform effectively abstracts away the underlying virtual machine/compute entirely and allows you to just run your code.

And, as with any Azure product, you pay only for what you use - in this case, that means your event-driven code is only billed when it's running (per gigabyte of memory used, per second, not VMs billed per minute.. and usually left on 24/7/365). This is in contrast to something like Azure Container Instances, which will bill you for the underlying VMs regardless of actual usage (though you can scale in and scale out, you're still paying for the VMs).

For up-to-date info on Azure Functions pricing, see [the relevant pricing page](https://azure.microsoft.com/en-us/pricing/details/functions/).

Today, we'll be doing a couple things:

- Discussing the price and administrative benefits of Functions versus something like a cronjob, timer, or scheduled task in a VM for smaller scripts
- Briefly going over the various Function Apps plans (noting the distinction between the standard "serverless" offerings and cohabited hosting)
- Setting up a local development environment with Azurite and Visual Studio Code, and demonstrating running a Function there
- Deploying a Function to Azure from VS Code
- Running a lab - configuring private networking and examining Managed Identity for authentication against an Azure Key Vault with Python

Other stuff worth a look that I don't have time to demo at this instant:

- Managed Identity for authentication against SharePoint Online (doable with the Graph API)
- CI/CD with GitHub Actions

## First, an example! Where does Functions make sense for me, the humble sysadmin?

Let's model the dollars and cents with an ideal use-case!

I've got a Python script/"job" that runs for 2 - 3 seconds every 5 minutes, then for about 10 seconds once a day. I've got another that runs for about 10 more seconds once a day, ever day.

If I were to cohabit them in an Azure VM, even something like a B1s (1 vCPU, 1 Gb of RAM), I'd be paying around $10 a month (give or take). And I also have to maintain the VM (patch it, fix it if patches break it, etc).

> To be fair, I can run a lot more than the two jobs in that VM - but a gig of memory total is tight for a modern, full-fat Linux install, I don't really have more than the two to cohabit (in this environment, at the moment), and loading that machine up with stuff would substantially increase its blast radius.
>
> Never mind how much fun inheriting a random VM running "jobs" is for the next guy. Bonus points if it's missing every normal system utility for "lightness".
>
> Additionally, my time is rather limited and expensive! I would prefer to not spend it on patching/troubleshooting if I don't have a good reason to.

Now, I'll mention that Microsoft throws in 100,000 gigabyte-seconds of runtime and 250,000 executions for Functions for free, so you can get a lot done before even opening your wallet. We're going to ignore this, say it's all been used up.

Since Azure Functions is billed by runtime (in gigabytes of memory used per second), let's figure.. 3 seconds of runtime \* 12 (5 minute increments in an hour) \* 24 hours \* 30 days = ~25,920 seconds of runtime, plus the ten-second daily jobs (300 seconds total) for a sum of 26,220 seconds of runtime per month.

This script will use far less than half a gig of memory, but we'll round up to 512 Mb.. so divide that 25,920 by 2 to get our value of approximately 13,110 billable GB-s, with 8670 executions.

At the standard rate (March 2026) that's $0.000016 per GB-s, so about $0.21 a month, plus about $0.0017 in 8670 executions (so we'll round up to $0.22/mo for script 1).

Script 2 will add a nominal 150 billable gig/seconds of runtime, in 30 executions.. so it'll fit in that "round up" to $0.22/mo.

We'll be generous and say the storage account will cost another $0.88/mo. It probably won't, since it doesn't need to be fast and will be storing a few megabytes of text files, but we'll say it will.

Our total cost? Less than a tenth of the VM, with much less maintenance or management overhead. Assuming we're using up all of the 400,000 free gig/seconds allocated by Microsoft, that is.

So, serverless can make sense! Less maintenance, and less cost (just for hosting) if you refactor infrequently-run jobs or scheduled tasks. And, we're likely to gain significantly better availability, since Azure can now orchestrate our job on whichever machine is available (we no longer need to care about the VM being a SPOF).

Additionally, our backups can become our Git repository and soft-delete on the storage account or a mirror somewhere else for a couple of cents (assuming there's something persistent in the storage account).

## Function Apps

Before we get into Functions themselves, we need to go over their different hosting models.

Function Apps are containers for Azure Functions - this works along the same lines as an Azure App Service plan and individual webapps in it. In fact, Azure Functions Apps are built on some of the same logic as App Services - you'll see some of this as you work with Function Apps.

For traditional or flex consumption-based Functions, this is less relevant than it is for, say, the dedicated-compute Functions Premium plan.

We'll start with the consumption-based Function Apps plans (Flex Consumption and legacy Consumption) are truly pay-for-what-you-use - you will pay only for runtime, calculated in terms of 256 Mb blocks of memory that your code runs.

The Functions "Elastic" Premium option, on the other hand, allows you to dynamically scale (or permanently reserve) dedicated workers. In contrast to the standard Consumption plans, this means you're paying for the workers, not necessarily your Functions' runtime - more along the lines of Container Instances or App Service offerings. You can cohabit multiple Functions on one set of Premium workers, and scale down to zero or reserve instances to avoid waiting for cold starts.

> Cold starts?
>
> In a Consumption Functions plan, when a Function runs from zilch, Azure allocates the app to a worker with capacity, the worker pulls the Function from the configured storage account (incurring cost), the worker applies app settings and loads any applicable extensions (setting up the environment), and *then* the Function runs. This adds time and cost for infrequently-run operations - they may be better suited for something like Azure Automation Runbooks to minimize access costs from your storage accounts.
>
> Microsoft keeps Functions "hot" and ready to run on Functions workers for about twenty minutes post-execution. Therefore, our five-minute example above should always be "hot", but the other script that runs once a day will be a cold start every time.

Finally, the third "tier" are the cohabited Functions plans - in these examples, your Functions run on the same workers running App Service plans or Container Service plans.

To create a Function App, navigate to "Create a resource" in the Azure portal, then click "create" under "Function App".

{{< figure src="images/0-create-resource-function-app.png" >}}

Then, you'll be met a choice between the different Function App hosting options:

{{< figure src="images/1-function-app-plans.png" >}}

The types of Function App plans are:

- **Flex Consumption**: the truly 'serverless' option with virtual network endpoints and pay-as-you-use billing
- **Functions Premium**: dedicated infrastructure for lower latency
- **App Service**: as Function Apps are a derivative of App Service plans, you can also run Function Apps on existing compute dedicated to your App Services
- **Container Apps environment**: Alternatively, you can cohabit Functions with your Azure Container Apps - similarly to Functions Premium and App Service, you'll pay for the compute capacity, not your direct Functions usage.
- **Consumption**: the older 'serverless' option, with less scalability and no virtual networking. Deprecated for Linux workers.

You should choose "Flex Consumption" unless you have other special requirements. While Functions Premium may have a lower cold-start time, if this is not needed you'll be paying for the underlying compute 24/7 for no gain. Flex Consumption-based Function Apps are the most flexible and typically the cheapest option (until you start getting into extreme numbers of executions or long runtimes).

Creating a Function App plan will also create a storage account for your code, access keys, and configuration files.

## Creating and running your first Function

Now that we have a Function App plan, we can start writing a Function.

I'll be using VS Code since it's my preferred editor. It happens to have a nice plugin for Azure Functions - if you're following along, you should [install it now](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions).

If you're not using VS Code, [see Microsoft's docs on developing Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local?pivots=programming-language-python). If you are using VS Code, [additionally see Microsoft's docs on using it to develop Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=node-v4%2Cpython-v2%2Cisolated-process%2Cquick-create&pivots=programming-language-python).

You'll also want to install the Azure Functions tools. Back in the command palette, this can be run with "Azure Functions: Install or Update Azure Functions Core Tools".

{{< figure src="images/3-install-core-tools.png" >}}

On a Mac, this will install the `azure-functions-core-tools` package with Homebrew. On a Windows PC, this will install the package with Node. Alternatively, [see the docs](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Cisolated-process%2Cnode-v4%2Cpython-v2%2Chttp-trigger%2Ccontainer-apps&pivots=programming-language-python).

You may also want to install [the Azurite Azure Storage emulator](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite) - Azure Functions requires a storage account for configuration and code.

Then, in the Command Palette (Ctrl + Shift + P), search for Azure Functions: Create New Project:

{{< figure src="images/4-create-new-project.png" >}}

Select the containing directory, language, a trigger template (e.g., timer, HTTP, blob (e.g., on write to a storage account), CosmosDB, EventGrid, MCP, Queue...)

In my case, I chose a simple timer trigger, and the extension gave me a template for a Python Azure Function. I punched in a 6 AM daily cron-style schedule. VS Code handled creating me a virtual environment.

{{< figure src="images/5-new-project.png" >}}

To test out a function, for example, the "timer trigger" example, adjust it so it'll run (I set mine to run every minute during the current hour, for example) and "Start Debugging" in VS Code to run it (or activate the virtual environment and run `func start` from your shell).

> If you *don't* have the Azurite storage emulator installed or it isn't running, the Azure Functions extension will prompt you to install and/or start it when you start a Debug session from VS Code.

A successful execution of this simple timer-based function should look like:

```txt
[2026-03-02T22:29:00.133Z] Running timer...
[2026-03-02T22:29:00.134Z] Python timer trigger function executed.
[2026-03-02T22:29:00.188Z] Executed 'Functions.main' (Succeeded, Id=e163a913-323e-455a-9956-791233b49044, Duration=166ms)
```

Now, let's do something with blob storage..

I've made a couple of quick changes to the example function:

```python
import logging
import azure.functions as func

app = func.FunctionApp()

@app.timer_trigger(schedule="0 * 17 * * *", arg_name="myTimer", run_on_startup=False,
              use_monitor=False) 

@app.blob_input(arg_name="inputblob", path="example-container/input.txt", connection="BlobStorage")

@app.blob_output(arg_name="outputblob", path="example-container/output.txt", connection="BlobStorage")

def main(myTimer: func.TimerRequest, inputblob: func.InputStream, outputblob: func.Out[bytes]) -> None:
    logging.info("Running timer...")

    if myTimer.past_due:
        logging.info('The timer is past due!')

    outputblob.set(inputblob)

    logging.info('Python timer trigger function executed.')

```

I've also connected to Azurite with Azure Storage Explorer, created a container, and seeded it with an "input.txt" file.

{{< figure src="images/6-storage-explorer-azurite-blobs.png" >}}

{{< figure src="images/7-example-container.png" >}}

And, finally I've populated a configuration value, `BlobStorage`, in the `local.settings.json` file in my repository (with `UseDevelopmentStorage` to tell it to use my Azurite local emulated storage). Here's my full `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "BlobStorage": "UseDevelopmentStorage=true"
  }
}
```

When I "Start Debugging", I see a successful execution on a timer, and I see my input file has been modified:

```txt
[2026-03-02T22:20:59.068Z] Host lock lease acquired by instance ID '0000000000000000000000002C4D9616'.
[2026-03-02T22:21:00.035Z] Executing 'Functions.main' (Reason='Timer fired at 2026-03-02T17:21:00.0199055-05:00', Id=ae2cc464-cbc2-471f-8b6c-7d28e030c092)
[2026-03-02T22:21:00.135Z] Running timer...
[2026-03-02T22:21:00.136Z] Python timer trigger function executed.
[2026-03-02T22:21:00.195Z] Executed 'Functions.main' (Succeeded, Id=ae2cc464-cbc2-471f-8b6c-7d28e030c092, Duration=172ms)
```

{{< figure src="images/8-output.png" >}}

## Deploying to Azure

To deploy your Function, use the "Azure Functions: Deploy to Azure" command in the command palette, and select a Function App.

The deployment process will archive (zip) up your working directory, ship it to Azure, and deploy it to the Function App (overwriting anything currently present in the Function App). The code will live in the storage account provisioned for the Function App at creation time.

{{< figure src="images/9-deploy.png" >}}

{{< figure src="images/10-deployed.png" >}}

Now, if you browse to the Function App and select the Function, you'll get a Monaco text-editor iframe - however, since we deployed the Function with VS Code, we can't edit it from the portal.

{{< figure src="images/11-click-function.png" >}}

{{< figure src="images/12-example.png" >}}

Now, let's try to access the VM's files. They'll be in the storage account that was created with the Function App. You'll probably have to grant yourself the Blob Storage Contributor/Owner IAM role on the storage account to access the data it contains.

{{< figure src="images/13-storage-explorer-func-data.png" >}}

Opening up the app-package container will get you to a .zip archive of the Function. Downloading and opening it will get you back to where you started - the initial bits of code:

{{< figure src="images/14-func-data-2.png" >}}

{{< figure src="images/15-package.png" >}}

And the webjobs-secrets container will contain the Function's settings (and secrets, obviously), for example:

{{< figure src="images/16-secrets.png" >}}

```JSON
{
  "keys": [
    {
      "name": "default",
      "value": "CfDJ8AAAAAAAAAAAAAAAAAAAAAB6sBwj5USi_tF0J-KYhsVoilHFZ8qlQPfWIfA_ngDnXNoMjnjgftOKBDFFSi2p1Yc1DSpDsb4gW_8GlggeIvBzzoCj50c1KjJdUS3bUHbimnodW199HVjc4FKF9w1VAJ58oU9DaH9JsHHGRgnWN1nZ7m95LOpNnjqrH9Sv9T4kwA",
      "encrypted": true
    }
  ],
  "hostName": "cctvfifteenlineage8gerry-hucufxgqfvf7athn.eastus-01.azurewebsites.net",
  "instanceId": "000000000000000000000000588AF602",
  "source": "runtime",
  "decryptionKeyId": "AzureWebEncryptionKey=dgk9izDAuohGA9z1KJQ9q4ybcJooUfHj37eRPrX+HrY=;"
}
```

## Containerized Functions

You can also run Functions in a container. You're intended to take [one of several provided base images](https://mcr.microsoft.com/catalog?search=functions) and customize it with a Dockerfile. This effectively allows you to make use of Functions event-driven orchestration with a more flexible runtime environment.

## Lab: Access a certficate in Azure Key Vault with Managed Identity from an Azure Function

Let's do some secrets!

We'll access an Azure Key Vault with Managed Identity over a private network to retrieve a certificate for our Python application (that we'll use to SSH to a VM and SCP a file).

First, I've spun up a virtual network, "function-vnet", with the 172.20.128.0/17 IP range, and created two subnets: "sub1" (172.20.128.0/24) for the VM, and "sub2" (172.20.129.0/24) for the Function App.

I've also created the Azure DNS zone "azure.lab.wporter.org" and configured a delegation to it from my public authortiative nameservers (Cloudflare).

Then, I've deployed an AlmaLinux B1s with a public IPv4 address and keypair auth, and configured its public IP as an alias record to "ssh-target.azure.lab.wporter.org".

Finally, I've created an Azure Key Vault:

{{< figure src="images/17-create-key-vault.png" >}}

We'll be using RBAC:

{{< figure src="images/18-with-rbac.png" >}}

Then, under the networking configuration, I've left public access on (but restricted it to 'selected networks' - I'll whitelist my public IP shortly for local testing), ticked the "allow trusted Microsoft services to bypass this firewall" box (so we can interact with the key vault from the Azure portal), and configured virtual network integration for the subnet I'll be deploying the Function App into ("func-sub" in my "function-vnet") to permit access:

{{< figure src="images/19-vnet.png" >}}

I've then granted myself Key Vault Administrator IAM permissions on the resource (so we can upload secrets - we'll do this in a little bit), and whitelisted my public IPv4 address on the key vault (Settings > Networking > Firewall > Add your client IP address).

Then, on an administrative machine, I've configured an example SSH certificate authority:

```txt
wporter@wm3 af-akv % mkdir .ca
wporter@wm3 af-akv % cd .ca
wporter@wm3 .ca % ssh-keygen -t rsa -b 4096 -f af-akv-ssh-ca.key
Generating public/private rsa key pair.
Enter passphrase for "af-akv-ssh-ca.key" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in af-akv-ssh-ca.key
Your public key has been saved in af-akv-ssh-ca.key.pub
The key fingerprint is:
SHA256:o7n9bBzkqgHp7c381opnrLF15oiXYYahvN/lf4Ymol0 wporter@wm3
The key's randomart image is:
+---[RSA 4096]----+
|                 |
|                 |
|                 |
|     .  . .      |
|    o. .S=       |
|   . ooo..*      |
|    . =..*.=E  . |
|     ..B.X@Oo o o|
|      +o%@Ooo+.o |
+----[SHA256]-----+
wporter@wm3 .ca % scp af-akv-ssh-ca.key.pub ssh-target.azure.lab.wporter.org:~/
af-akv-ssh-ca.key.pub                                        100%  737    42.5KB/s   00:00
```

And I've copied its public key to the Azure VM:

```txt
[wporter@af-ssh-target ~]$ sudo mv af-akv-ssh-ca.key.pub /etc/ssh/
[wporter@af-ssh-target ~]$ sudo tee -a /etc/ssh/sshd_config > /dev/null << 'EOT'
> TrustedUserCAKeys /etc/ssh/af-akv-ssh-ca.key.pub
> EOT
[wporter@af-ssh-target ~]$ sudo systemctl restart sshd
```

I'll also create a user for the Functions script:

```txt
[wporter@af-ssh-target ~]$ sudo useradd -m functionuser
```

And I'll create a SSH certificate for the new user:

> Note that I'm making this cert valid forever. This is not good practice, but the resource group has already been obliterated at the time of posting, so...

```txt
wporter@wm3 .ca % ssh-keygen -t rsa -b 4096 -C "Azure Functions user" -f functionuser.key
Generating public/private rsa key pair.
Enter passphrase for "functionuser.key" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in functionuser.key
Your public key has been saved in functionuser.key.pub
The key fingerprint is:
SHA256:1U6QYfnnkyBINjb6E1DCYb5s2FJwNmryZ1bHaqlgp34 Azure Functions user
The key's randomart image is:
+---[RSA 4096]----+
|    ..Bo. ++     |
|     Boo*ooo     |
|  . o o*.++.o    |
|   + =.oo=.oo .  |
|    * X.S. ..+ . |
|   . X oo     +  |
|    . .  .     . |
|   .  E          |
|    ..           |
+----[SHA256]-----+
wporter@wm3 .ca % ssh-keygen -s af-akv-ssh-ca.key \
> -I "user_functionuser" \
> -n "functionuser" \
> -V "-1w:forever" \
> -z $RANDOM \
> "functionuser.key.pub"
Signed user key functionuser.key-cert.pub: id "user_functionuser" serial 15796 for functionuser valid after 2026-02-25T05:54:07
```

Finally, I'll confirm I can access the user account with the SSH certificate:

```txt
wporter@wm3 .ca % ssh -i functionuser.key functionuser@ssh-target.azure.lab.wporter.org
[functionuser@af-ssh-target ~]$
```

Now that our certs are working, I'll upload the public and private keys to the key vault as secrets.

> I'm uploading these as secrets, not certificates, because OpenSSH certs and X509 certs are different things. Azure Key Vault certificate objects are restricted to X509 certs, while secrets can be arbitrary strings.
>
> Also note that, if using the Azure portal, newlines will be stripped (so you should base64-encode the file if you intend to work with it this way). That's why I'll be `b64decode`'ing the string we retrieve in my script.

{{< figure src="images/20-create-secret.png" >}}

{{< figure src="images/21-create-secret.png" >}}

{{< figure src="images/22-secrets.png" >}}

To retrieve this secret, all we'll have to do is grant access to it to the managed identity of the Function, and then, more or less:

```Python
credential = DefaultAzureCredential()
secret_client = SecretClient(vault_url="https://af-demo-akv.vault.azure.net", credential=credential)

public_key = (secret_client.get_secret("functionuser-signed-public-key")).value
```

Here's a minimal working example to test locally (connecting to the Key Vault from a workstation):

```Python
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

def main() -> None:

  key_vault_name = "af-demo-akv"
  key_vault_uri = f"https://{key_vault_name}.vault.azure.net/"

  credential = DefaultAzureCredential() # this will automatically retrieve managed identity.
  # to test, log in to the azure cli with az login and it will use that credential instead
  secret_client = SecretClient(vault_url=key_vault_uri, credential=credential)

  public_key = (secret_client.get_secret("functionuser-signed-public-key")).value
  private_key = (secret_client.get_secret("functionuser-private-key")).value

  print(f"public key: {public_key}")

if __name__ == "__main__":
    main()
```

Once I've confirmed that works, I'll whip together a quick Function to retrieve the public and private key from our Key Vault, then connect to our host and write a "function-was-here.txt" file.

If you add requirements, be sure to update your requirements.txt to reflect the changes so your environment in Azure can properly pull the dependencies in!

> Note: I use the SSH AutoAddPolicy to automatically accept the VM's host key. Don't do this in production! Pre-populate your known_hosts, or configure a RejectPolicy with explicit host key verification at risk of a MITM. This is equivalent to accepting any self-signed SSL cert.

```Python
import logging
# import modules from paramiko to handle SSH connections and keys
from paramiko import Message, RSAKey, SSHClient, AutoAddPolicy
# StringIO is used to convert the private key string into a file-like object that paramiko can read from
from io import StringIO
# we base64 encode the private key before storing it in Key Vault to preserve formatting, so we need to decode it before using it
from base64 import b64decode
# azure imports
import azure.functions as func
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

app = func.FunctionApp()

# run every 5 minutes
@app.timer_trigger(schedule="0 */5 * * * *", arg_name="myTimer", run_on_startup=False, use_monitor=False)

def write_file_on_timer(myTimer: func.TimerRequest) -> None:

    key_vault_name = "af-demo-akv"
    key_vault_uri = f"https://{key_vault_name}.vault.azure.net/"

    credential = DefaultAzureCredential()
    # this will automatically retrieve managed identity. to test, log in
    # to the azure cli with az login and it will use that credential instead

    # initialize a keyvault.secrets.SecretClient to easily interact with Key Vault
    secret_client = SecretClient(vault_url=key_vault_uri, credential=credential)

    if myTimer.past_due:
        logging.info('The timer is past due!')

    logging.info('Python timer trigger function executed.')

    private_key_b64_string: str | None = (secret_client.get_secret("functionuser-private-key")).value
    cert_string: str | None = (secret_client.get_secret("functionuser-signed-public-key")).value

    if private_key_b64_string is None:
        print("Failed to retrieve private key from Key Vault.")
        return

    if cert_string is None:
        print("Failed to retrieve certificate from Key Vault.")
        return

    # decode the base64 encoded private key
    private_key_string: str = b64decode(private_key_b64_string).decode('utf-8')
    
    # convert the string private key to a paramiko.RSAKey object
    private_key: RSAKey = RSAKey.from_private_key(StringIO(private_key_string), password=None)

    # grab the base64 encoded cert from the openssh formatted cert string
    # decode it, and load it into the private key object so paramiko can
    # use it for authentication - this is required for openssh cert auth
    cert_b64: str = cert_string.split()[1]
    cert_msg: Message = Message(b64decode(cert_b64))
    private_key.load_certificate(cert_msg)

    # alternatively, store parameters as env variables in the function app
    hostname: str = "ssh-target-private.azure.lab.wporter.org"
    username: str = "functionuser"
    port: int = 22

    with SSHClient() as ssh:
        # this will automatically add the host key to known hosts, which is fine for this demo but not recommended for production use
        ssh.set_missing_host_key_policy(AutoAddPolicy())
        ssh.connect(
            hostname=hostname,
            port=port,
            username=username,
            pkey=private_key
        )

        with ssh.open_sftp() as sftp:
            with sftp.file('function-was-here.txt', 'w') as file:
                file.write('This file was created by an Azure Function using a private key stored in Key Vault!')

```

I'll deploy it to Azure (from VS Code, again) in this case, creating a new Flex Consumption Function App plan.

Then, I'll configure a "virtual network integration" for outbound traffic. In the Azure portal, navigate to the Function App, expand the Settings dropdown, and select Networking. Then, click the link next to "Virtual network integration", and pick a subnet to dedicate to the Function App.

Once done, your outbound traffic will be routed through the virtual network you chose:

{{< figure src="images/23-function-app-networking.png" >}}

Then, let's get Managed Identity working! By default, a Function App will be associated with a user-assigned Managed Identity - confirm this under Function App > the app > Settings > Identity > User assigned:

{{< figure src="images/24-identities.png" >}}

Per [Microsoft's best practice](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide?tabs=azure-cli#best-practices-for-individual-keys-secrets-and-certificates-role-assignments), we're going to dedicate this Key Vault to this application and grant the Key Vault Secrets User role to the Managed Identity for the Function App. In the IAM tab for the Key Vault, select "Add role assignment" and the "Key Vault Secrets User" role.

{{< figure src="images/25-rbac-key-vault.png" >}}

Under the role members dialog, tick the "Assign access to: Managed identity" radio, and select the Function's managed identity.

{{< figure src="images/26-assign-access.png" >}}

> Since you can assign multiple user-defined Managed Identities to a resource, you'll need to either:
>
> - Specify the client ID for the Identity you're using in the `AZURE_CLIENT_ID` environment variable
> - Initialize the Credential with the ManagedIdentityClientId parameter.

Then, I'll run the Function - hopefully it'll write to the `function-was-here.txt` file, like so:

```txt
[wporter@af-ssh-target ~]$ sudo ls /home/functionuser/
function-was-here.txt
```
