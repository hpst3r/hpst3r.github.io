---
title: "Azure Application Gateway (HTTP load balancer) lab"
date: 2025-09-21T22:00:00-00:00
draft: false
---

[Relevant MS Learn training module here.](https://learn.microsoft.com/en-us/training/modules/intro-to-azure-application-gateway/)

Just dumping some notes here so it doesn't look totally dead.. I've been busy doing work & study things that are not particularly bloggable this month.

## Review of docs

Azure App Gateway is a regional, web-specific load balancer that processes traffic to web apps hosted by a pool of web servers. It can load balance HTTP traffic, offload SSL from your web servers, and inspect traffic (serving as a WAF - e.g., filtering SQL injection or XSS attempts).

AAG supports HTTP, HTTPS, HTTP/2\* and WebSocket traffic. It includes a WAF, supports end-to-end request encryption, has autoscaling, and supports connection draining (for graceful removal of pool members during planned maintenance).

> HTTP/2 protocol support is available to clients that connect to application gateway listeners only. The communication to backend server pools is over HTTP/1.1.

Load balancing is performed by the App Gateway using a L7 round-robin mechanism (requests are load balanced based on routing parameters, e.g., host names and paths, used by AAG rules). You can configure session stickiness, as with other Azure load balancers, if you need all requests from one client to reach one server.

The Web Application Firewall (WAF) component of the Azure Application Gateway is optional. To take full advantage of it, you'll need to pay up for the "WAF v2" Azure App Gateway SKU.

The WAF implements the OWASP Core Rule Set (CRS) to mitigate common webapp threats like SQL injection, XSS, command injection, request smuggling, response splitting, remote file inclusion, bots, crawlers, scanners, and protocol violations.

When using the WAF, you can specify which elements of a request should be inspected or limit the size of messages to prevent your servers from being overwhelmed. You can also direct firewall logs from the WAF to your destination of choice for analysis or alerting, e.g., a Log Analytics workspace.

### Gateway front-end

The App Gateway's front-end is a zone-redundant ingress IP. This may be a public or private IP; if public, it'll always be a standard SKU static IP address. IPv4 and IPv6 are both supported.

The Basic App Gateway does not support private front-ends (use as a Private App Gateway) - you will need to use the Standard or WAF App Gateways for this.

### Load balancer back-end pools

Members of an AAG back-end pool can be selected from:

- IP address or FQDN (e.g., on-premise resources, or an Azure VM with multiple IPs)
- Azure virtual machine object (be sure it only has one IP)
- Azure virtual machine scale set (VMSS)
- An Azure App Services PaaS instance

Each pool has its own managed load balancer.

### Traffic routing

You can use path-based routing or multi-site routing to send traffic to specific servers based on the URI's components. In addition, you can redirect traffic, rewrite headers, or create custom error pages at the Application Gateway.

#### Path-based routing

Path-based routing sends requests with different URL paths to different pools of servers. For example, you can configure requests to /video/* to reach servers optimized for video streaming, and /photos/* to servers configured for image retrieval.

#### Multiple-site routing

Multiple-site routing is used to redirect requests to different domains to different pools. For example, you can configure "fabrikam.com" and "contoso.com", and send "fabrikam" requests to the "fabrikam" pool. Multi-site routing is useful primarily for multitenant applications. (e.g., customer1.domain.co.au, customer2.domain.co.au are separate servers).

### TLS termination

If you want to terminate SSL on the AAG, you can do so by simply feeding the Azure App Gateway a cert (Let's Encrypt/ACME are not supported natively [but can be made to work](https://intelequia.com/es/blog/post/automating-azure-application-gateway-ssl-certificate-renewals-with-let-s-encrypt-and-azure-automation)) and configuring it to forward HTTP traffic along to the back-end pool(s).

If you need end-to-end encryption, you can have your Application Gateway decrypt, inspect, then re-encrypt the request with a different certificate.

### Health probes

App Gateways make HTTP requests and look for anything between 200 and 399 in response. **If you don't configure a health probe, the App Gateway creates a default probe** (30 second wait to mark nodes unhealthy).

### WebSocket and HTTP/2 traffic

Notably, App Gateways support WebSocket and HTTP/2 - these are full duplex protocols that enable more interactive, bidirectional, and low-overhead connections that utilize resources more efficiently for modern webapps (can reuse TCP connections, etc).

### SKUs and limitations

The three Azure Application Gateway SKUs are:

- Basic
- Standard v2
- WAF v2

**Basic AAGs** are highly available, support HTTP/2 and HTTPS, WebSocket, and have **core load balancing features** like URL-based, host-based, or multi-site routing and cookie-based affinity. The fixed cost of a Basic AAG is approximately 1/6 of a Standard v2 AAG.

I'll quickly go over the Basic SKU's limitations, since they'll tell you when you need to use a more expensive SKU instead. Source for this info is the [Basic SKU introduction](https://techcommunity.microsoft.com/blog/fasttrackforazureblog/save-costs-with-basic-sku-application-gateway-for-more-features-and-less-fixed-c/4099223) blog post.

- **Basic SKUs offer worse performance** (vs Standard v2)
  - Approx. 200 CPS vs 62500 max
  - Max 5 listeners (vs 100)
  - Max 10 CPS per compute unit (vs 50)
- A maximum of **5 back-end pools** (vs 100)
  - Max. 5 servers per pool vs 1200
- **Support fewer routing rules** (5 max. vs 400)
- **Do not support autoscaling**
- **Do not support TCP/TLS proxies**
- **Do not support URL rewrite**
- **No private app gateway**/private link support (must be a public app gateway, can't assign a private IP to the frontend)
- Do not support mutual TLS (mTLS) authentication

**Standard v2 AAGs** are highly available (with autoscaling), and boast a larger feature set than (e.g., mTLS, private app gateway support, URL rewrite support) and better performance than the Basic SKU, but cost more.

**WAF v2 AAGs** are the Standard SKU with the Web Application Firewall (WAF) enabled. You can customize which parts or version of the OWASP core rule set are active on your WAF (3.2, 3.1, 3.0, and 2.2.9 are supported; default is 3.1).

### Networking

AAGs can only exist in a vnet subnet with other AAGs. Subnets must be either empty, or only populated with other AAGs if you would like to put another App Gateway there.

You can configure a private link from a Standard or WAF Application Gateway to another virtual net.

### AAG vs other Azure load balancers

[See the Azure Architecture Center article on the available load balancing offerings.](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview)

Azure App Gateway is a layer 7 web traffic load balancer/web application firewall. It is intended for use as a load balancer/gateway/WAF sitting between the Internet and pools of private servers (in one region) hosted somewhere with access to Azure. It is a **single region solution** and not intended for global use.

**Azure App Gateway is what you would use if you're hosting a web app in one region that needs a WAF and SSL offload**. It's a high performance web traffic load balancer. In addition, if you need complex path-based routing, App Gateway is probably the best choice.

Azure App Gateway is not capable of, for example, balancing UDP traffic, or database traffic. You would need to use Azure Load Balancer or other middleware (e.g., dnsdist) for this.

App Gateway isn't intended to distribute global traffic to local regions. You might combine it with Azure Front Door or Traffic Manager to balance traffic within a region, but you would use Azure Front Door or Traffic Manager to direct traffic to the App Gateway in this scenario.

The other load balancers are:

**Azure API Management** is intended to secure and load balance HTTP/S APIs ONLY. This is one feature of API Management, which an API gateway that happens to provide load balancing as a feature, not just a load balancer.

**Azure Front Door is a CDN** ("App Delivery Network") solution that can serve as a L7 load balancer. It's intended for **global use** and most notably **offers caching** to improve availability. It does not include WAF functionality. It does offload SSL. It offers faster fail-over than Azure Traffic Manager, since it doesn't depend on DNS.

**Azure Load Balancer is a plain, but high-performance Layer 4 load balancer** supporting TCP and UDP traffic. Since it is a Layer 4 load balancer, it doesn't offload SSL or inspect traffic, and health checks are more rudimentary (can a session be established/do I get a HTTP 200?)

**Azure Traffic Manager is a DNS-based global traffic load balancer** intended to help you distribute traffic to services spread out over global Azure regions. It load balances only at the domain level.

In summary:

- **API Management** is **regional or global**, and serves **HTTP** APIs only
- **Application Gateway** is a **regional** web traffic load balancer, and serves **HTTP** only
- **Azure Front Door** is a **global CDN** for web applications, and serves **HTTP** only
- **Azure Load Balancer** is a **L4 load balancer** that can be either **regional or global**. It serves TCP or UDP.
- **Azure Traffic Manager** is a **global DNS-based traffic management** solution that directs traffic to specific services (e.g., other load balancers).

## Lab - Working with an AAG to load balance, terminate SSL for, and firewall web traffic

We'll be deploying two web servers and an Azure Application Gateway in a vNet. The deployment will look a little something like this:

{{< figure src="images/0-app-gateway-lab.drawio.png" >}}

I'll first create a resource group, "aag-lab-rg", to contain my resources.

{{< figure src="images/1-create-rg.png" >}}

Then, I'll create a virtual network, "aag-lab-vnet", with the IP range 10.128.0.0/23 and two subnets, "webserver-sub" and "gateway-sub".

{{< figure src="images/2-create-vnet.png" >}}

{{< figure src="images/3-create-vnet.png" >}}

And I'll create two virtual machines, "nginx0" and "nginx1", running AlmaLinux 9. I'll leave them with the image default storage (30gb of Premium LRS), add my SSH public key, and, for simplicity's sake, give them both a public IP.

{{< figure src="images/4-create-vm.png" >}}

I'll create `nginx1` with an ARM template:

{{< figure src="images/5-create-vm-2.png" >}}

{{< figure src="images/6-vm-arm-deployment.png" >}}

My "nginx0" VM was assigned IP 20.36.130.36, so I'll SSH to it, install NGINX, and enable the service:

```sh
[wporter@nginx0 ~]$ sudo dnf install -y nginx
[wporter@nginx0 ~]$ sudo systemctl enable --now nginx
```

Then, I'll add a NSG rule to permit HTTP traffic inbound, and confirm that I'm able to get to the first host.

{{< figure src="images/7-nginx-test-page.png" >}}

I'll do the same with the second host (20.97.208.236), including creating a NSG rule to allow traffic inbound. In addition, I'll modify the test page so we can tell which server we get.

```sh
[wporter@nginx1 ~]$ sudo dnf install -y nginx
[wporter@nginx1 ~]$ sudo vi /usr/share/nginx/html/index.html
[wporter@nginx1 ~]$ sudo systemctl enable nginx --now
```

{{< figure src="images/8-nginx1-test-page.png" >}}

Now that our web servers are up and running, let's configure our ASG.

I'll navigate to the "load balancing and content delivery" service area, and select "application gateways" under "load balancing". Then, I'll click "Create", and make an application gateway.

{{< figure src="images/9-app-gateways.png" >}}

I'll pick my lab resource group, the Basic SKU (doesn't support autoscaling, so the option will disappear) and an IPv4 address (even the Basic SKU uses a static v4 address). I'll leave HTTP/2 enabled (the default).

Then, I'll select my "aag-lab-vnet" virtual network, and the "webserver-sub" subnet.

Note how the "webserver-sub" subnet isn't eligible. As the error notes, you can't put an Application Gateway in a subnet with anything other than Application Gateways.

{{< figure src="images/10-create-app-gateway.png" >}}

I'll change my subnet selection to the gateway subnet in our "aag-lab-vnet" virtual network, then click Next.

{{< figure src="images/11-create-app-gateway-basics.png" >}}

The "frontend" for an application gateway is its IP address. Since we're using a basic SKU, private app gateways aren't supported - we can only choose a public IP address.

We're able to choose an existing IP address, but both of mine are associated with our web servers:

{{< figure src="images/12-create-app-gateway-frontends.png" >}}

So, I'll create a new one, then click Next:

{{< figure src="images/13-create-app-gateway-frontends-2.png" >}}

I'll create my back-end pool of web servers, selecting "nginx0" and "nginx1". I'll click Add, to add the pool to the app gateway's configuration, then, since we've only got the one pool, I'll click Next.

{{< figure src="images/14-create-app-gateway-backend-pool.png" >}}

I ran into some trouble with my pool here - specifying the VMs (with an associated public IP and a private IP) as a "VM" in the pool caused the application gateway deployment to fail. I had to recreate the application gateway (more specifically, the backend pool) with the VMs' private IPs (10.128.0.4 and 10.128.0.5) instead to get it working.

Deleting the VMs' public IPs and then adding the VMs to the pool *did* succeed, so I suspect Azure got a little confused as to what address to use as a pool member.

{{< figure src="images/15-create-app-gateway-backend-pool.png" >}}

On the Configuration page, we'll need to add routing rules (telling the app gateway when to send traffic and what traffic to send to our back-end pool). Click "Add a routing rule".

{{< figure src="images/16-create-app-gateway-routing-rule.png" >}}

To keep things relatively simple, we'll just add a rule for HTTP traffic on port 80. Since we're not using multiple-site routing, leave "Listener type" set to Basic.

I'll configure the listener as follows:

- Name: "http-listener"
- Frontend IP: "Public IPv4"
- Port: 80
- Listener type: Basic
- Custom error pages: Unset

Then, I'll switch over to the 'backend targets'.

{{< figure src="images/17-create-app-gateway-routing-rule.png" >}}

I'll configure the 'backend targets' to be our recently configured pool of web servers. We don't have any existing backend settings, so I'll click 'add new' to create a new set, and configure the backend protocol as HTTP on port 80, with connection draining enabled.

The connection draining timeout can be set to any value from 1 to 3600. We'll leave it at the default, 60.

I'll also leave the request timeout at its default of 20 seconds. This field supports a value between 1 and 86400 (seconds).

We won't override the backend path, hostname, or create custom probes. Click Add when done.

{{< figure src="images/18-create-app-gateway-backend-settings.png" >}}

Back in the routing rule config, we can now click Add to add the routing rule to our app gateway. Since we don't have multiple pools, we can't use path-based routing (PBR).

{{< figure src="images/19-create-app-gateway-routing-rule.png" >}}

Our app gateway now has a minimal configuration that should let us test it out. Click Next twice to skip by 'tags'.

{{< figure src="images/20-created-app-gateway-routing-rules.png" >}}

Review the configuration, then click Create to deploy the gateway.

Note that, by default, the app gateway will exist in three availability zones in my region - this is a zone-redundant service by default.

{{< figure src="images/21-create-app-gateway-validate.png" >}}

Wait for the deployment to create your new public IP and app gateway. Then, let's test it out. I'll attempt to reach its newly created public IP with HTTP: http://132.196.153.82.

{{< figure src="images/22-app-gateway-test-page-1.png" >}}

One request got me to `nginx1`... let's try again!

{{< figure src="images/23-app-gateway-test-page-0.png" >}}

That's `nginx0`!

Now, let's replace our listener so we can do SSL. Navigate to Listeners, under the app gateway, and click 'Add listener':

{{< figure src="images/24-app-gateway-new-listener.png" >}}

I'll quickly throw together a self-signed cert, since the listeners don't have native Let's Encrypt support. What I'm doing here is:

- Generating a private key and self-signed certificate with `openssl req -newkey`
- Converting the private key and certificate to .pfx format, which is what Azure wants (.pfx is a combination of the private key and certificate)

```sh
% openssl req -newkey rsa:2048 -nodes -keyout server.key -x509 -out server.crt

% openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt
```

Once I've added the cert to the portal, I'll click Add to create the new HTTPS listener.

{{< figure src="images/25-app-gateway-ssl-listener.png" >}}

Then, move on down to Rules, and create a new one for our new listener.

I'll name it "https-to-http-rule", with priority 2, using listener "https-listener".

For backend targets, I'll specify our existing "nginx-pool", and "http-settings". Then, I'll click Add to create the routing rule. Once done, I'll try to connect to the Application Gateway via HTTPS (https://132.196.153.82):

{{< figure src="images/26-app-gateway-ssl-working.png" >}}

I've reached nginx1 in this case. I can tell that my cert is being used, and the page is being served over HTTPS. The internal connection (from the app gateway to the web servers) is HTTP only, but SSL is terminating at the app gateway (protecting my poor little B1s VMs).

Now, let's try to enable the web application firewall (WAF) functionality available in the WAF v2 App Gateway SKU.

I'll navigate to the app gateway > Configuration, and change Tier from "Basic" to "WAF v2". Autoscale will become an option, but I'll leave it set to manually scale to 2 instances of the AAG.

{{< figure src="images/27-app-gateway-sku-change.png" >}}

The keen-eyed will notice that this immediately enables the WAF, private link, and SSL settings options in the left-hand navigation pane.

**Private link** would allow us to directly attach the App Gateway's frontend to another Azure virtual network.

The **SSL settings** would enable you to configure client authentication (by uploading a trusted root certificate or chain of certificates), or listener-specific SSL policies (allowed cipher suites).

The WAF settings allow you to choose which tier of WAF you're using (Standard V2, which has some features, or the fully-unlocked WAF V2 tier), enable or disable the web app firewall, and configure rules:

{{< figure src="images/28-app-gateway-waf-toggle.png" >}}

I'll enable the WAF, which gives us a few more options:

{{< figure src="images/29-app-gateway-waf-enabled-options.png" >}}

You can choose the WAF mode (detect or prevent), set exclusions, and set maximums on request size and file upload limit.

Exclusions are a list of headers, cookies, or attributes in the request that should not be screened by the WAF:

{{< figure src="images/30-app-gateway-waf-exclusions.png" >}}

Under Rules, you can specify a rule set (OWASP version), and, if you toggle advanced rule configuration, you can disable specific OWASP rules:

{{< figure src="images/31-app-gateway-waf-rules.png" >}}

I'll set our WAF to "Prevention" mode, then send a SQL injection attempt (just hit the app gateway with a request like `https://132.196.153.82/?id=1%20OR%201%3D1`) to test its function.

We'll get a 403 back if the WAF blocks the request:

{{< figure src="images/32-app-gateway-waf-blocked-request-403.png" >}}

```txt
wporter@MacBookAir ~ % curl -ik "http://132.196.153.82/?id=1%20OR%201%3D1"
HTTP/1.1 403 Forbidden
Server: Microsoft-Azure-Application-Gateway/v2
Date: Sun, 07 Sep 2025 23:57:40 GMT
Content-Type: text/html
Content-Length: 179
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>Microsoft-Azure-Application-Gateway/v2</center>
</body>
</html>
```

To view logged events, you'll need to configure diagnostics settings for the application gateway. For example, I've created the `app-gateway-log-workspace` Azure Log Analytics Workspace in my resource group, and I want to stream my firewall logs there so I can query them.

I'll first navigate to "diagnostic settings" under my WAF App Gateway:

{{< figure src="images/33-app-gateway-diagnostic-settings.png" >}}

Then, I'll create a diagnostic setting to point the application gateway firewall log to my Log Analytics workspace, and click Save:

{{< figure src="images/34-app-gateway-waf-to-log-analytics.png" >}}

If I now navigate to Logs and select the firewall table or query the AzureDiagnostics category ApplicationGatewayFirewallLog, I can see the events caused by my SQL injection attempts:

{{< figure src="images/35-app-gateway-waf-logs-in-table.png" >}}

Good stuff!

I'm going to leave this here for now.. on to the next one!
