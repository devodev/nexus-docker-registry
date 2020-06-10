# nexus-docker-registry

## Introduction

> Make sure nexus dns name is resolvable from the docker engine host (add "127.0.0.1 nexus" to /etc/hosts)

When using nexus3 docker registry plugin, one need to make sure content is served using TLS. This is a hard requirement from the docker client.

By default, nexus3 exposes only a http endpoint. The following two methods explains how to configure a TLS enabled endpoint, either directly from nexus, or using a reverse proxy, and make sure the the certificate used is trusted.

## Method 1: Enable https endpoint served from nexus

> More info here: <https://help.sonatype.com/repomanager3/system-configuration/configuring-ssl#ConfiguringSSL-InboundSSL-ConfiguringtoServeContentviaHTTPS>

Use native compose

```bash
cd native
docker-compose up -d
```

## Method 2: Use reverse proxy

Use reverse-proxy compose

```bash
cd reverse-proxy
docker-compose up -d
```

## Setup server

Add `Docker Bearer Token Realm` to the list of Active Realms.

Create a blobStore with:

- name: docker

Create a docker-proxy repository with:

- name: docker-hub
- Repository Connectors
  - HTTP connector (HTTPS if using Method 1)
    - Port: 18443
  - Allow anonymous docker pull: True
- Docker Registry API Support
  - Enable Docker V1 API: True
- Proxy
  - Remote Storage: <https://registry-1.docker.io>
  - Docker Index: Custom
    - <https://index.docker.io/>
- Storage
  - BlobStore: docker

Create a docker-hosted repository with:

- name: docker-internal
- Repository Connectors
  - HTTP connector (HTTPS if using Method 1)
    - Port: 18442
  - Allow anonymous docker pull: True
- Docker Registry API Support
  - Enable Docker V1 API: True
- Storage
  - BlobStore: docker
- Hosted
  - Deployment policy: Disable redeploy

Create a docker-group with:

- name: docker-group
- Repository Connectors
  - HTTP connector (HTTPS if using Method 1)
    - Port: 18444
  - Allow anonymous docker pull: True
- Docker Registry API Support
  - Enable Docker V1 API: True
- Storage
  - BlobStore: docker
- Group
  - Members
    - docker-hub
    - docker-internal

## Search an image on newly created docker-group

```bash
# Method 1
docker search nexus:18444/postgres
# Method 2
docker search nexus:18445/postgres
```

## Help

### Get nexus admin password

```bash
docker exec -it nexus cat /nexus-data/admin.password
```

### Generate keystore (Method 1)

```bash
# Here, we use CN=nexus and IP=127.0.0.1
keytool -genkeypair -keystore ./etc/ssl/keystore.jks -storepass password -keypass password -alias jetty -keyalg RSA -keysize 2048 -validity 5000 -dname "CN=*.nexus, OU=Example, O=Sonatype, L=Unspecified, ST=Unspecified, C=US" -ext "SAN=DNS:nexus,IP:127.0.0.1" -ext "BC=ca:true" -deststoretype pkcs12
```

### Generate cert and key (Method 2)

> On windows, if using git bash (MSYS/MinGW), set this env var to prevent autoconversion: `MSYS_NO_PATHCONV=1 [CMD..]`

```bash
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ssl/nexus.key -out ssl/nexus.crt -days 365 -nodes -subj '/C=CA/ST=Quebec/L=Montreal/O=Company Name/OU=Org/CN=nexus'
```

### Extract cert and install on your system

Extract using keytool

```bash
keytool -printcert -sslserver <server_host>:<server_port> -rfc > nexus.crt
```

Extract using openssl

```bash
openssl s_client -showcerts -connect <server_host>:<server_port> </dev/null | sed -ne '/BEGIN\ CERTIFICATE/,/END\ CERTIFICATE/ p' 2>/dev/null > nexus.crt
```

Install cert (Windows)

```bash
certutil -addstore root nexus.crt
# restart docker engine
```
