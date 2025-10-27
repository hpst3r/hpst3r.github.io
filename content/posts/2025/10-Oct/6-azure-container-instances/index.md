---
title: "Azure Container Instances fundamentals & quick lab"
date: 2025-10-26T20:30:00-00:00
draft: false
---


Let's wrap a basic Python script in a Docker container and run it in Azure Container Instances. Additionally, we'll set up Azure Monitor to alert us when the container fails. Pretty simple!

This will target the following areas of the AZ-104 study guide:

- Create and configure storage accounts
- Provision and manage containers
  - Create and manage an Azure container registry
  - Provision a container using Azure Container Instances

The benefits of using Azure Container Instances are many:

- Simple management. No underlying OS.
- Very affordable pricing for compute if the service doesn't need to keep running 24/7, though you will need to orchestrate the container *somehow* - more on that in a bit.
  - $0.0000015/gb of memory per second, ~70% off for Spot
  - $0.0000135 per vCPU per second, ~70% off for Spot

For a real-world example, at $DayJob, I recently spun up a service. It runs in a Podman container on a Linux machine, because this was the simplest way to run it in our existing infrastructure.

The container uses far less than 100mb of memory and far less than a core when it does become active (most of its runtime is waiting on API requests). It only has a couple meg of data (processed log files). The rest is just sitting there!

This is *incredibly* wasteful in terms of CPU, memory, and disk, especially when you consider that the container hosting the service runs for about three seconds every five minutes, and an extra four seconds once a day.

This means that it'll run for about 14.5 minutes a day (870 seconds). 870 seconds! It costs ~2 Gb of memory on the hypervisor, and gives me a whole VM to manage!

Compare that to the cost for Azure Container Instances compute. Theoretically, this would be:

- 870s * $0.0000015 per GB * 0.5 GB = $0.0006525 a day
- 870s * $0.0000135 per vCPU * 0.5 vCPUs = $0.0058725 a day

That's a total of about $0.006525 a day (a little more than half a cent). About $0.20 a month. *Far* less than it would cost for me to manage the single VM to run this same workload locally.

Obviously, this doesn't include a public IPv4 address (~$0.005/hr) or storage, but the compute costs are extremely low.

Before we get too far along, I will mention that there's one rather frustrating catch - Azure Container Instances itself doesn't support scheduling containers; you'd need to use Logic Apps or something else making API requests to coordinate that. I won't be going over this here in the interest of time (it's not a topic for the AZ-104) but I do want to mention this omission since it's pretty annoying and means that Container Instances is much less cost-effective unless you jump through some hoops.

Anyway, whatever, who cares about limitations. It's a Microsoft product, of course it has silly omissions. Let's go over what you need to know to run a container in Azure Container Instances, run a Hello World, then build and run a custom Python script.

## What you need to know

### Compute details

Azure Container Instances spins up a VM worker per container group to run your Linux containers, or runs Windows containers with Hyper-V isolation (lightweight VMs pretending to be containers).

Since Azure Container Instances runs Windows containers with Hyper-V isolation, container groups aren't really a thing (since containers are each a VM), and thus you can't add more than one container to a Windows container group.

Linux containers in the same container group run on the same host. Since we're just putting one Linux container in a group, this means we're going to be running a VM to run our container. Azure will manage this VM's existence; it is abstracted away for you.

### Compute options

Container Instances SKUs are Standard or Confidential. The Confidential SKU uses AMD SEV-SNP to provide a cordoned-off, hardware-isolated (memory is transparently encrypted) container host VM (in other words, Confidential means that the VM hosting your containers is run in a 'trusted execution environment').

You can allocate between 0.1 and 32 vCPUs to your container, and between 0.1 - 256 Gb of memory. In some regions, you can also attach a GPU, though the offerings aren't all that great (Kepler is still around! WTF Microsoft.) Billing is per-second, rounded up.

If the container is for something low-priority, doesn't need to be exposed to the Internet, and can tolerate infrastructure loss, you can run it with the Azure Spot discount; this gives you a large price cut with the understanding that, if the infrastructure is required for other workloads, you'll be temporarily deallocated.

I believe that you'll need to put in a support request to get your quota raised from 0 for ACI Spot cores, or to get more than a Tesla K80 (half a K80?)

### Networking

When you create a container, you'll have the option to choose from "Public", "Private" and "None" for networking.

You can directly expose Azure Container Instances containers to the Internet; if you want, you can get a public IP address (choose 'Public'). Azure will create a custom reachable hostname for you, too (`customname.azureregion.azurecontainer.io`) if desired.

If you choose 'Private', you can attach your container to a virtual network (or create a new vNet to attach it to).

Both Public and Private networking will automatically create a NSG for/on the network endpoint.

If you choose 'None', the container won't have network access.

### Storage

Persistent storage is provided via direct mounting of Azure Files shares. This is the only supported option for adding persistent storage to your Azure Container Instances resources. The entire share is mounted; CIFS is used, and only Linux containers can do this - Windows containers aren't supported.

Mounted Azure Files shares support concurrent read-write access from multiple containers and other systems or services simultaneously.

Containers running on the ACI platform are assigned a set amount of ephemeral storage depending on core count (1, 2, 4, 8 Gb, doubling with every quarter of a vCPU from 0.25 to 1.) It's not possible to expand this. This storage is lost when the container is shut down, restarted, or deleted.

You can use replica-scoped storage to transfer data between replica containers; this persists through restarts and is mountable and writable by all containers in the single replica.

### Environment variables/secrets

You can pass 'secure' or insecure env variables to your container (obviously). These are all managed by Azure. I assume the 'secure' environment variables are stored in encrypted files like Podman secrets (or something along those lines) but who knows; this doesn't seem to be publicized.

Microsoft recommends using Azure Key Vault rather than environment variables for secrets.

## Creating our containers

Let's run some containers with Container Instances.

First, we'll create a basic one running an NGINX webserver from the Quickstart library.

Then, we'll build and deploy our own little app into a container hosted in an Azure Container Registry (ACR) with persistent storage courtesy of Azure Files.

### Hello World from the Azure Portal

In the Azure portal, I'll navigate to Container Instances, then click Create to make a new one.

On the 'Basics' page of the 'Create container instance' wizard, I'll tell Azure to create a new resource group in my subscription and populate the container's name, region (USE2), SKU, image source, image, and size.

Since this is just a test instance, 0.1 vCPU and 0.5 Gb of memory is perfectly adequate. The VM worker node's overhead isn't included here; you aren't billed for this, just for the actual container size. I'll choose the NGINX image from the Quickstart selection.

{{< figure src ="images/0-create-container.png" >}}

Clicking on to Networking, I'll choose Public networking (which will create a public IPv4 address for my container) and select a DNS name. I'll leave the default port 80 allow NSG rule in place.

{{< figure src ="images/1-container-networking.png" >}}

I'll just turn off logging on the Monitoring tab so I'm not billed via another resource group. Then, under Advanced, I'm just going to make two environment variables for fun (they won't do anything here). Note that you can also set the restart policy to Always, Never, or On failure (as pictured).

{{< figure src ="images/2-container-env.png" >}}

Finally, I'll validate and create the container. Once the deployment runs, I can find my new container under Container Instances.

{{< figure src ="images/3-container-overview.png" >}}

If I navigate to the FQDN or IPv4 address assigned by Azure, I'll see the NGINX hello world page:

{{< figure src ="images/4-nginx-hello.png" >}}

Then, I'll go ahead and kill the resource group to get rid of this container.

Note how we didn't really have to think about anything but the container itself - that's the main benefit of Azure Container Instances. It's a great way to run a single or small number of simple containers that don't need complex orchestration.

### Custom Hello World with the Azure CLI and Azure Container Registry

Now, let's write a little Hello World script in Python that writes to a file. We'll build the image with an Azure Container Registry task, then run the custom image with Azure Container Images (and have it write to an Azure Files share to demonstrate persistent storage).

First, I'm going to create a resource group and container registry with the Azure CLI:

```sh
az group create -n acr-rg -l eastus2
```

Then, I'll create a new container registry in the resource group (the registry's name must not already be used):

```sh
az acr create -g acr-rg -n acr5beep294 --sku basic
```

I'll also enable admin user access to the container registry (alternative to anonymous pull, which isn't supported in a Basic ACR, or Entra identity, which is marginally more complex to use):

```bash
az acr update -n acr5beep294 --admin-enabled true
az acr credential show -n acr5beep294
```

I've cast the username and password output by `acr credential show` to `$ACR_USER` and `$ACR_PASS`; we'll use them when we create the container.

I've got a Containerfile in my project directory (at `/container/Containerfile`) containing the following quick definition:

```Dockerfile
FROM python:3.13-slim

COPY script.py /app/script.py

WORKDIR /app

CMD ["python", "script.py"]
```

The `script.py` script is very simple:

```python
import time

print("Starting script...")

with open("/data/hello.txt", "w") as f:
    f.write(f"hello world at {time.time()}\n")
    print("File written successfully.")

print("Script execution completed.")
```

I'll build the container with `az acr build`:

```sh
az acr build --image images/hello-write -r acr5beep294 -f ./container/Containerfile ./container
```

```txt
wporter@wm3 aci % az acr build --image images/hello-write -r acr5beep294 -f ./container/Containerfile ./container
Packing source code into tar to upload...
Uploading archived source code from '/var/folders/qz/5s9rql4n3bl45v4p2mw81jlc0000gn/T/build_archive_2f5e4edf517d4626af30226112f82e91.tar.gz'...
Sending context (599.000 Bytes) to registry: acr5beep294...
Queued a build with ID: ch1
Waiting for an agent...
2025/10/26 23:04:32 Downloading source code...
2025/10/26 23:04:33 Finished downloading source code
2025/10/26 23:04:33 Using acb_vol_4477df4d-6e0b-4b84-8213-ad21a360dc2a as the home volume
2025/10/26 23:04:33 Setting up Docker configuration...
2025/10/26 23:04:34 Successfully set up Docker configuration
2025/10/26 23:04:34 Logging in to registry: acr5beep294.azurecr.io
2025/10/26 23:04:35 Successfully logged into acr5beep294.azurecr.io
2025/10/26 23:04:35 Executing step ID: build. Timeout(sec): 28800, Working directory: '', Network: ''
2025/10/26 23:04:35 Scanning for dependencies...
2025/10/26 23:04:35 Successfully scanned dependencies
2025/10/26 23:04:35 Launching container with name: build
Sending build context to Docker daemon  4.096kB
Step 1/4 : FROM python:3.13-slim
3.13-slim: Pulling from library/python
38513bd72563: Pulling fs layer
89fe90952b6b: Pulling fs layer
0ee66acd8266: Pulling fs layer
303fe1bfb649: Pulling fs layer
303fe1bfb649: Waiting
89fe90952b6b: Verifying Checksum
89fe90952b6b: Download complete
303fe1bfb649: Verifying Checksum
303fe1bfb649: Download complete
0ee66acd8266: Verifying Checksum
0ee66acd8266: Download complete
38513bd72563: Verifying Checksum
38513bd72563: Download complete
38513bd72563: Pull complete
89fe90952b6b: Pull complete
0ee66acd8266: Pull complete
303fe1bfb649: Pull complete
Digest: sha256:0222b795db95bf7412cede36ab46a266cfb31f632e64051aac9806dabf840a61
Status: Downloaded newer image for python:3.13-slim
 ---> 441ed6af7d7c
Step 2/4 : COPY script.py /app/script.py
 ---> 85acf39543e8
Step 3/4 : WORKDIR /app
 ---> Running in 2a70e35b3c89
Removing intermediate container 2a70e35b3c89
 ---> 23fa5750c848
Step 4/4 : CMD ["python", "script.py"]
 ---> Running in 0f04a729f055
Removing intermediate container 0f04a729f055
 ---> 77a8704ae87d
Successfully built 77a8704ae87d
Successfully tagged acr5beep294.azurecr.io/images/hello-write:latest
2025/10/26 23:04:41 Successfully executed container: build
2025/10/26 23:04:41 Executing step ID: push. Timeout(sec): 3600, Working directory: '', Network: ''
2025/10/26 23:04:41 Pushing image: acr5beep294.azurecr.io/images/hello-write:latest, attempt 1
The push refers to repository [acr5beep294.azurecr.io/images/hello-write]
1efe742425b2: Preparing
816b1f62d73f: Preparing
b9e88dc86e01: Preparing
618eb9b43e60: Preparing
d7c97cb6f1fe: Preparing
1efe742425b2: Pushed
816b1f62d73f: Pushed
618eb9b43e60: Pushed
b9e88dc86e01: Pushed
d7c97cb6f1fe: Pushed
latest: digest: sha256:6a5b1e78188680971487cb663372ae9a3c249d73c35f9f185ab70d0073bb30b5 size: 1366
2025/10/26 23:04:46 Successfully pushed image: acr5beep294.azurecr.io/images/hello-write:latest
2025/10/26 23:04:46 Step ID: build marked as successful (elapsed time in seconds: 6.127770)
2025/10/26 23:04:46 Populating digests for step ID: build...
2025/10/26 23:04:47 Successfully populated digests for step ID: build
2025/10/26 23:04:47 Step ID: push marked as successful (elapsed time in seconds: 5.693399)
2025/10/26 23:04:47 The following dependencies were found:
2025/10/26 23:04:47
- image:
    registry: acr5beep294.azurecr.io
    repository: images/hello-write
    tag: latest
    digest: sha256:6a5b1e78188680971487cb663372ae9a3c249d73c35f9f185ab70d0073bb30b5
  runtime-dependency:
    registry: registry.hub.docker.com
    repository: library/python
    tag: 3.13-slim
    digest: sha256:0222b795db95bf7412cede36ab46a266cfb31f632e64051aac9806dabf840a61
  git: {}


Run ID: ch1 was successful after 15s
```

Next, let's create our storage account and Azure Files share. Since the account name needs to be random I'll just name it something dumb with a couple of random digits at the end.

```sh
STORAGE_ACCOUNT="storageaccount$RANDOM"
SHARE="share$RANDOM"

az storage account create -g acr-rg -n "$STORAGE_ACCOUNT" -l eastus2 --sku Standard_LRS

az storage share create -n "$SHARE" --account-name "$STORAGE_ACCOUNT"
```

Then, grab the key for the storage account. We'll need it in a moment.

```sh
STORAGE_KEY=$(az storage account keys list -g acr-rg --account-name "$STORAGE_ACCOUNT" --query "[0].value" --output tsv)
```

Then, deploy the container, either with a YAML template or a long Azure CLI command.

Set the CPU and memory allocation if the image doesn't have a default. Set the restart policy or it'll default to "Always". Set the OS type or the Azure CLI will complain (it can't reliably auto-detect from private registries).

You'll have to specify credentials for the registry if you're using a private registry.

You'll also need to pass info on the file share you want to mount - mine will be mounted to `/data`, since that's what our crappy little Python script expects.

Here's how you'd run the container with one rather long `az container create` command:

```sh
az container create -g acr-rg -n hello-write \
    --image acr5beep294.azurecr.io/images/hello-write:latest \
    --os-type Linux \
    --cpu 0.1 \
    --memory 0.1 \
    --restart-policy Never \
    --registry-username "$ACR_USER" \
    --registry-password "$ACR_PASS" \
    --azure-file-volume-account-name "$STORAGE_ACCOUNT" \
    --azure-file-volume-account-key "$STORAGE_KEY" \
    --azure-file-volume-share-name "$SHARE" \
    --azure-file-volume-mount-path "/data"
```

YAML would look something like this:

```yaml
apiVersion: 2019-12-01
location: eastus2
name: hello-write
properties:
  containers:
  - name: hello-write
    properties:
      image: acr5beep294.azurecr.io/images/hello-write:latest
      resources:
        requests:
          cpu: 0.1
          memoryInGb: 0.1
      volumeMounts:
      - name: azurefile
        mountPath: /data
  imageRegistryCredentials:
  - server: acr5beep294.azurecr.io
    username: $ACR_USER
    password: $ACR_PASS
  osType: Linux
  restartPolicy: Never
  volumes:
  - name: azurefile
    azureFile:
      shareName: $SHARE
      storageAccountName: $STORAGE_ACCOUNT
      storageAccountKey: $STORAGE_KEY
tags: {}
type: Microsoft.ContainerInstance/containerGroups
```

And you'd run the YAML template with:

```sh
az container create -g acr-rg --file container.yaml
```

Anyway, let's run that bad boy. Wait a few moments, confirm the container is running (you should see `"state": "Running"` in the JSON output from your `container create`), then pop open the file share.

If I use Azure Storage Explorer to take a peek at the "hello.txt" file that we're writing to, I can see that my Python script worked:

{{< figure src ="images/5-ase.png" >}}

Remember to clean up your resource groups if you're following along.
