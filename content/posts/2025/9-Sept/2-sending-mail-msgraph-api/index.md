---
title: "Sending mail with Python via the Microsoft Graph API"
date: 2025-09-24T22:00:00-00:00
updated: 2025-09-25T23:00:00-00:00
draft: false
---

You can use the application Mail.Send permission and the `https://graph.microsoft.com/v1.0/users/user/sendMail` Graph API endpoint to send mail from a script or program via Microsoft 365. Pretty straightforward - the only slightly complicated part is restricting the scope of your Mail.Send permissions to a specific inbox or group of inboxes, which is an Exchange Online-ism.

Let's go through a quick Python example of sending an email. We'll get the app registration created, then scope down access to members of a security group. Then, we'll send a message with a basic example, before getting everything running in a Podman container.

## Create an app registration

Create an App Registration in the Entra admin center, or with PowerShell (`New-MgApplication`).

We don't need a redirect URI, and we can ignore the 'supported account types' since we'll be using client credentials, not delegated authentication.

{{< figure src="images/1-create-app-reg.png" >}}

Next, create a client secret or upload a certificate that the app will use to authenticate with Microsoft Graph.

{{< figure src="images/2-create-secret.png" >}}

Finally, grant the application the "Mail.Send" Graph permission.

{{< figure src="images/3-api-perms.png" >}}

At this stage, you could send email with the configured client secret and client ID. However, your app will be able to impersonate *anyone* with a mailbox in your tenant. We probably don't want this.

Note the Client (application) ID and let's keep going.

## Restrict the app to certain mailboxes

By default, the Mail.Send permissions grant full access to impersonate anyone in your tenant. To avoid this, you can use RBAC for applications in Exchange Online. You will need Exchange Online PowerShell for this bit.

Create a mail-enabled security group, either in the admin center or with PowerShell:

{{< figure src="images/4-security-group.png" >}}

{{< figure src="images/5-security-group.png" >}}

Now, we'll create an Application Access policy to restrict the app ID to a specific scope (our group).

```PowerShell
$AppID='0000000-0000-0000-0000-000000000000'
$GroupName='allowmailsendapiimpersonation@domain.example' 

Connect-ExchangeOnline

New-ApplicationAccessPolicy `
  -AppId $AppID `
  -PolicyScopeGroupId $GroupName `
  -AccessRight RestrictAccess
```

```txt
PS /Users/wporter> New-ApplicationAccessPolicy `
>>   -AppId $AppID `
>>   -PolicyScopeGroupId $GroupName `
>>   -AccessRight RestrictAccess

ScopeName        : Allow Mail.Send API impersonation
ScopeIdentity    : Allow Mail.Send API impersonation20250922222222
Identity         : 0
AppId            : 0
ScopeIdentityRaw : 0
Description      : AppId: 0;
                   ScopeIdentitySid: 0;
                   ScopeIdentityOid: 0;
                   AccessRight: RestrictAccess; ShardType: All
AccessRight      : RestrictAccess
ShardType        : All
IsValid          : True
ObjectState      : Unchanged
```

## Send a message with Python

Finally, we can get on to sending an email.

We'll be using the Microsoft Authentication Library (MSAL) and Requests Python modules for this.

```python
import requests
import msal
```

Define your:

Client ID
Client secret
Tenant ID
Authority (`f'https://login.microsoftonline.com/{tenant_id}'`)
Scope (`['https://graph.microsoft.com/.default']`)

Create a ConfidentialClientApplication object that you can use to authenticate with the Graph API using the MSAL auth handler:

```python
app = msal.ConfidentialClientApplication(
    client_id,
    authority=authority,
    client_credential=client_secret
)
```

Acquire an OAUTH token for the application:

```python
result = app.acquire_token_for_client(scopes=scope)
access_token = result['access_token']
```

Assemble a message:

```python
headers = {
    'Authorization': f'Bearer {access_token}',
    'Content-Type': 'application/json'
}
mail_data = {
    "message": {
        "subject": "Message from Python",
        "body": {
            "contentType": "Text",
            "content": "Why hello there"
        },
        "toRecipients": [
            {
                "emailAddress": {
                    "address": "liam@foo.co.au"
                }
            }
        ]
    }
}
```

All together, this little script is:

```python
import requests
import msal
import os

client_id = os.getenv('CLIENT_ID')
client_secret = os.getenv('CLIENT_SECRET')
tenant_id = os.getenv('TENANT_ID')

authority = f'https://login.microsoftonline.com/{tenant_id}'
scope = ['https://graph.microsoft.com/.default']

app = msal.ConfidentialClientApplication(
    client_id,
    authority=authority,
    client_credential=client_secret
)

result = app.acquire_token_for_client(scopes=scope)

if 'access_token' in result:
    access_token = result['access_token']
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json'
    }
    email_data = {
        "message": {
            "subject": "",
            "body": {
                "contentType": "Text",
                "content": ""
            },
            "toRecipients": [
                {
                    "emailAddress": {
                        "address": "recipient@domain.com"
                    }
                }
            ],
            "CCRecipients": [
                {
                    "emailAddress": {
                        "address": "ccrecipient@domain.com"
                    }
                }
            ]
        }
    }

    response = requests.post('https://graph.microsoft.com/v1.0/users/sender@domain.com/sendMail', headers=headers, json=email_data)

    if response.status_code == 202:
        print("Email sent successfully.")
    else:
        print(f"Failed to send email: {response.status_code} - {response.text}")
    
else:
    print("Failed to acquire token.")
    print(result.get("error"))
    print(result.get("error_description"))
    print(result.get("correlation_id"))
    print(result)
```

Make a POST request to the `graph/v1.0/users/user@domain.example/sendMail` API endpoint (as below) to send your message. It should quickly arrive in the user's inbox if they're a local EXO account.

```python
response = requests.post('https://graph.microsoft.com/v1.0/users/user@domain.example/sendMail', headers=headers, json=mail_data)
```

Attempting to send a message as a user the app is not permitted to impersonate (per the EXO app access policy) will return a 403 error:

```json
Failed to send email: 403 - {"error":{"code":"ErrorAccessDenied","message":"Access to OData is disabled: [RAOP] : Blocked by tenant configured AppOnly AccessPolicy settings."}}
```

## Put it in a Podman container

I suppose we can go over Podman secrets while we're at it. Why the heck not?

I'll be working from a fresh AlmaLinux 10 install. I've copied my files (script and requirements.txt) over, and installed the `podman` package. My credentials have been set to `os.getenv()` calls in the script (we'll deal with getting them to the script later):

```python
client_id = os.getenv('CLIENT_ID')
client_secret = os.getenv('CLIENT_SECRET')
tenant_id = os.getenv('TENANT_ID')
```

I'll write a quick Containerfile:

```Dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY . /app/

RUN pip install --no-cache-dir -r requirements.txt

CMD ["python", "sendmail.py"]
```

Then I'll build the container with `podman image build`:

```sh
podman image build . -t py-mail-secrets
```

I'll create my secrets (which will be set to env variables in the container) with `read -s` (to take a line of input without writing it to the console) and `podman secret create`. Podman secrets in this case (rootless, file driver) are stored in an access protected (ugo 600) file at `~/.local/share/containers/storage/secrets/filedriver/secretsdata.json` (`$GRAPHROOT/secrets/filedriver`).

```sh
read -s tenant_id
read -s client_id
read -s client_secret

echo "$tenant_id" | podman secret create tenant_id -
echo "$client_id" | podman secret create client_id -
echo "$client_secret" | podman secret create client_secret -
```

I can confirm that these were created with `podman secret ls`:

```sh
$ podman secret ls
ID                         NAME           DRIVER      CREATED         UPDATED
0000570ba4c48aae000000000  tenant_id      file        42 minutes ago  42 minutes ago
0000a88888488888888000000  client_id      file        42 minutes ago  42 minutes ago
00002a2a2a2a2a2a2d0000000  client_secret  file        42 minutes ago  42 minutes ago
```

Then, I can run the container with my secrets and confirm it works:

```sh
$ podman run \
    --secret client_id,type=env,target=CLIENT_ID \
    --secret client_secret,type=env,target=CLIENT_SECRET \
    --secret tenant_id,type=env,target=TENANT_ID \
    localhost/py-mail-secrets

Email sent!
```

If you check your recipient mailbox, you should find your test email. All there is to it!

I'll deal with certificate auth another day...
