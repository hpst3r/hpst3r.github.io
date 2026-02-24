---
title: "Regenerating HTTP certs in an Elasticsearch cluster"
date: 2026-02-23T20:35:00-00:00
draft: false
---

[Docs](https://www.elastic.co/docs/deploy-manage/security/set-up-basic-security). This requires the "minimum security setup" to be complete (Elasticsearch security must be running for SSL to be in use). The automatic initialization will do this for you.

By default, initializing an Elasticsearch cluster will automatically generate some certificates for the transport and HTTP components of the Elastic application:

```txt
[root@es9-1 elasticsearch]# ls -l certs
total 24
-rw-rw----. 1 root elasticsearch  1939 Dec 12 16:49 http_ca.crt
-rw-rw----. 1 root elasticsearch 10077 Dec 12 16:49 http.p12
-rw-rw----. 1 root elasticsearch  5838 Dec 12 16:49 transport.p12
[root@es9-1 elasticsearch]# pwd
/etc/elasticsearch
```

After changing a node's IP (use DNS, gosh darn it!) or hostname, you may need to regenerate its HTTP certificates, or things will permanently cease to function (Elastic uses mTLS for authentication for the cluster's transport layer and HTTP REST API - no REST API, Elastic is very unhappy).

Bear in mind that, depending on your certificate `verification_mode` (e.g., `full`), you may also need to regenerate transport certs, which is a separate process.

To regenerate HTTP certificates, we can use the `elasticsearch-certutil http` utility. However, this will first require us to extract the automatically generated CA keypair's private key, which takes a few more steps.

This private key is stored in the `/usr/share/elasticsearch/bin/elasticsearch-keystore` (`$ES_HOME/bin/elasticsearch-keystore`) file.

To retrieve the password for the `http_ca` private key, inspect the `xpack.security.http.ssl.keystore.secure_password` object's value in the `elasticsearch-keystore` Java keystore:

```txt
[root@es9-1 ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password
foobar
```

Then, inspect the PKCS12 cert bundle for the CA (`http.p12`) in `$ELASTIC_HOME/certs` with OpenSSL. Look for a private key with the friendly name "http_ca".

```txt
[root@es9-1 ~]# openssl pkcs12 -info -in /etc/elasticsearch/certs/http.p12 -nodes -passin pass:foobar
MAC: sha256, Iteration 10000
MAC length: 32, salt length: 20
PKCS7 Data
Shrouded Keybag: PBES2, PBKDF2, AES-256-CBC, Iteration 10000, PRF hmacWithSHA256
Bag Attributes
    friendlyName: http_ca
    localKeyID: 54 69 6D 65 20 31 37 36 35 35 37 36 31 34 36 38 32 31
Key Attributes: <No Attributes>
-----BEGIN PRIVATE KEY-----
foobarbaz
```

```sh
HTTP_P12_PASSWORD=$(/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password)

openssl pkcs12 -in /etc/elasticsearch/certs/http.p12 \
  -passin pass:$HTTP_P12_PASSWORD \
  -nokeys -out /tmp/all-certs.pem

openssl pkcs12 -in /etc/elasticsearch/certs/http.p12 \
  -passin pass:$HTTP_P12_PASSWORD \
  -nocerts -nodes -out /tmp/all-keys.pem
```

```txt
[root@es9-1 ~]# cat /tmp/all-keys.pem
Bag Attributes
    friendlyName: http_ca
    localKeyID: 54 69 6D 65 20 31 37 36 35 35 37 36 31 34 36 38 32 31
Key Attributes: <No Attributes>
-----BEGIN PRIVATE KEY-----
```

Extract just the CA's private key. We'll need it in a file to pass to the cert utility.

```sh
# this won't make a clean PEM but is fine for openssl
openssl pkcs12 -in /etc/elasticsearch/certs/http.p12 \
  -passin pass:$HTTP_P12_PASSWORD \
  -nocerts -nodes \
  | awk '/friendlyName: http_ca/,/-----END PRIVATE KEY-----/' \
  > /tmp/http_ca.key
```

For the sake of Future Me, we will drop the CA materials in a directory and keep them around so we don't have to do the whole keystore song and dance next time:

```sh
mkdir -p /root/elasticsearch-ca
chmod 700 /root/elasticsearch-ca

cp /tmp/http_ca.key /root/elasticsearch-ca/
cp /etc/elasticsearch/certs/http_ca.crt /root/elasticsearch-ca/

# set restrictive permissions on private key
chmod 600 /root/elasticsearch-ca/http_ca.key
chmod 644 /root/elasticsearch-ca/http_ca.crt
chown root:root /root/elasticsearch-ca/*

# clean up sensitive temp file
shred -u /tmp/http_ca.key

# remove temp certs and keys
rm -f /tmp/all-keys.pem /tmp/all-certs.pem
```

This is what our directory will look like:

```txt
[root@es9-1 ~]# ls -l /root/elasticsearch-ca/
total 8
-rw-r--r--. 1 root root 1939 Dec 22 22:03 http_ca.crt
-rw-------. 1 root root 3401 Dec 22 22:03 http_ca.key
```

Now, finally, we can run the `elasticsearch-certutil` tool and regenerate some certs.

```txt
[root@es9-1 ~]# /usr/share/elasticsearch/bin/elasticsearch-certutil http

## Elasticsearch HTTP Certificate Utility

The 'http' command guides you through the process of generating certificates
for use on the HTTP (Rest) interface for Elasticsearch.

Generate a CSR? [y/N]n

Use an existing CA? [y/N]y

## What is the path to your CA?

Please enter the full pathname to the Certificate Authority that you wish to
use for signing your new http certificate. This can be in PKCS#12 (.p12), JKS
(.jks) or PEM (.crt, .key, .pem) format.
CA Path: /root/elasticsearch-ca/http_ca.crt

## What is the path to your CA key?

/root/elasticsearch-ca/http_ca.crt appears to be a PEM formatted certificate file.
In order to use it for signing we also need access to the private key
that corresponds to that certificate.

CA Key: /root/elasticsearch-ca/http_ca.key

## How long should your certificates be valid?

Every certificate has an expiry date. When the expiry date is reached clients
will stop trusting your certificate and TLS connections will fail.

Best practice suggests that you should either:
(a) set this to a short duration (90 - 120 days) and have automatic processes
to generate a new certificate before the old one expires, or
(b) set it to a longer duration (3 - 5 years) and then perform a manual update
a few months before it expires.

You may enter the validity period in years (e.g. 3Y), months (e.g. 18M), or days (e.g. 90D)

For how long should your certificate be valid? [5y]

## Do you wish to generate one certificate per node?

If you have multiple nodes in your cluster, then you may choose to generate a
separate certificate for each of these nodes. Each certificate will have its
own private key, and will be issued for a specific hostname or IP address.

Alternatively, you may wish to generate a single certificate that is valid
across all the hostnames or addresses in your cluster.

If all of your nodes will be accessed through a single domain
(e.g. node01.es.example.com, node02.es.example.com, etc) then you may find it
simpler to generate one certificate with a wildcard hostname (*.es.example.com)
and use that across all of your nodes.

However, if you do not have a common domain name, and you expect to add
additional nodes to your cluster in the future, then you should generate a
certificate per node so that you can more easily generate new certificates when
you provision new nodes.

Generate a certificate per node? [y/N]n

## Which hostnames will be used to connect to your nodes?

These hostnames will be added as "DNS" names in the "Subject Alternative Name"
(SAN) field in your certificate.

You should list every hostname and variant that people will use to connect to
your cluster over http.
Do not list IP addresses here, you will be asked to enter them later.

If you wish to use a wildcard certificate (for example *.es.example.com) you
can enter that here.

Enter all the hostnames that you need, one per line.
When you are done, press <ENTER> once more to move on to the next step.

*.lab.wporter.org
*.es.lab.wporter.org

You entered the following hostnames.

 - *.lab.wporter.org
 - *.es.lab.wporter.org

Is this correct [Y/n]y

## Which IP addresses will be used to connect to your nodes?

If your clients will ever connect to your nodes by numeric IP address, then you
can list these as valid IP "Subject Alternative Name" (SAN) fields in your
certificate.

If you do not have fixed IP addresses, or not wish to support direct IP access
to your cluster then you can just press <ENTER> to skip this step.

Enter all the IP addresses that you need, one per line.
When you are done, press <ENTER> once more to move on to the next step.

172.27.30.91
172.27.30.92
172.27.30.93

You entered the following IP addresses.

 - 172.27.30.91
 - 172.27.30.92
 - 172.27.30.93

Is this correct [Y/n]y

## Other certificate options

The generated certificate will have the following additional configuration
values. These values have been selected based on a combination of the
information you have provided above and secure defaults. You should not need to
change these values unless you have specific requirements.

Key Name: es.lab.wporter.org
Subject DN: CN=es, DC=lab, DC=wporter, DC=org
Key Size: 2048
Key Usage: digitalSignature,keyEncipherment

Do you wish to change any of these options? [y/N]n

## What password do you want for your private key(s)?

Your private key(s) will be stored in a PKCS#12 keystore file named "http.p12".
This type of keystore is always password protected, but it is possible to use a
blank password.

If you wish to use a blank password, simply press <enter> at the prompt below.
Provide a password for the "http.p12" file:  [<ENTER> for none]

## Where should we save the generated files?

A number of files will be generated including your private key(s),
public certificate(s), and sample configuration options for Elastic Stack products.

These files will be included in a single zip archive.

What filename should be used for the output zip file? [/usr/share/elasticsearch/elasticsearch-ssl-http.zip]

Zip file written to /usr/share/elasticsearch/elasticsearch-ssl-http.zip
```

Unzip the generated archive to a temporary directory:

```txt
[root@es9-1 ~]# unzip /usr/share/elasticsearch/elasticsearch-ssl-http.zip -d /tmp/new-http-certs
Archive:  /usr/share/elasticsearch/elasticsearch-ssl-http.zip
   creating: /tmp/new-http-certs/elasticsearch/
  inflating: /tmp/new-http-certs/elasticsearch/README.txt
  inflating: /tmp/new-http-certs/elasticsearch/http.p12
  inflating: /tmp/new-http-certs/elasticsearch/sample-elasticsearch.yml
   creating: /tmp/new-http-certs/kibana/
  inflating: /tmp/new-http-certs/kibana/README.txt
  inflating: /tmp/new-http-certs/kibana/elasticsearch-ca.pem
  inflating: /tmp/new-http-certs/kibana/sample-kibana.yml
[root@es9-1 ~]# cat /tmp/new-http-certs/elasticsearch/README.txt
There are three files in this directory:

1. This README file
2. http.p12
3. sample-elasticsearch.yml

## http.p12

The "http.p12" file is a PKCS#12 format keystore.
It contains a copy of your certificate and the associated private key.
You should keep this file secure, and should not provide it to anyone else.

You will need to copy this file to your elasticsearch configuration directory.

Your keystore has a blank password.
It is important that you protect this file - if someone else gains access to your private key they can impersonate your Elasticsearch node.

## sample-elasticsearch.yml

This is a sample configuration for Elasticsearch to enable SSL on the http interface.
You can use this sample to update the "elasticsearch.yml" configuration file in your config directory.
The location of this directory can vary depending on how you installed Elasticsearch, but based on your system it appears that your config
directory is /etc/elasticsearch

This sample configuration assumes that you have copied your http.p12 file directly into the config directory without renaming it.
[root@es9-1 ~]# ls -l /tmp/new-http-certs/elasticsearch
total 16
-rw-r--r--. 1 root root 4564 Dec 22 22:07 http.p12
-rw-r--r--. 1 root root 1091 Dec 22 22:07 README.txt
-rw-r--r--. 1 root root  657 Dec 22 22:07 sample-elasticsearch.yml
```

Then, I'll just overwrite the existing certs. As the http.p12 keystore contains private keys, we'll be sure to `chown` and `chmod` it so only the Elasticsearch user/group can access it.

```txt
[root@es9-1 ~]# cp /tmp/new-http-certs/elasticsearch/http.p12 /etc/elasticsearch/certs/
cp: overwrite '/etc/elasticsearch/certs/http.p12'? y
[root@es9-1 ~]# chown elasticsearch:elasticsearch /etc/elasticsearch/certs/http.p12
chmod 640 /etc/elasticsearch/certs/http.p12
```

We will then need to import the CA keypair into the new `http.p12` certificate package (so it has the full chain - this is required.)

Create a PKCS12 file containing your CA cert and key:

```bash
openssl pkcs12 -export \
  -in /root/elasticsearch-ca/http_ca.crt \
  -inkey /root/elasticsearch-ca/http_ca.key \
  -out /tmp/ca.p12 \
  -name "http_ca" \
  -passout pass:
```

We'll make a quick copy of the new keystore before the import:

```sh
cp -p /etc/elasticsearch/certs/http.p12 /etc/elasticsearch/certs/http.p12.backup-before-ca-import
```

Import the CA keystore to the HTTP keystore (http.p12). The HTTP keystore must have the full certificate chain.

```sh
SRC_PASS="" # Empty since we used -passout pass: above
DEST_PASS="" # Empty if you want no password, retrieve from keystore, or feed with read -s

/usr/share/elasticsearch/jdk/bin/keytool -importkeystore \
  -destkeystore /etc/elasticsearch/certs/http.p12 \
  -srckeystore /tmp/ca.p12 \
  -srcstoretype PKCS12 \
  -deststoretype PKCS12 \
  -srcstorepass $SRC_PASS \
  -deststorepass $DEST_PASS
```

Then, clean up CA material once it's been imported:

```sh
shred -u /tmp/ca.p12
```

Update the password for the keystore to match the password you just set in the cert utility. In my case, this is blank, since I'm going to blow this cluster away in a few minutes anyway.. but don't do this in production.

```txt
[root@es9-1 ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore remove xpack.security.http.ssl.keystore.secure_password
[root@es9-1 ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
Enter value for xpack.security.http.ssl.keystore.secure_password:
```

Finally, quickly query the `http.p12` keystore to confirm it's valid and that the CA keypair is present:

```sh
/usr/share/elasticsearch/jdk/bin/keytool -list \
  -keystore /etc/elasticsearch/certs/http.p12 \
  -storepass "$DEST_PASS"
```
