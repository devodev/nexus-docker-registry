# nexus-docker-registry

## Create nexus keystore

> More info here: <https://help.sonatype.com/repomanager3/system-configuration/configuring-ssl#ConfiguringSSL-InboundSSL-ConfiguringtoServeContentviaHTTPS></br></br>
> Make sure the CN used is resolvable from the docker engine host</br>
> For windows, use: C:/Windows/System32/drivers/etc/hosts</br>
> Ex.: echo "127.0.0.1 nexus" >> /etc/hosts

Generate keystore

```bash
keytool -genkeypair -keystore ./etc/ssl/keystore.jks -storepass password -keypass password -alias jetty -keyalg RSA -keysize 2048 -validity 5000 -dname "CN=*.nexus, OU=Example, O=Sonatype, L=Unspecified, ST=Unspecified, C=US" -ext "SAN=DNS:nexus,IP:127.0.0.1" -ext "BC=ca:true" -deststoretype pkcs12
```

Add/update attributes in nexus.properties

```bash
# nexus.properties
application-port-ssl=8443
ssl.etc=${karaf.data}/etc/ssl
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml,${jetty.etc}/jetty-https.xml
```

Extract cert and install it on your system

```bash
keytool -printcert -sslserver nexus:8443 -rfc > nexus.crt
certutil -addstore root nexus.crt
```

restart docker engine
