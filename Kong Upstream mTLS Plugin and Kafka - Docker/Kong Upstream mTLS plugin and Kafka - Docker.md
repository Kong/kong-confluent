# Kong Enterprise and Kafka Upstream mTLS plugin - Docker

## Overview
Event streaming allows developers to build more scalable and loosely coupled real-time applications supporting massive concurrency demands and simplifying the construction of services.

At the same time, API management provides capabilities to securely control the upstream services consumption, including the event processing infrastructure.

This Tech Guide will walk you through the integration between Kong Enterprise and Kafka Event Streaming. We're going to expose Kafka to new and external consumers while applying specific and critical policies to control its consumption, including API key, OAuth/OIDC and others for authentication, rate limiting, caching, log processing, etc.


## System Requirements
Before getting started make sure you have the following tools already installed:

- Docker
- Httpie
- Curl
- Jq
- OpenSSL


## Kafka installation

1. Create a Docker Network<p>
We're going to create a specific Docker Network for both Kong and Kafka containers:

<pre>
docker network create kong-net
</pre>



2. Kafka Installation
Create Zookeeper and Kafka Containers<p>
The Kafka installation uses the official Docker Images provided by Confluent. You can check them out here: https://hub.docker.com/u/confluent

<pre>
docker run -d --name zookeeper -p 2181:2181 --hostname zookeeper --network kong-net -e ZOOKEEPER_CLIENT_PORT=2181 confluentinc/cp-zookeeper:7.0.1


docker run -d --name kafka -p 9092:9092 --hostname kafka --network kong-net --link zookeeper:zookeeper \
-e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
confluentinc/cp-kafka:7.0.1


docker run -d --name kafka-cc -p 9021:9021 --hostname kafka-cc --network kong-net --link zookeeper:zookeeper \
-e CONTROL_CENTER_BOOTSTRAP_SERVERS=kafka:9092 \
-e CONTROL_CENTER_REPLICATION_FACTOR=1 \
-e CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS=1 \
-e CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS=1 \
-e CONFLUENT_METRICS_TOPIC_REPLICATION=1 \
-e PORT=9021 \
confluentinc/cp-enterprise-control-center:7.0.1
</pre>


3. Install local Kafka utilities<p>
For basic testing and event consumption install Kafka utilities locally also. For MacOS run:
<pre>
brew install kafka
</pre>


4. Create a Kafka topic<p>

Insert a "kafka" entry in the /etc/hosts file with 127.0.0.1
<pre>
$ kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
Created topic test.
</pre>

Check the topic with:
<pre>
$ kafka-topics --bootstrap-server localhost:9092 --describe --topic test
Topic: test	TopicId: RSzKeWoHTlS_rQXno9OBgA	PartitionCount: 1	ReplicationFactor: 1	Configs: 
	Topic: test	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
</pre>

If you want to delete the topic run:
<pre>
kafka-topics --delete --bootstrap-server localhost:9092 --topic test
</pre>

Test the Kafka topic
On one local terminal start the Kafka producer:
<pre>
kafka-console-producer --broker-list localhost:9092 --topic test
</pre>

Open another terminal to run the Kafka consumer:
<pre>
kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
</pre>

On the producer type any string and check the string being consumed:
<pre>
$ kafka-console-producer --broker-list localhost:9092 --topic test
>testing
>
</pre>

<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
testing
</pre>

Type ˆC on the consumer:
<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
testing
^CProcessed a total of 1 messages
</pre>



## Certificate Authority creation
In order to implement a mTLS encrypted tunnel with Kafka and Kong, we're going to issue Digital Certificates for both products. A new CA (Certificate Authority) will be created to issue certificates and private keys.

Create a local directory to store all artifacts we're going to produce.

1. Issue the CA's Private Key and Digital Certificate<p>
Create a local file named "AcquaCA.cnf" with the "CA:TRUE" constraint

<pre>
HOME            = .
RANDFILE        = ./rnd

####################################################################
[ ca ]
default_ca    = CA_default      # The default ca section

[ CA_default ]

base_dir      = .
certificate   = ./cacert.pem   # The CA certifcate
private_key   = ./cakey.pem    # The CA private key
new_certs_dir = .              # Location for new certs after signing
database      = ./index.txt    # Database index file
serial        = ./serial.txt   # The current serial number

default_days     = 1000         # How long to certify for
default_crl_days = 30           # How long before next CRL
default_md       = sha256       # Use public key default MD
preserve         = no           # Keep passed DN ordering

x509_extensions = ca_extensions # The extensions to add to the cert

email_in_dn     = no            # Don't concat the email in the DN
copy_extensions = copy          # Required to copy SANs from CSR to cert

####################################################################
[ req ]
default_bits       = 4096
default_keyfile    = cakey.pem
distinguished_name = ca_distinguished_name
x509_extensions    = ca_extensions
string_mask        = utf8only

####################################################################
[ ca_distinguished_name ]
countryName         = Country Name (2 letter code)
countryName_default = BR

stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = Sao Paulo

localityName                = Locality Name (eg, city)
localityName_default        = Sao Paulo

organizationName            = Organization Name (eg, company)
organizationName_default    = Acqua Corp

organizationalUnitName         = Organizational Unit (eg, division)
organizationalUnitName_default = Technology

commonName         = Common Name (e.g. server FQDN or YOUR name)
commonName_default = AcquaCorp

emailAddress         = Email Address
emailAddress_default = acquaviva@uol.com.br

####################################################################
[ ca_extensions ]

subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always, issuer
basicConstraints       = critical, CA:true
keyUsage               = keyCertSign, cRLSign

####################################################################
[ signing_policy ]
countryName            = optional
stateOrProvinceName    = optional
localityName           = optional
organizationName       = optional
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

####################################################################
[ signing_req ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment

</pre>


2. Then create a database and serial number file, these will be used to keep track of which certificates were signed with this CA. Both of these are simply text files that reside in the same directory as your CA keys.

<pre>
echo 01 > serial.txt
touch index.txt
</pre>

3. Submit the file to create the CA's PrivateKey and Digital Certificate, accepting the default values. The command will create the "AcquaCA.key" and "AcquaCA_cert.pem" files:

<pre>
$ openssl req -x509 -config acquaCA.cnf -newkey rsa:4096 -sha256 -nodes -out AcquaCA_cert.pem -keyout AcquaCA.key -outform PEM
Generating a 4096 bit RSA private key
............................................................................................++
................................................++
writing new private key to 'AcquaCA.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [BR]:
State or Province Name (full name) [Sao Paulo]:
Locality Name (eg, city) [Sao Paulo]:
Organization Name (eg, company) [Acqua Corp]:
Organizational Unit (eg, division) [Technology]:
Common Name (e.g. server FQDN or YOUR name) [AcquaCorp]:
Email Address [acquaviva@uol.com.br]:
</pre>






## Kafka Digital Certificate

### Create the Keystore file
In a local directory, create a Keystore file for the Kafka Server:

<pre>
$ keytool -genkey -keystore server.keystore.jks -alias localhost -validity 365 -storepass "serverpwd" -dname "CN=kafka" -keyalg RSA
Generating 2,048 bit RSA key pair and self-signed certificate (SHA256withRSA) with a validity of 365 days
	for: CN=kafka
</pre>

### Create the Certificate Signing Request
<pre>
keytool -keystore server.keystore.jks -alias localhost -certreq -file kafka-server.csr -storepass "serverpwd"
</pre>


### Create the Signed Server Certificate
<pre>
openssl x509 -req -CA AcquaCA_cert.pem -CAkey AcquaCA.key -in kafka-server.csr -out kafka-server.crt -days 365 -CAcreateserial -passin pass:"serverpwd"
</pre>

### Include the CA Certificate and Kafka Signed Certificate in the Keystore
<pre>
$ keytool -keystore server.keystore.jks -alias CARoot -import -file AcquaCA_cert.pem -storepass "serverpwd" --noprompt
Certificate was added to keystore

$ keytool -keystore server.keystore.jks -alias localhost -import -file kafka-server.crt -storepass "serverpwd"
Certificate reply was installed in keystore
</pre>





## Kafka with mTLS on

### Delete Kafka Container

<pre>
docker stop kafka
docker container rm kafka -v
</pre>

### Create a Kafka credentials file
<pre>
echo "serverpwd" > keystore_creds
</pre>


### Start new Container
The new Kafka container has two new settings:
SSL: it enables the specific 9093 port and requests for SSL Client Authentication
It sets all specific Kafka parameters related to the .jks file we crafted before.

<pre>
docker run -d --name kafka -p 9092:9092 -p 9093:9093 --hostname kafka --network kong-net --link zookeeper:zookeeper \
-e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,SSL://kafka:9093 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
-e KAFKA_LISTENERS=PLAINTEXT://:9092,SSL://:9093 \
-e KAFKA_SSL_TRUSTSTORE_LOCATION=/etc/kafka/secrets \
-e KAFKA_SSL_TRUSTSTORE_FILENAME=server.truststore.jks \
-e KAFKA_SSL_TRUSTSTORE_PASSWORD=serverpwd \
-e KAFKA_SSL_KEYSTORE_LOCATION=/etc/kafka/secrets \
-e KAFKA_SSL_KEYSTORE_FILENAME=server.keystore.jks \
-e KAFKA_SSL_KEYSTORE_PASSWORD=serverpwd \
-e KAFKA_SSL_KEY_PASSWORD=serverpwd \
-e KAFKA_SSL_KEY_CREDENTIALS=keystore_creds \
-e KAFKA_SSL_KEYSTORE_CREDENTIALS=keystore_creds \
-e KAFKA_SSL_TRUSTSTORE_CREDENTIALS=keystore_creds \
-e KAFKA_SSL_CLIENT_AUTH=required \
-e KAFKA_SSL_ENDPOINT_IDENTIFICATION_ALGORITHM= \
-v /Users/claudio/kong/tech/Confluent/mTLS-SASL/server.truststore.jks:/etc/kafka/secrets/server.truststore.jks:ro \
-v /Users/claudio/kong/tech/Confluent/mTLS-SASL/server.keystore.jks:/etc/kafka/secrets/server.keystore.jks:ro \
-v /Users/claudio/kong/tech/Confluent/mTLS-SASL/keystore_creds:/etc/kafka/secrets/keystore_creds:ro \
confluentinc/cp-kafka:7.0.1
</pre>


## Kong Enterprise Installation

Set Environment Variable with license
<pre>
export KONG_LICENSE_DATA='{"license":{"version":1,"signature":"YYYYY","payload":{"customer":"Kong_SE_Demo_H1FY22","license_creation_date":"2020-11-30","product_subscription":"Kong Enterprise Edition","support_plan":"None","admin_seats":"5","dataplanes":"5","license_expiration_date":"2021-06-30","license_key":"XXXXX"}}}'
</pre>

### Pull Kong Enterprise Image
<pre>
docker pull kong/kong-gateway:2.7.1.0-alpine

docker tag kong/kong-gateway:2.7.1.0-alpine kong-ee
</pre>


### Install and initialize the Database
<pre>
docker run -d --network kong-net --name kong-ee-database \
   -p 5432:5432 \
   -e "POSTGRES_USER=kong" \
   -e "POSTGRES_DB=kong" \
   -e "POSTGRES_HOST_AUTH_METHOD=trust" \
   postgres:latest

docker run --rm --network kong-net --link kong-ee-database:kong-ee-database \
   -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-ee-database" \
   -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
   -e "KONG_PASSWORD=kong" \
   -e "POSTGRES_PASSWORD=kong" \
   kong-ee kong migrations bootstrap
</pre>


### Start Kong Enterprise
<pre>
docker run -d --network kong-net --name kong-ee --link kong-ee-database:kong-ee-database \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-ee-database" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PORTAL_API_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_PORTAL_API_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
  -e "KONG_ADMIN_GUI_LISTEN=0.0.0.0:8002, 0.0.0.0:8445 ssl" \
  -e "KONG_PORTAL=on" \
  -e "KONG_PORTAL_GUI_PROTOCOL=http" \
  -e "KONG_PORTAL_GUI_HOST=localhost:8003" \
  -e "KONG_PORTAL_SESSION_CONF={\"cookie_name\": \"portal_session\", \"secret\": \"portal_secret\", \"storage\":\"kong\", \"cookie_secure\": false}" \
  -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
  -e "KONG_LOG_LEVEL=debug" \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  -p 8002:8002 \
  -p 8445:8445 \
  -p 8003:8003 \
  -p 8446:8446 \
  -p 8004:8004 \
  -p 8447:8447 \
  kong-ee
</pre>

### Test the installation
<pre>
http :8001 | jq -r .version
</pre>


## Kong Enterprise Kafka Upstream plugin
### Create Kong Service and Route
<pre>
http :8001/services name=kafkaupstreamservice url='http://httpbin.org'

http :8001/services/kafkaupstreamservice/routes name='kafkaupstreamroute' paths:='["/kafkaupstream"]'
</pre>

### Test the Route:
<pre>
$ http :8000/kafkaupstream/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 436
Content-Type: application/json
Date: Fri, 17 Dec 2021 13:34:01 GMT
Server: gunicorn/19.9.0
Via: kong/2.7.0.0-enterprise-edition
X-Kong-Proxy-Latency: 90
X-Kong-Upstream-Latency: 307

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/2.6.0",
        "X-Amzn-Trace-Id": "Root=1-61bc91c9-5c06eb442f037ee02bebb4c2",
        "X-Forwarded-Host": "localhost",
        "X-Forwarded-Path": "/kafkaupstream/get",
        "X-Forwarded-Prefix": "/kafkaupstream"
    },
    "origin": "172.18.0.1, 186.204.48.46",
    "url": "http://localhost/get"
}
</pre>


### Kafka Upstream plugin with mTLS off
<pre>
curl -X POST http://localhost:8001/routes/kafkaupstreamroute/plugins \
    --data "name=kafka-upstream" \
    --data "config.bootstrap_servers[1].host=kafka" \
    --data "config.bootstrap_servers[1].port=9092" \
    --data "config.topic=test" \
    --data "config.timeout=10000" \
    --data "config.keepalive=60000" \
    --data "config.forward_method=false" \
    --data "config.forward_uri=false" \
    --data "config.forward_headers=true" \
    --data "config.forward_body=false" \
    --data "config.producer_request_acks=1" \
    --data "config.producer_request_timeout=2000" \
    --data "config.producer_request_limits_messages_per_request=200" \
    --data "config.producer_request_limits_bytes_per_request=1048576" \
    --data "config.producer_request_retries_max_attempts=10" \
    --data "config.producer_request_retries_backoff_timeout=100" \
    --data "config.producer_async=true" \
    --data "config.producer_async_flush_timeout=1000" \
    --data "config.producer_async_buffering_limits_messages_in_memory=50000"
</pre>


### Check the Route
<pre>
$ http :8001/routes/kafkaupstreamroute
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 499
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:34:33 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Admin-Latency: 6
X-Kong-Admin-Request-ID: Ifve1EHUpMyhOl6i5q40UETkZoReoKSO
vary: Origin

{
    "created_at": 1639748036,
    "destinations": null,
    "headers": null,
    "hosts": null,
    "https_redirect_status_code": 426,
    "id": "a33911db-1a4a-49bc-8ffd-f6f499b33b53",
    "methods": null,
    "name": "kafkaupstreamroute",
    "path_handling": "v0",
    "paths": [
        "/kafkaupstream"
    ],
    "preserve_host": false,
    "protocols": [
        "http",
        "https"
    ],
    "regex_priority": 0,
    "request_buffering": true,
    "response_buffering": true,
    "service": {
        "id": "a2a62b08-60dd-4613-b183-fcd9f1c47648"
    },
    "snis": null,
    "sources": null,
    "strip_path": true,
    "tags": null,
    "updated_at": 1639748036
}
</pre>


### Check the Plugin
<pre>
$ http :8001/plugins
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 1007
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:34:56 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Admin-Latency: 2
X-Kong-Admin-Request-ID: 82ePkc3DhpnjHyv11xl4efVq3KKrvzoC
vary: Origin

{
    "data": [
        {
            "config": {
                "authentication": {
                    "mechanism": null,
                    "password": null,
                    "strategy": null,
                    "tokenauth": null,
                    "user": null
                },
                "bootstrap_servers": [
                    {
                        "host": "kafka",
                        "port": 9092
                    }
                ],
                "forward_body": false,
                "forward_headers": true,
                "forward_method": false,
                "forward_uri": false,
                "keepalive": 60000,
                "keepalive_enabled": false,
                "producer_async": true,
                "producer_async_buffering_limits_messages_in_memory": 50000,
                "producer_async_flush_timeout": 1000,
                "producer_request_acks": 1,
                "producer_request_limits_bytes_per_request": 1048576,
                "producer_request_limits_messages_per_request": 200,
                "producer_request_retries_backoff_timeout": 100,
                "producer_request_retries_max_attempts": 10,
                "producer_request_timeout": 2000,
                "security": {
                    "certificate_id": null,
                    "ssl": null
                },
                "timeout": 10000,
                "topic": "test"
            },
            "consumer": null,
            "created_at": 1639748068,
            "enabled": true,
            "id": "b66694e7-c734-4884-95e1-7b8b5471cd86",
            "name": "kafka-upstream",
            "protocols": [
                "grpc",
                "grpcs",
                "http",
                "https"
            ],
            "route": {
                "id": "a33911db-1a4a-49bc-8ffd-f6f499b33b53"
            },
            "service": null,
            "tags": null
        }
    ],
    "next": null
}
</pre>


### Consume the Route and check the Kafka Topic
Start the Kafka consumer on one local terminal:
<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
testing
</pre>

On another terminal send a request to consume the Route:
<pre>
$ http :8000/kafkaupstream/get aaa:444
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 26
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:35:39 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Response-Latency: 60

{
    "message": "message sent"
}
</pre>


The consumer should show the new message
<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
testing
{"headers":{"connection":"keep-alive","accept-encoding":"gzip, deflate","accept":"*/*","host":"localhost:8000","user-agent":"HTTPie/2.6.0","aaa":"444"}}
</pre>




## Kong Enterprise Digital Certificates

### Create the Kong Private Key
<pre>
openssl genrsa -out kong.key 2048
</pre>

### Create the Kong Public Key
<pre>
openssl rsa -in kong.key -pubout -out kong_public.key
</pre>

### Create the CSR for the Kong Digital Certificate
<pre>
openssl req -new -key kong.key -out kong.csr -subj "/CN=kong"
</pre>

### Issue the Kong Digital Certificate
<pre>
openssl x509 -req -CA AcquaCA_cert.pem -CAkey AcquaCA.key -in kong.csr -out kong.crt -days 365
</pre>



## Kafka Upstream plugin with mTLS on

### Delete the mTLS plugin
We're going to recreate the Kafka mTLS Upstream plugin to turn mTLS on:

<pre>
$ http :8001/plugins | jq -r .data[0].id
b66694e7-c734-4884-95e1-7b8b5471cd86

http delete :8001/plugins/b66694e7-c734-4884-95e1-7b8b5471cd86
</pre>


### Inject the CA Digital Certificate in Kong Enterprise
<pre>
curl -sX POST http://localhost:8001/ca_certificates -F "cert=@./AcquaCA_cert.pem"
</pre>

### Inject the Kong Digital Certificate in Kong Enterprise
<pre>
curl -sX POST http://localhost:8001/certificates \
    -F "cert=@./kong.crt" \
    -F "key=@./kong.key"
</pre>


### Check both Digital Certificates
<pre>
$ http :8001/ca_certificates
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 2401
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:37:13 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Admin-Latency: 2
X-Kong-Admin-Request-ID: 9yR3LvHiRMr6DYbTUkTFfAx0u9enEwTE
vary: Origin

{
    "data": [
        {
            "cert": "-----BEGIN CERTIFICATE-----\nMIIGFTCCA/2gAwIBAgIJAIAk1Jij0hwiMA0GCSqGSIb3DQEBCwUAMIGYMQswCQYD\nVQQGEwJCUjESMBAGA1UECAwJU2FvIFBhdWxvMRIwEAYDVQQHDAlTYW8gUGF1bG8x\nEzARBgNVBAoMCkFjcXVhIENvcnAxEzARBgNVBAsMClRlY2hub2xvZ3kxEjAQBgNV\nBAMMCUFjcXVhQ29ycDEjMCEGCSqGSIb3DQEJARYUYWNxdWF2aXZhQHVvbC5jb20u\nYnIwHhcNMjExMjE3MTMwMDIzWhcNMjIwMTE2MTMwMDIzWjCBmDELMAkGA1UEBhMC\nQlIxEjAQBgNVBAgMCVNhbyBQYXVsbzESMBAGA1UEBwwJU2FvIFBhdWxvMRMwEQYD\nVQQKDApBY3F1YSBDb3JwMRMwEQYDVQQLDApUZWNobm9sb2d5MRIwEAYDVQQDDAlB\nY3F1YUNvcnAxIzAhBgkqhkiG9w0BCQEWFGFjcXVhdml2YUB1b2wuY29tLmJyMIIC\nIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAyjuwN24Qy84dxZjPVjht5c/l\ndVwfxUvyJ2vDk5Og0wPcqC5MDAlD1+mfBzd9fvUqCKJ/pbI8pN1GUcFJWMdqVV+E\ng0kyDHmi16NnBUqaCxwKoV1Hu6WW8fUGKz13z0n7VeOQ4OhE5/Ixve1abUtBey76\n9y1U+rHnIXhQ10joEWmoH4UGGPC1guRpIsSz5oTr8d2VKjMeVKqvRi3h+CiG8WfN\nowPzMVKAt74Og/fYQBmW2FU6RY2LyoFzuTvrm22X3vr0AGZ7pY79FWRsqy04VJ9H\nzvxlBruG9TbRv6AXXP13ROM/VtkE4hQFmpueh0MH5buhCCUd+mputrIVHkgqlUSU\nnzybCm2GpdmklZ/auPMt7zGRP1vMh2zrUwguQTexKeMgzwDjtRC/bslfJ4CzcVb7\njlW8b3qEnxhsa7lgA6Pkvat2juyUM7SZ4aX1NqVa3eU1qhVQXwKqhO4zT0GmzXbP\ngb8/c/g6szaZ90JLqzf9htRt73ibldRxBVIYCesCid2rsJ7urYBEbNSfBJJzzg40\n7bo0m4AGoWZV3SmmUj1QJ0Z9RxLkKiRs0yhjxnAejWnEqsqvSLO+A/WnMLsrHAD8\nPqrYa0hgTb7x5Y02IVAGQmmG2aTujgRf0I2a8I7W3RmYXOkFVJX/iqq2Zd+3/JNB\nPGm6bm8Ri62spUaOlpcCAwEAAaNgMF4wHQYDVR0OBBYEFJ6ydrNYhFe9RPLFRr2r\nLEozjM+aMB8GA1UdIwQYMBaAFJ6ydrNYhFe9RPLFRr2rLEozjM+aMA8GA1UdEwEB\n/wQFMAMBAf8wCwYDVR0PBAQDAgEGMA0GCSqGSIb3DQEBCwUAA4ICAQC3rNotrZeA\n/ElvBdzg7e+5SE0YC4FIZVweAjNoClm+u3/uqEnn7+p1K8woTPWZl//kWU06dk4A\nFBSjXG3vL/pHzufc9atD71YQwZ5uT0VzpIkxVLYIp8nfVAXt8deM+JTs61q1MbSY\nMj+37+BHae6ho9KMGArU4VfOpnE6LqmX5o2mn2Rtt2x1Amf/tBYKpqC5avyoKoE4\nEYf5u1KnNaaAN0wao2TBhyCozMTmdRaLdFCO0LbBj+U7oWmyC8qXEY4+QLhS1Xgo\nSNYXdedbIdERNFLiilA/WXviMHq3qR4F4PnERfWf80pXTXov6Z0xAriuxYDTBDjA\n1RQJhijwBHCSuPW1oNruDX4AQQbt9UyY+JbJT0nUZ3hUn/s6gFwADTW9ThrDGwmg\nSVTqBjsMdH2NXOXhXPA3jOwfyJzV5o5gPTPTphlVqi2sg9mqMfCZEJgXfoyc0rYu\nJ4R/7Toie/d97UNH7AiqVjDk81spUYHbS0xJsVVA+xT6tODaGN8SWhi/22B33ICV\nBuMCvapZVUuSMM+V2UpWRSq/gYXwhWShvZIZl6NqW2vvzfkb9LfkoRW3OSr8I2S7\nWUZrem60Vp/1XeMV6q8l6A1f+TrpGd/LXrihBoyN0HwrbHsYnzh6em/bIy0fo31m\nkzdXPtso/fx/AuQFi4P+rQTrbBUhzPpWMg==\n-----END CERTIFICATE-----\n",
            "cert_digest": "20e9d94a0ddca1a69df0003b3436e90e1b069c9179175066b14773ba611a1dbf",
            "created_at": 1639748220,
            "id": "b8d16238-015c-4c35-9d25-5b1bb0991986",
            "tags": null
        }
    ],
    "next": null
}



$ http :8001/certificates
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 3392
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:37:37 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Admin-Latency: 3
X-Kong-Admin-Request-ID: ofxbx8x936E9jIQhQvz9vCwwK4hFRqen
vary: Origin

{
    "data": [
        {
            "cert": "-----BEGIN CERTIFICATE-----\nMIIEJDCCAgwCCQDpPOCPyakCAjANBgkqhkiG9w0BAQUFADCBmDELMAkGA1UEBhMC\nQlIxEjAQBgNVBAgMCVNhbyBQYXVsbzESMBAGA1UEBwwJU2FvIFBhdWxvMRMwEQYD\nVQQKDApBY3F1YSBDb3JwMRMwEQYDVQQLDApUZWNobm9sb2d5MRIwEAYDVQQDDAlB\nY3F1YUNvcnAxIzAhBgkqhkiG9w0BCQEWFGFjcXVhdml2YUB1b2wuY29tLmJyMB4X\nDTIxMTIxNzEzMzYxOFoXDTIyMTIxNzEzMzYxOFowDzENMAsGA1UEAwwEa29uZzCC\nASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANqLbnca3+v7Irw5RC1dW2BK\nopwBSKPN7sINbcLn+W6LRqZr8mroH9RLl9oN5osxu0B+WZdVQKeVsOUyfewEftmD\nm+BiL7P8DrZmO9pbaCYCo2N9KPIKzhljhnvja1gSvVrl+dGEcHnUBPqN8zXbMVwx\ng6EiYirrv46F+KQwKWCjFt8CdV/qlWsG+Id10jt0zgkuGE6YEGO6JK2ZwGW+AbRB\naw7oU+RurMU9W0yIjCEYnJ66L+aLBdTAmRD68g8pwmmKN52GUkEqruYWtwLp60Qk\nDNtYpKJNs5FBntdXoF/sD2DFP5PZjDIVEOH4EAQEuWL+M1kS+2CGWOQH4no+xlUC\nAwEAATANBgkqhkiG9w0BAQUFAAOCAgEAbVep7wH1Rmi93hBsOl5rSrK3gxLtMdxI\nGQXx+7L/V+1xv9L6dqLjhkEoIcF5me/wH4WIRR8U8Dp8J3yl21mix5z/tHfi83RM\nEazIT56MLlw4cvGzVTP+umBuUhyimwv8KXplRMWFPXfDzkSZJp5+l4oolhHP9WpO\n5U+F28Sj6TPNM/BIMlAnjgXGVvz97p3nb+EoaTyLI+VFeqqPPNpzhhQURXzCT9Mb\nbq2NosBd6+UlWFf0iO9d8pfe9yPTFhkGWdF5+rvM4xFC4uKvW6LEirmIDpkJs4XP\nSpU1xVdS8+nHfWU/mpXTmISgsrMw/zgTggY4p/8Ybb0gk/ANNT02zrOeRVcYCOJf\n4UO6Tt07n9cTp21zM9fI9oq3NDnrsX5PewqvFg3leTNLCZt26rM6Vsx5GS4ek73l\nKcnlHAO1rA9Y2ZagFyvE6I8SAb1lj1fZVh8XMA5HqwjIG/9+kPkHDsBtcUjPBFXU\ntQzhyrSRXYKkqnLhNqJ+Q0RdMPM4217RXn9NgVhMQwDV6y2uh7PPjd4/oHZCKInw\n9Rg9R3zBceShgJemgzNl7gBaGIzGw4/EN91hQSBlvPHytvNfD5HhSmFRYYwBhwKl\ntdo7fvDsrvRf5gAEgM6Vk55VAWhIAEisvvABZFNQR4QksptHW8ojD+tEisNiWaUi\nOF9oraAptiI=\n-----END CERTIFICATE-----\n",
            "cert_alt": null,
            "created_at": 1639748225,
            "id": "e707d119-2b1c-4ee0-8073-f5cbd3958660",
            "key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEA2otudxrf6/sivDlELV1bYEqinAFIo83uwg1twuf5botGpmvy\naugf1EuX2g3mizG7QH5Zl1VAp5Ww5TJ97AR+2YOb4GIvs/wOtmY72ltoJgKjY30o\n8grOGWOGe+NrWBK9WuX50YRwedQE+o3zNdsxXDGDoSJiKuu/joX4pDApYKMW3wJ1\nX+qVawb4h3XSO3TOCS4YTpgQY7okrZnAZb4BtEFrDuhT5G6sxT1bTIiMIRicnrov\n5osF1MCZEPryDynCaYo3nYZSQSqu5ha3AunrRCQM21ikok2zkUGe11egX+wPYMU/\nk9mMMhUQ4fgQBAS5Yv4zWRL7YIZY5Afiej7GVQIDAQABAoIBAQCYqjAniaGEwnFY\nVRS4L/AGCv0ex5LLwq6X5jOXpN7MhwR6ewvj/HVHourYCz/SWpI5EkpZeddpehsR\ncL0gI1/NaK96BnzWWSyZ5D7JYXMWol8qv6LbugqRF8I5RvuUkbqvBdoGr2K26BH2\nSTTtmUoY4gnWhSNYYkj1MccoQvCUrO4ijVfBGvNHCIBQ40478yDZ9LBqDjn2yNSu\n6tSBfzjYXyqxe1Tr5zOUf4xNjitXneAY2S5c1wCMuaFrnAk5D4eJVA86DtAtCewS\niWr2zOmGztQ/qegcM5zUSJ8jMMxvk0w6COGpYfntiuCJj0Bqt8qeD7U3pIhf8dPV\n9FbLoK39AoGBAPc3bMf4cV9DYN0FwxdtFvipkiDWK6BwDmMF3o3G/YioFedicXK3\nPbHtPjyeUJ+3dQeCD99ZCHrMzfCT+uTwiB3Cy82/UwDbx0OxH+ZURtsbrKKFqj0o\nrapF6/qpGxO7y6whAgSVU4J0bCa8GdAscTp4fr/gF/hHM9a+9JBXuwVnAoGBAOJP\nOEIlBY8ZSN9s/PSBwwwBWaU4sgWlM6htHV2emjYpBKlxZFzd4EDcX884TPoynqKi\ndD74z0iHYRoNNhBSv63fgkJPhkKpkbDM5EOkwcKvSKI7yklP5n+4BUXnO1DP2y8n\nVM3zrttTYHdg6d/8qPpM/wy4d1AhIgofcBuNlaTjAoGAXcia7emkKL2I25A6CIML\n+d1qYCafekfITWyGl0ZsHBGX7aV84EX/k6YqvBhbAZw5O1Xt6479Fojnf2LEBWHy\nYUfqxOzV8jduCpIBRgGmt6xx+121zWnHKBdKhFbuvLe7dls3RsHXYmAEP1WQfVa+\nxa28d9HthfSNB+R9Jt0BR/UCgYEA3y5PBfQqukeuNSDviVXa+6DtPmJeNfEIs8X/\n2s7JuDXVciDwYCEzweNS3THhwDBhf3QEfgGzsgxId3+l3I0umRM+C5UPi/hcRGab\nihYWO5/PWqbqREh2wWfCU4DJX1XNC4CXQpBZ1dQw4yoBGzK5ljaOpIXarHwwbJk6\nXwHPHQ8CgYBg4TJTE2x5OtfkLblHehipFUcQTh5bioz4NqiphE7kulpanWWGe7PN\n/GjobPU8xqi1+2a1jM932hTdEPOK8+tmshQ/obbQVxfJbHLbH5BWzHSPY6DAERac\nH46zQdXMh8YMXd3O+6ji+c2UEq5NAS5M51KfIPBlfOb9N/CYFs6iRA==\n-----END RSA PRIVATE KEY-----\n",
            "key_alt": null,
            "snis": [],
            "tags": null
        }
    ],
    "next": null
}
</pre>


### Save the Certificate Id to configure the Kafka plugin
<pre>
$ http :8001/certificates | jq -r .data[0].id
e707d119-2b1c-4ee0-8073-f5cbd3958660
</pre>


### Enable the Kafka mTLS Upstream plugin to the Route with mTLS on
The new Kafka plugin settings include:
. The 9093 port defined by the Kafka Cluster for SSL connections
. SSL config set to true
. Uses the Kong Enterprise Digital Certificate id injected previously.
. Asynchronous configuration

All other settings remain the same

<pre>
curl -X POST http://localhost:8001/routes/kafkaupstreamroute/plugins \
    --data "name=kafka-upstream" \
    --data "config.bootstrap_servers[1].host=kafka" \
    --data "config.bootstrap_servers[1].port=9093" \
    --data "config.security.ssl=true" \
    --data "config.security.certificate_id=e707d119-2b1c-4ee0-8073-f5cbd3958660" \
    --data "config.topic=test" \
    --data "config.timeout=10000" \
    --data "config.keepalive=60000" \
    --data "config.forward_method=false" \
    --data "config.forward_uri=false" \
    --data "config.forward_headers=true" \
    --data "config.forward_body=false" \
    --data "config.producer_request_acks=1" \
    --data "config.producer_request_timeout=2000" \
    --data "config.producer_request_limits_messages_per_request=200" \
    --data "config.producer_request_limits_bytes_per_request=1048576" \
    --data "config.producer_request_retries_max_attempts=10" \
    --data "config.producer_request_retries_backoff_timeout=100" \
    --data "config.producer_async=true" \
    --data "config.producer_async_flush_timeout=1000" \
    --data "config.producer_async_buffering_limits_messages_in_memory=50000"
</pre>

### Check the plugin
<pre>
$ http :8001/plugins
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 1041
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:39:22 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Admin-Latency: 3
X-Kong-Admin-Request-ID: KnC0HJ1wtxTr0eajVYye2h3GN5HVZrEq
vary: Origin

{
    "data": [
        {
            "config": {
                "authentication": {
                    "mechanism": null,
                    "password": null,
                    "strategy": null,
                    "tokenauth": null,
                    "user": null
                },
                "bootstrap_servers": [
                    {
                        "host": "kafka",
                        "port": 9093
                    }
                ],
                "forward_body": false,
                "forward_headers": true,
                "forward_method": false,
                "forward_uri": false,
                "keepalive": 60000,
                "keepalive_enabled": false,
                "producer_async": true,
                "producer_async_buffering_limits_messages_in_memory": 50000,
                "producer_async_flush_timeout": 1000,
                "producer_request_acks": 1,
                "producer_request_limits_bytes_per_request": 1048576,
                "producer_request_limits_messages_per_request": 200,
                "producer_request_retries_backoff_timeout": 100,
                "producer_request_retries_max_attempts": 10,
                "producer_request_timeout": 2000,
                "security": {
                    "certificate_id": "e707d119-2b1c-4ee0-8073-f5cbd3958660",
                    "ssl": true
                },
                "timeout": 10000,
                "topic": "test"
            },
            "consumer": null,
            "created_at": 1639748302,
            "enabled": true,
            "id": "d0fead58-eb8a-48d4-8e66-ee77b2c49000",
            "name": "kafka-upstream",
            "protocols": [
                "grpc",
                "grpcs",
                "http",
                "https"
            ],
            "route": {
                "id": "a33911db-1a4a-49bc-8ffd-f6f499b33b53"
            },
            "service": null,
            "tags": null
        }
    ],
    "next": null
}
</pre>




### Consume the Route and check the Kafka Topic
<pre>
$ http :8000/kafkaupstream/get bbb:555
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 26
Content-Type: application/json; charset=utf-8
Date: Fri, 17 Dec 2021 13:39:58 GMT
Server: kong/2.7.0.0-enterprise-edition
X-Kong-Response-Latency: 60

{
    "message": "message sent"
}
</pre>


The consumer should show the new message
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
{"headers":{"host":"localhost:8000","accept-encoding":"gzip, deflate","user-agent":"HTTPie/2.4.0","accept":"*/*","bbb":"555","connection":"keep-alive"}}