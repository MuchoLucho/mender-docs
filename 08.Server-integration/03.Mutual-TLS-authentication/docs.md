---
title: Mutual TLS authentication
taxonomy:
    category: docs
---

<!-- AUTOMATION: execute=if [ "$TEST_ENTERPRISE" -ne 1 ]; then echo "TEST_ENTERPRISE must be set to 1!"; exit 1; fi -->

<!-- Cleanup code: stops the mTLS ambassador if running -->
<!-- AUTOMATION: execute=function cleanup() { -->
<!-- AUTOMATION: execute=if docker ps | grep registry.mender.io/mendersoftware/mtls-ambassador -->
<!-- AUTOMATION: execute=then -->
<!-- AUTOMATION: execute=docker stop $(docker ps | grep registry.mender.io/mendersoftware/mtls-ambassador | sed 's/ .*//') -->
<!-- AUTOMATION: execute=fi -->
<!-- AUTOMATION: execute=} -->
<!-- AUTOMATION: execute=trap cleanup EXIT -->

!!!!! Mutual TLS authentication is only available in the Mender Enterprise plan.
!!!!! To gain access to the mtls proxy container please [contact support](https://support.northern.tech/hc/en-us).
!!!!! In the message please mention the 'Access to the mtls proxy'.

Mender supports setting up a reverse proxy at the edge of the network, which can authenticate devices using TLS client certificates. Each client presents a certificate signed by a CA certificate (Certificate Authority), and the edge proxy authenticates devices by verifying this signature. Authenticated devices are automatically authorized in the Mender backend, and do not need manual approval.

This is in particular useful in a mass production setting because you can sign client certificates during the manufacturing process, so they automatically get accepted into the Mender Server when your customer turns them on (which might happen several months after manufacturing).

See [Device authentication](../../02.Overview/13.Device-authentication/docs.md) for a general overview of how device authentication works in Mender.

If you are using hosted Mender, you can host the mTLS ambassador in your infrastructure and point it to the upstream server `https://hosted.mender.io`.
In case you need assistance or use a hosted mTLS ambassador, contact us describing your use case.

## Prerequisites

### Access to the mtls-ambassador container

You have to have the correct credentials to get the enterprise container.

```bash
docker logout registry.mender.io
docker login  registry.mender.io
docker run  registry.mender.io/mendersoftware/mtls-ambassador:latest -h
```

As a result mtls ambasador help will be printed which confirms successful access.

```text
time="2023-07-14T15:29:12Z" level=info msg="starting mtls-ambassador" file=main.go func=main.doMain line=37
NAME:
   mtls-ambassador

USAGE:
    [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config FILE  Configuration FILE. Supports JSON, TOML, YAML and HCL formatted configs. (default: "config.yaml")
   --help, -h     show help
```

### A board integrated with Mender

You need a device that has already been integrated with Mender.
You may either a reference or a virtual board from the [getting started](../../01.Get-started/01.Preparation#next-step) section.

### CLI requirements

You need to have `openssl` present on the machine where you will be generating the keys.

## Generate certificates

The following sections guide you to generate and sign certificates for the server and the devices.

!!! This document aims to provide you the basics to evaluate the mTLS authentication in Mender, and you should not consider it as a basis for production-grade PKI (Public-Key infrastructure).

### Generate a CA key pair

The Certificate Authority (CA) key pair is the root of the chain of trust.
These keys are assumed to be trusted, because of those assumption the CA signs it's own certificate.

In production environment the CA private key needs to be kept secure.

<!--AUTOVERSION: "generate a % certificate"/ignore-->
First generate a master certificate to sign each client certificate. Start by generating a private key::

```bash
openssl ecparam -genkey -name P-256 -noout -out ca-private.key
```

!!! You can switch the "P-256" with a different curve if necessary.

Next, create a configuration file which contains information about the Certificate Authority. Execute the following command to create the file.
This is the basis for self signing the CA certificate.

```bash
cat > ca-cert.conf <<EOF
[req]
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
commonName=My CA
organizationName=My Organization
organizationalUnitName=My Unit
emailAddress=myusername@example.com
countryName=NO
localityName=Oslo
stateOrProvinceName=Oslo
EOF
```

Then generate a self signed CA certificate:

```bash
openssl req -new -x509 -key ca-private.key -out ca-cert.pem -config ca-cert.conf -days $((365*10))
```

! The `-days` argument specifies how long the certificate is valid, and you can adjust it as needed. The example expression gives a certificate which is valid for approximately 10 years. Since the CA certificate will only be used on the Mender Server, it is usually not important that it expires, and it's better to have a long expiry time to avoid having to rotate certificates on the devices.

### Generate a mtls proxy certificate

Use OpenSSL to generate a private key using Elliptic Curve cryptography:

```bash
openssl ecparam -genkey -name P-256 -noout -out server-private.key
```

!!! You can switch the "P-256" curve with a different curve if necessary.

Next, we create a configuration file which contains information about the server certificate. Execute the following command to create the file:

```bash
MTLS_DOMAIN="my-server.com"

cat > server-cert.conf <<EOF
[req]
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
commonName=$MTLS_DOMAIN
organizationName=My Organization
organizationalUnitName=My Unit
emailAddress=myusername@example.com
countryName=NO
localityName=Oslo
stateOrProvinceName=Oslo
EOF
```

Fill the fields with information about your organization, locality and contact information. In particular, make sure `commonName` matches the edge proxy's domain name, which will serve as the mTLS ambassador.

Then generate a certificate request from the newly generated private key:

```bash
openssl req -new -key server-private.key -out server-cert.req -config server-cert.conf
```

### Sign the server certificate

Now that we have both a CA certificate, and a certificate request for the server, we need to sign the latter with the former. This produces a signed certificate the edge proxy will use to terminate the TLS traffic.

! This can also be signed by a public CA if you own the domain.
! As we have a fake domain we'll use the internal CA for the signature

```bash
openssl x509 -req -CA ca-cert.pem -CAkey ca-private.key -CAcreateserial -in server-cert.req -out server-cert.pem -days $((365*2))
```

! The `-days` argument specifies how long the certificate is valid, and you can adjust it as needed. The example expression gives a certificate which is valid for approximately 2 years.

### Generate a client certificate

When preparing a client certificate for a device, you generate the certificate key on a separate system (not on the device), and then provision it into the device storage. This way you can keep records of the public key of the device and ensure sufficient entropy during key generation, so the resulting keys are securely random.

!!! Make sure the system you generate keys on is adequately secured, as it will also generate the device private keys. You should consider securely deleting (e.g. `shred`) the *private* keys after provisioning the device if you do not truly need a record of them (you can keep the public keys).

Once again, use OpenSSL to generate a private key using Elliptic Curve cryptography:

```bash
openssl ecparam -genkey -name P-256 -noout -out device-private.key
```

!!! You can switch the "P-256" curve with a different curve if necessary.

Next, we create a configuration file which contains information about the device certificate. Execute the following command to create the file:

```bash
cat > device-cert.conf <<EOF
[req]
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
commonName=my-device-hostname.com
organizationName=My Organization
organizationalUnitName=My Unit
emailAddress=myusername@example.com
countryName=NO
localityName=Oslo
stateOrProvinceName=Oslo
EOF
```

The field `commonName` is device specific, and needs to be changed for every device. Fill the rest of the fields with information about your organization, locality and contact information.

Then generate a certificate request from the newly generated private key:

```bash
openssl req -new -key device-private.key -out device-cert.req -config device-cert.conf
```

### Sign the client certificate

Now that we have both a CA certificate, and a certificate request for the device, we need to sign the latter with the former. This produces a signed certificate which the server will recognize when the client connects.

```bash
openssl x509 -req -CA ca-cert.pem -CAkey ca-private.key -CAcreateserial -in device-cert.req -out device-cert.pem -days $((365*10))
```

! The `-days` argument specifies how long the certificate is valid, and you can adjust it as needed. The example expression gives a certificate which is valid for approximately 10 years. Since the certificate will only be used by the server to authenticate devices, it is usually not desirable that it expires after a short time, since this requires certificate rotation on the devices. To manage compromised devices, it is often better to maintain a certificate blacklist on the server.

You need to repeat the generation and signing of the client certificate for each device, so these are natural steps to automate in your device provisioning workflow.

## Set up the mTLS edge proxy

The mTLS ambassador acts as an edge proxy running in front of your Mender Server. The Mender client running on the devices connects to it, providing its client TLS certificate and establishing a mutual TLS authentication. If the client certificate's signature matches the certification authority recognized by the mTLS ambassador, the Mender Server will automatically accept the device. The edge proxy transparently forwards all the requests from the Mender client to the Mender Server. From the client's perspective, it provides the same API end-points as the upstream Mender Server.

The mTLS ambassador is distributed as a Docker image and can be run on a Docker host, using docker-compose or on Kubernetes.

You need the following certificates to start the service:

* `server.crt`, a regular HTTPS server certificate the ambassador can use to terminate the TLS connections
* `server.key`, the corresponding private key for the certificate above
* `ca.crt`, the Certification Authority's certificate used to sign the server and client certificates.

You also need to specify a username and password pair. The ambassador will use it to connect to the Mender Server to authorize clients who connect using a valid certificate signed by the known CA.

<!--AUTOMATION: ignore -->
```bash
  MTLS_MENDER_USER=mtls@mender.io /
  MTLS_MENDER_PASS=password /
  MTLS_MENDER_BACKEND=https://hosted.mender.io
```
<!-- AUTOMATION: execute=MTLS_MENDER_USER="$CI_MTLS_TEST_HM_USER" -->
<!-- AUTOMATION: execute=MTLS_MENDER_PASS="$CI_MTLS_TEST_HM_PASS" -->
<!-- AUTOMATION: execute=MTLS_MENDER_BACKEND=https://hosted.mender.io -->

As the mtls-ambassador container runs as user `nobody`, with UID 65534, we change the owner of the files we'll volume mount:

<!-- AUTOMATION: execute={ -->
```bash
sudo chown 65534 $(pwd)/server-cert.pem $(pwd)/server-private.key $(pwd)/ca-cert.pem
sudo chmod 0600 $(pwd)/server-private.key
```
<!-- AUTOMATION: execute=} & -->

To start the edge proxy, run the following command:

<!-- AUTOMATION: execute={ -->
<!--AUTOVERSION: "registry.mender.io/mendersoftware/mtls-ambassador:mender-%"/integration-->
```bash
# [Optional] Clean up running containers so there are no surprises
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)

docker run \
  -p 443:8080 \
  -e MTLS_MENDER_USER="$MTLS_MENDER_USER" \
  -e MTLS_MENDER_PASS="$MTLS_MENDER_PASS" \
  -e MTLS_MENDER_BACKEND=$MTLS_MENDER_BACKEND \
  -e MTLS_DEBUG_LOG=true \
  -v $(pwd)/server-cert.pem:/etc/mtls/certs/server/server.crt \
  -v $(pwd)/server-private.key:/etc/mtls/certs/server/server.key \
  -v $(pwd)/ca-cert.pem:/etc/mtls/certs/tenant-ca/tenant.ca.pem \
  registry.mender.io/mendersoftware/mtls-ambassador:latest
```
<!-- AUTOMATION: execute=} & -->

<!-- AUTOMATION: execute=for i in {1..10}  -->
<!-- AUTOMATION: execute=do -->
<!-- AUTOMATION: execute=sleep 1 -->
<!-- AUTOMATION: execute=if docker ps | grep registry.mender.io/mendersoftware/mtls-ambassador | grep Up -->
<!-- AUTOMATION: execute=then -->
<!-- AUTOMATION: execute=break -->
<!-- AUTOMATION: execute=else -->
<!-- AUTOMATION: execute=echo "The mTLS ambassador container is not 'Up', retrying" -->
<!-- AUTOMATION: execute=fi; -->
<!-- AUTOMATION: execute=done; -->

<!-- AUTOMATION: execute=sleep 5 -->
<!--AUTOMATION: test=docker ps | grep registry.mender.io/mendersoftware/mtls-ambassador | grep Up -->

Replace the following values with the ones that match your configuration:

* **MTLS_MENDER_USER** and **MTLS_MENDER_PASS** are the user security credentials that allow the mTLS ambassador to connect to the Mender Server and authorize new devices connecting using the mTLS authentication.
* **MTLS_MENDER_BACKEND** is the URL of the upstream Mender Server; the edge proxy will forward the HTTPS requests to.
* **MTLS_DEBUG_LOG** (optional) enables verbose debugging log.
* **server.crt** and **server.key** are the paths to your server TLS certificate and key.
* **ca-crt.pem** is the file which contains the certificate of the Certification Authority.

You can now publish the HTTPS port of the host to the Internet to let the clients connect to it.

### Confirm communication with the proxy

Assuming you run the docker image on your host, the container will start listening to the host port 443.

You can test this by getting the host IP:

``` bash
# Example how to do it on a Linux machine
# If it doesn't work please fill it manually
MTLS_IP=$(ip route get 8.8.8.8 | awk '{print $7}' | head -n 1)

# While looking at the logs of the container shell run
openssl s_client -connect $MTLS_IP:443

# If you see something like:
# 2023/08/18 15:51:21 http: TLS handshake error from 192.168.88.249:46612: tls: client didn't provide a certificate
# This means you can communicate with the server correctly
```

## Configure the device to use the proxy

Now that we have generated a key and certificate for the device and signed the certificate, we need to configure the device to use the proxy.

### Using a virtual device

```bash

# Replace with your Hosted Mender tenant token
TENANT_TOKEN=4324h23kj4h23kj4h


# Start the virtual client
# It will run in daemon mode
docker run -d -it -p 85:85 --pull=always mendersoftware/mender-client-qemu

CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aqf "ancestor=mendersoftware/mender-client-qemu"))

# Update certificates to the device
scp -o StrictHostKeyChecking=no -P 8822 device-private.key root@$CONTAINER_IP:/data/mender/mender-cert-private.pem
scp -o StrictHostKeyChecking=no -P 8822 device-cert.pem root@$CONTAINER_IP:/data/mender/mender-cert.pem

ssh -o StrictHostKeyChecking=no -p 8822 root@$CONTAINER_IP systemctl stop mender-client

# Create mender.conf and modified etc/hosts
cat > mender.conf << EOF
{
  "InventoryPollIntervalSeconds": 5,
  "ServerURL": "https://$MTLS_DOMAIN",
  "TenantToken": "$TENANT_TOKEN",
  "UpdatePollIntervalSeconds": 5,
  "HttpsClient": {
    "Certificate": "/data/mender/mender-cert.pem",
    "Key": "/data/mender/mender-cert-private.pem"
  }
}
EOF

cat > hosts << EOF
127.0.0.1   localhost.localdomain           localhost
$MTLS_IP    $MTLS_DOMAIN
EOF

# Copy configuration files into the device's rootfs
scp -o StrictHostKeyChecking=no -P 8822 mender.conf root@$CONTAINER_IP:/etc/mender/mender.conf
scp -o StrictHostKeyChecking=no -P 8822 hosts root@$CONTAINER_IP:/etc/hosts
scp -o StrictHostKeyChecking=no -P 8822 ca-cert.pem root@$CONTAINER_IP:/usr/local/share/ca-certificates/mender/ca-cert.crt

# Load the certificates on the device and start the mender-client service
ssh -o StrictHostKeyChecking=no -p 8822 root@$CONTAINER_IP update-ca-certificates
ssh -o StrictHostKeyChecking=no -p 8822 root@$CONTAINER_IP systemctl start mender-client
```

### Using a real device

Using the real device requires `mender-artifact` tool from the [Downloads section](../../10.Downloads/docs.md).

#### Copy key and certificate into disk image

Find the location of the [key and certificate we generated](#generate-certificates) and copy it into place on the data partition by running the following commands:

<!--AUTOMATION: ignore -->
```bash
mender-artifact install -m 600 device-private.key mender-disk-image.sdimg:/data/mender/mender-cert-private.pem
mender-artifact install -m 644 device-cert.pem mender-disk-image.sdimg:/data/mender/mender-cert.pem
```

!!! The files we just added are per device, and therefore it is natural to automate this step in your device provisioning workflow.

#### Set up Mender configuration for client certificate

First, copy the existing `mender.conf` out of the disk image, so that we can edit it.

<!--AUTOMATION: ignore -->
```bash
mender-artifact cp mender-disk-image.sdimg:/etc/mender/mender.conf mender.conf
```

Next, open `mender.conf` in a text editor, and add the following content:

```json
  "HttpsClient": {
    "Certificate": "/data/mender/mender-cert.pem",
    "Key": "/data/mender/mender-cert-private.pem"
  }
```

Make sure that the result is valid JSON, in particular that commas appear on every line except the last in a block. Add the snippet inside the first set of curly braces in the file. For example, it might look like this in a typical `mender.conf` file:

TODO: Why does this work? It has [Hosted Mender URL](https://hosted.mender.io/) while mtls proxy is set on my-server.com.

```json
{
  "ServerURL": "https://hosted.mender.io/",
  "TenantToken": "TENANT_TOKEN",
  "HttpsClient": {
    "Certificate": "/data/mender/mender-cert.pem",
    "Key": "/data/mender/mender-cert-private.pem"
  }
}
```

Then copy the modified file back into the disk image:

<!--AUTOMATION: ignore -->
```bash
mender-artifact cp mender.conf mender-disk-image.sdimg:/etc/mender/mender.conf
```

!!! Since this change is the same on every device, it is natural to automate this as part of the build process for the disk image. See file installation instructions for [the Debian family](../../04.Operating-System-updates-Debian-family/03.Customize-Mender/docs.md#configuration-file) or [the Yocto Project](../../05.Operating-System-updates-Yocto-Project/05.Customize-Mender/docs.md#configuration-file) for more information.

### Boot the device

Now provision the storage with this new disk image, just like you have done in the past. If you are using a SD card, insert it into your workstation and use a command similar to the following:

<!--AUTOMATION: ignore -->
```bash
sudo dd if=<PATH-TO-IMAGE>.sdimg of=<DEVICE> bs=1M && sudo sync
```

Then insert the SD card back into your device and boot it.

## Verify that the device is accepted

If everything went as intended, your device shows up as `accepted` status in the Mender Server. You can log in to the Mender UI to ensure your device appears on the device list and reports inventory.

If your device is not showing up, make sure you installed the certificates correctly - both on the server and on the device. Check client logs and/or server logs for error messages that can identify what is wrong. See the [troubleshooting section on connecting devices](../../301.Troubleshoot/05.Device-Runtime/docs.md#mender-server-connection-issues) in this case.

## Check the health of the `mtls-ambassador`

You can rely on the `/status` endpoint for checking the ambassador status. You can check it from a device (physical or virtual) with the right certificates.

Run the following command:

```bash
MTLS_DOMAIN=my-server.com
curl -X GET --cert /data/mender/mender-cert.pem --key /data/mender/mender-cert-private.pem  https://${MTLS_DOMAIN}/status
```
