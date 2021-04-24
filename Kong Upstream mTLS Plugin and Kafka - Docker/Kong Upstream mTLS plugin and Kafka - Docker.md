# Kong Enterprise and Kafka Upstream mTLS plugin - Docker

## Overview
Event streaming allows developers to build more scalable and loosely coupled real-time applications supporting massive concurrency demands and simplifying the construction of services.

At the same time, API management provides capabilities to securely control the upstream services consumption, including the event processing infrastructure.

This Tech Guide will walk you through the integration between Kong Konnect Enterprise and Kafka Event Streaming. We're going to expose Kafka to new and external consumers while applying specific and critical policies to control its consumption, including API key, OAuth/OIDC and others for authentication, rate limiting, caching, log processing, etc.


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
docker run -d --name zookeeper -p 2181:2181 --hostname zookeeper --network kong-net confluent/zookeeper

docker run -d --name kafka -p 9092:9092 --hostname kafka --network kong-net --link zookeeper:zookeeper confluent/kafka
</pre>


3. Install local Kafka utilities<p>
For basic testing and event consumption install Kafka utilities locally also. For MacOS run:
<pre>
brew install kafka
</pre>


4. Create a Kafka topic<p>

Insert a "kafka" entry in the /etc/hosts file with 127.0.0.1
<pre>
$ kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
Created topic test.
</pre>

Check the topic with:
<pre>
$ kafka-topics --zookeeper localhost:2181 --describe --topic test
Topic: test	PartitionCount: 1	ReplicationFactor: 1	Configs: 
	Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
</pre>

If you want to delete the topic run:
<pre>
kafka-topics --delete --zookeeper localhost:2181 --topic test
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


## Kong Enterprise Installation

In this guide we'll be using the OpenID Connect (OIDC) plug-in for Kong, which is only available for Kong Enterprise. If you don't already have a Kong Enterprise account, you can get a 30-day trial here.

1. Login to Kong Bintray using your credentials
<pre>
docker login -u <BINTRAY_USER_ID> -p <BINTRAY_API_KEY> kong-docker-kong-enterprise-edition-docker.bintray.io
</pre>

2. Define a environment variable with your Kong Enterprise license
<pre>
export KONG_LICENSE_DATA='{"license":{"version":1,"signature":"YYYYY","payload":{"customer":"Kong_SE_Demo_H1FY22","license_creation_date":"2020-11-30","product_subscription":"Kong Enterprise Edition","support_plan":"None","admin_seats":"5","dataplanes":"5","license_expiration_date":"2021-06-30","license_key":"XXXXX"}}}'
</pre>

3. Pull Kong Enterprise Image
<pre>
docker pull kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:2.2.0.0-alpine

docker tag kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:2.2.0.0-alpine kong-ee
</pre>

4. Install and initialize the Database
<pre>
docker run -d --name kong-ee-database \
   -p 5432:5432 \
   -e "POSTGRES_USER=kong" \
   -e "POSTGRES_DB=kong" \
   -e "POSTGRES_HOST_AUTH_METHOD=trust" \
   postgres:latest

docker run --rm --link kong-ee-database:kong-ee-database \
   -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-ee-database" \
   -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
   -e "KONG_PASSWORD=kong" \
   -e "POSTGRES_PASSWORD=kong" \
   kong-ee kong migrations bootstrap
</pre>

5. Start Kong Enterprise
<pre>
docker run -d --name kong-ee --link kong-ee-database:kong-ee-database \
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

6. Test the installation
<pre>
docker network connect kong-net kong-ee
docker network connect kong-net kong-ee-database

http --verify=no https://localhost:8443
http --verify=no https://localhost:8444

docker stop kong-ee
docker stop kong-ee-database

docker start kong-ee-database
docker start kong-ee

http :8001 | jq .version
</pre>

## Kong Enterprise and Kafka mTLS Upstream plugin installation
1. Clone the Git repository locally
<pre>
git clone --branch enhanced-security https://github.com/kong/kong-plugin-kafka-upstream.git

cd kong-plugin-kafka-upstream

luarocks make *.rockspec
</pre>

2. Copy the plugin to the Kong Enterprise Container
<pre>
docker cp /Users/claudio/kong/tech/Confluent/Plugin/kong-plugin-kafka-upstream/kong/plugins/kafka-upstream kong-ee:/usr/local/share/lua/5.1/kong/plugins
</pre>

3. Check the Container:
<pre>
$ docker exec -ti -u0 kong-ee /bin/sh
/ # ls -l /usr/local/share/lua/5.1/kong/plugins
total 260
drwxr-xr-x    3 root     root          4096 Nov 17 22:10 acl
drwxr-xr-x    4 root     root          4096 Nov 17 22:10 acme
...
drwxr-xr-x    2 root     root          4096 Nov 17 22:10 kafka-log
drwxr-xr-x    1 502      dialout       4096 Dec 30 14:40 kafka-upstream
...
/ # chown -R root /usr/local/share/lua/5.1/kong/plugins/kafka-upstream
/ # chgrp -R root /usr/local/share/lua/5.1/kong/plugins/kafka-upstream
/ # exit

docker stop kong-ee
docker start kong-ee
</pre>

4. Create Kong Service and Route
<pre>
http :8001/services name=httpbinservice url='http://httpbin.org'

http :8001/services/httpbinservice/routes name='httpbinroute' paths:='["/httpbin"]'
</pre>

5. Test the Route:
<pre>
$ http :8000/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 426
Content-Type: application/json
Date: Wed, 30 Dec 2020 14:34:23 GMT
Server: gunicorn/19.9.0
Via: kong/2.2.0.0-enterprise-edition
X-Kong-Proxy-Latency: 71
X-Kong-Upstream-Latency: 324

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/2.3.0",
        "X-Amzn-Trace-Id": "Root=1-5fec8fef-4b84bf6f1715fb2f4913f1c6",
        "X-Forwarded-Host": "localhost",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "172.17.0.1, 186.204.132.234",
    "url": "http://localhost/get"
}
</pre>


## Testing Kafka mTLS Upstream plugin with mTLS off
1. Configure the plugin
<pre>
curl -X POST http://localhost:8001/routes/httpbinroute/plugins \
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


2. Check the Route
<pre>
$ http :8001/routes/httpbinroute
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 488
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 14:54:47 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Admin-Latency: 12
X-Kong-Admin-Request-ID: KYQftUcK2y22TmKbyJD6s7iB26S8wUYa
vary: Origin

{
    "created_at": 1609338844,
    "destinations": null,
    "headers": null,
    "hosts": null,
    "https_redirect_status_code": 426,
    "id": "b4fcb711-b8dc-467b-955a-d231c973d932",
    "methods": null,
    "name": "httpbinroute",
    "path_handling": "v0",
    "paths": [
        "/httpbin"
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
        "id": "72cefabd-0200-4ddf-b000-7cc95410fd03"
    },
    "snis": null,
    "sources": null,
    "strip_path": true,
    "tags": null,
    "updated_at": 1609338844
}
</pre>

3. Check the Plugin
<pre>
$ http :8001/plugins
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 879
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 14:55:31 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Admin-Latency: 4
X-Kong-Admin-Request-ID: vfgG9hNGaaS5bi66H9jb2CxozDItEYM7
vary: Origin

{
    "data": [
        {
            "config": {
                "bootstrap_servers": [
                    {
                        "certificate_id": null,
                        "host": "kafka",
                        "port": 9092,
                        "tls_enable": false
                    }
                ],
                "forward_body": false,
                "forward_headers": true,
                "forward_method": false,
                "forward_uri": false,
                "keepalive": 60000,
                "producer_async": true,
                "producer_async_buffering_limits_messages_in_memory": 50000,
                "producer_async_flush_timeout": 1000,
                "producer_request_acks": 1,
                "producer_request_limits_bytes_per_request": 1048576,
                "producer_request_limits_messages_per_request": 200,
                "producer_request_retries_backoff_timeout": 100,
                "producer_request_retries_max_attempts": 10,
                "producer_request_timeout": 2000,
                "timeout": 10000,
                "topic": "test"
            },
            "consumer": null,
            "created_at": 1609340041,
            "enabled": true,
            "id": "5a32f7de-132b-42c2-aea7-7b35450970c7",
            "name": "kafka-upstream",
            "protocols": [
                "grpc",
                "grpcs",
                "http",
                "https"
            ],
            "route": {
                "id": "b4fcb711-b8dc-467b-955a-d231c973d932"
            },
            "service": null,
            "tags": null
        }
    ],
    "next": null
}
</pre>

4. Consume the Route and check the Kafka Topic<p>
Start the Kafka consumer on one local terminal:
<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
testing
</pre>

On another terminal send a request to consume the Route:
<pre>
$ http :8000/httpbin/get aaa:444
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 26
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 14:59:20 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Response-Latency: 30

{
    "message": "message sent"
}
</pre>

The consumer should show the new message
<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
testing
{"headers":{"host":"localhost:8000","accept-encoding":"gzip, deflate","user-agent":"HTTPie\/2.3.0","accept":"*\/*","aaa":"444","connection":"keep-alive"}}
</pre>



## Certificate Authority creation
In order to implement a mTLS encrypted tunnel with Kafka and Kong, we're going to issue Digital Certificates for both products. A new CA (Certificate Authority) will be created to issue certificates and private keys.

Create a local directory to store all artifacts we're going to produce.

1. Create the new Certificate Authority (CA) private key<p>
The command creates a "acquaCA.key" file.

<pre>
$ openssl genrsa -out acquaCA.key
Generating RSA private key, 2048 bit long modulus
..................................................+++
.+++
e is 65537 (0x10001)
</pre>

2. Issue a Digital Certificate based on the private key<p>
The command creates a "acquaCA.crt" file.

<pre>
$ openssl req -x509 -new -key acquaCA.key -out acquaCA.crt
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:BR
State or Province Name (full name) []:Sao Paulo
Locality Name (eg, city) []:Sao Paulo
Organization Name (eg, company) []:Acqua Corp
Organizational Unit Name (eg, section) []:Technology
Common Name (eg, fully qualified host name) []:acqua.com
Email Address []:acquaviva@uol.com.br
</pre>

3. Checking the new CA's private key and Digital Certificate
<pre>
$ openssl rsa -in acquaCA.key -check
RSA key ok
writing RSA key
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA1+8B4/fdBjnIRRc/W0HScMEpMwzNjKfE0BGbkOs+Vs16xzlW
RkTZaL/DGFbPNJV8V0DT//5SZrwjGHXfTooS0dqpUi/c0I2fk7IFT3Rm5MhcEGHh
/Pz6ox9zuOks0Lpp9DLZYK5NjxYcAMlV7SmRoDMCrh5kVxpfRj+sgnhMHPuSLk/x
Y3BYuv97b5+a2pS4zSyRonMy+hynEXLByn+3iNzzKHwWAnJR0bbJcuFKGDZYJu5y
+6r5cy0mqEU3SYU+4SPF+x/50P9tCyjkRZfxDVYJyaGXgIDD+PJi7MP4BHxR1KBV
0RSyTzECFh206QBeuyyhaRZHw01SYCoOLmXn2wIDAQABAoIBAAZrGEdKastwlD9Z
fYyc3EB1vV/DFakEo5j7rQAVvfieivO5BJN6IGw4pvfmPKp3dwaw6pxFVvWuyexE
NKsE96I9OaMzwQCB9ShStk2yTAyo1/O0tR7r9hc7LBlm9OoPYG7dxBBXnf6Oza5I
TcGK5sU4PvAl/x2HryVLZzlJkhmaYq3+KSXGqkC1Soi3zZ7hJOfPiIaQq5eOUFSi
OSF6dMkU4IQpEgU236NJgpJdok1lMQTyiD6BvSjRdfyAaQ3wl63WdaiSdxxoJXmc
t4QGKjG+qb3SHoL7x+Nq2HqoZBWOOYjEJ5ZocN5kQ/5LUspMeyc93uLlOcsIZCVp
giNgsgECgYEA/S8ONtKaykuKeVqFQXEqZTFYWppKspOVJ/NDFEZURMI0wZaUwMRX
h8lEDRx8DCBGB3B2LhxUcC3kgtoGP5q6CKWSPQjExzCDey30yGfHVdd9ZuAGP44h
XziYo52QtElXUJUn6CLg5Z+uKapOdw+Y+ZN68Y2XLx6FYSoHmxJLOZsCgYEA2lXh
pQWOlmcbN/ZOm9Go6MXTHpNNilvD9pUtV98qGXkqWWJilAW9olLxLV9OBmAWXyZ+
8jTC3he7ZHe4t8Snx4XupO8vm+iZIoUhCV7oftYP5iJlrol2q9Jjv6MxU8VavhhZ
XhlvS4ZcKIMHWHHxdG+TcbpdXPJK8Pl3JCSrDsECgYAc92c+6nV/M4lSPQMF67aY
AT9EjmaBa9Uizvgbt7gobbevdlTqgQwqouJARcQDdyXL8Bf1SpR2iSmdtugEGuWx
24+RoBEzYN+KFkXtL8JkldTpEjRkzRQQWt9LyNknZ0SwGYCJVIQ6gTxh0/RKNuSf
mTn1rOdhIrLL3Q0ltsAYhQKBgQC8G+Ic23zN+Gdq/7saZLiyVD5gyWi1G/rqJ/y5
CHytFcd2210zSv7nK66++K2wsHiV4gTdiLebwbaiCMQNEFG9hZbmY20RVoUZSLn9
6NdG8Acir+ALUEP+JXXrVh7Znd9giHn2qNNKrqgX/0wE16bAOqE+CuMFgXsvwr7z
VORMAQKBgQCJvDlVcYxrIOPIXGFdqoN7y3+HIHJW5T3jmd+MyM9rH06u7B3laMG4
hm/EONjzPq1gVoQNVw4Xt4CFPG8gwb3UKE+Yo6+v7orpIanyIALQXtx/XJoIOGUU
UEhaWlEr8WnSp/A7j8RrE195jiwnAH/OO8y5a9BJEtowZYT46ymyvQ==
-----END RSA PRIVATE KEY-----
</pre>

<pre>
$ openssl x509 -in acquaCA.crt -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 9575856616369054695 (0x84e446db86c063e7)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=BR, ST=Sao Paulo, L=Sao Paulo, O=Acqua Corp, OU=Technology, CN=acqua.com/emailAddress=acquaviva@uol.com.br
        Validity
            Not Before: Dec 30 15:19:40 2020 GMT
            Not After : Jan 29 15:19:40 2021 GMT
        Subject: C=BR, ST=Sao Paulo, L=Sao Paulo, O=Acqua Corp, OU=Technology, CN=acqua.com/emailAddress=acquaviva@uol.com.br
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d7:ef:01:e3:f7:dd:06:39:c8:45:17:3f:5b:41:
                    d2:70:c1:29:33:0c:cd:8c:a7:c4:d0:11:9b:90:eb:
                    3e:56:cd:7a:c7:39:56:46:44:d9:68:bf:c3:18:56:
                    cf:34:95:7c:57:40:d3:ff:fe:52:66:bc:23:18:75:
                    df:4e:8a:12:d1:da:a9:52:2f:dc:d0:8d:9f:93:b2:
                    05:4f:74:66:e4:c8:5c:10:61:e1:fc:fc:fa:a3:1f:
                    73:b8:e9:2c:d0:ba:69:f4:32:d9:60:ae:4d:8f:16:
                    1c:00:c9:55:ed:29:91:a0:33:02:ae:1e:64:57:1a:
                    5f:46:3f:ac:82:78:4c:1c:fb:92:2e:4f:f1:63:70:
                    58:ba:ff:7b:6f:9f:9a:da:94:b8:cd:2c:91:a2:73:
                    32:fa:1c:a7:11:72:c1:ca:7f:b7:88:dc:f3:28:7c:
                    16:02:72:51:d1:b6:c9:72:e1:4a:18:36:58:26:ee:
                    72:fb:aa:f9:73:2d:26:a8:45:37:49:85:3e:e1:23:
                    c5:fb:1f:f9:d0:ff:6d:0b:28:e4:45:97:f1:0d:56:
                    09:c9:a1:97:80:80:c3:f8:f2:62:ec:c3:f8:04:7c:
                    51:d4:a0:55:d1:14:b2:4f:31:02:16:1d:b4:e9:00:
                    5e:bb:2c:a1:69:16:47:c3:4d:52:60:2a:0e:2e:65:
                    e7:db
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
         37:ba:37:94:02:40:4b:23:59:04:e5:4a:da:a4:f6:4e:57:ce:
         68:04:e9:82:5a:fc:22:96:a0:e3:3b:c4:7a:94:34:ff:73:23:
         0b:d5:59:0b:19:4c:12:f0:0b:59:7e:e1:5e:85:1a:26:6c:37:
         ca:5b:87:cf:5f:ba:88:3e:9e:15:e4:76:0b:0d:f8:82:eb:33:
         28:d1:f9:5a:c5:12:21:97:22:f0:58:3e:5f:68:ac:a4:af:a7:
         2c:cb:ae:3a:4d:16:fb:1e:11:49:5a:45:80:58:7f:28:70:c5:
         4d:b9:2d:eb:5a:7e:93:d5:b6:92:68:2c:55:1d:b0:60:42:24:
         7f:57:b6:48:cc:92:ec:3b:ed:ee:e7:da:2c:cf:50:af:75:64:
         30:fb:c9:03:83:59:61:39:37:e3:d3:e6:2d:95:b2:d4:7b:a9:
         ac:4d:e0:e8:d1:2e:a6:25:71:9b:af:e8:bb:f6:f4:7a:0b:3a:
         fc:15:ff:a6:b2:f0:80:32:76:c2:42:23:82:f3:b9:25:ce:0b:
         b6:51:5b:b4:f6:30:fe:cf:0f:2c:05:69:44:f4:52:e3:a5:59:
         4c:08:1c:06:72:92:52:bf:f4:78:5a:62:12:30:57:0b:33:cf:
         3a:c8:57:2b:15:43:5d:b9:53:b9:2f:d9:94:6d:bf:71:d5:2f:
         dc:a4:66:77
</pre>


4. Issue a self-signed certificate for the CA<p>
Create a local file named "acquaCA_csr.conf" with the "CA:TRUE" constraint

<pre>
[ req ]
distinguished_name       = req_distinguished_name
extensions               = v3_ca
req_extensions           = v3_ca

[ v3_ca ]
basicConstraints         = CA:TRUE

[ req_distinguished_name ]
countryName              = Country Name (2 letter code)
countryName_default      = BR
countryName_min          = 2
countryName_max          = 2
organizationName         = Organization Name (eg, company)
organizationName_default = Acqua Corp
</pre>

<p>Submit the file to create a CSR ("Certificate signing request"), accepting the default values. The command will create a "acquaCA.csr" file:
<pre>
openssl req -new -sha256 -key acquaCA.key -nodes -out acquaCA.csr -config acquaCA_csr.conf
</pre>

Issue the self-signed certificate. The command creates the "acquaCA.pem" file.
<pre>
$ openssl x509 -req -days 3650 -extfile acquaCA_csr.conf -extensions v3_ca -in acquaCA.csr -signkey acquaCA.key -out acquaCA.pem
Signature ok
subject=/C=BR/O=Acqua Corp
Getting Private key
</pre>


## Kong Enterprise Digital Certificate
1. Issue a Private Key for Kong Enterprise<p>
The command creates the "kong.key" file.
<pre>
openssl genrsa -out kong.key
</pre>

2. Issue a CSR (Certificate Signing Request) for the Kong Enterprise Digital Certificate<p>
The command creates the "kong.csr" file.
<pre>
$ openssl req -new -key kong.key -out kong.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:BR
State or Province Name (full name) []:Sao Paulo
Locality Name (eg, city) []:Sao Paulo
Organization Name (eg, company) []:Kong Corp
Organizational Unit Name (eg, section) []:Technology
Common Name (eg, fully qualified host name) []:kong.com
Email Address []:acquaviva@uol.com.br

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:kong
</pre>

3. Issue the Kong Enterprise Digital Certificate<p>
The command creates the "kong.crt" file.
<pre>
$ openssl x509 -req -in kong.csr -days 3650 -sha1 -CAcreateserial -CA acquaCA.crt -CAkey acquaCA.key -out kong.crt
Signature ok
subject=/C=BR/ST=Sao Paulo/L=Sao Paulo/O=Kong Corp/OU=Technology/CN=kong.com/emailAddress=acquaviva@uol.com.br
Getting CA Private Key
</pre>

4. Check the Kong Enterprise Certificate
<pre>
$ openssl x509 -in kong.crt -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 17102241651444505032 (0xed575e2ba13d59c8)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: C=BR, ST=Sao Paulo, L=Sao Paulo, O=Acqua Corp, OU=Technology, CN=acqua.com/emailAddress=acquaviva@uol.com.br
        Validity
            Not Before: Dec 30 15:41:31 2020 GMT
            Not After : Dec 28 15:41:31 2030 GMT
        Subject: C=BR, ST=Sao Paulo, L=Sao Paulo, O=Kong Corp, OU=Technology, CN=kong.com/emailAddress=acquaviva@uol.com.br
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a9:2b:7b:ea:c3:32:e6:fe:f5:76:99:bb:5d:fb:
                    72:1d:64:a6:6f:5f:04:c6:fb:23:34:65:66:cc:db:
                    06:24:c7:b8:14:2e:94:43:ac:5e:2f:b6:f8:37:e5:
                    9c:c7:a8:79:f8:63:38:6c:1a:0e:0e:3b:b4:87:ed:
                    2d:36:1b:64:ea:48:cf:aa:88:f9:be:78:4d:f8:87:
                    68:2f:d7:7b:26:c4:ce:e0:ba:df:9f:06:c8:13:11:
                    57:18:1c:c0:ef:e2:ad:90:dd:e3:84:f4:69:4a:c1:
                    47:6c:1f:09:f2:03:34:70:56:94:a6:32:1a:78:89:
                    f9:f7:64:af:1d:08:c1:b7:0a:d2:a3:e8:2a:90:a5:
                    91:71:29:b7:c0:01:07:e7:91:64:18:cc:b2:16:45:
                    8e:f4:ed:84:52:d1:51:69:4d:1d:2b:c8:8a:90:b5:
                    bc:9f:62:71:9f:7a:02:a8:b1:ea:5c:b2:30:5d:2b:
                    32:96:59:5a:1a:5a:d5:13:9e:10:d7:09:29:36:bd:
                    a8:31:ec:e9:a4:2a:8a:00:30:62:7a:a6:be:0e:07:
                    65:94:fe:7e:40:0c:12:02:e5:c8:23:67:3f:fe:4e:
                    cc:72:21:8c:1c:79:a7:b0:ec:e2:c0:dd:11:09:e2:
                    f6:6b:5c:38:db:58:70:54:37:d4:f7:c3:bf:49:23:
                    b7:b5
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha1WithRSAEncryption
         14:be:e7:2d:7c:d2:cc:96:44:52:0e:fa:35:7e:61:95:41:57:
         3a:b3:c1:6f:15:54:86:a3:4c:e7:ba:d8:f5:70:84:40:d3:fb:
         5c:7a:3e:86:d2:a6:de:77:7b:0b:19:f4:b6:d1:a4:00:36:a3:
         1a:f6:d0:a9:74:af:a2:9d:39:cd:b4:1c:54:cd:e3:e7:4e:d2:
         c8:34:cc:27:e6:6c:d9:3e:aa:cf:d0:1a:cb:db:24:70:fd:2d:
         49:ca:85:6b:a2:46:98:82:29:22:87:a2:50:60:60:50:40:2b:
         7d:ad:11:db:da:da:c0:2d:71:a5:5b:c7:f6:38:ad:68:d0:49:
         02:49:91:58:62:4b:ef:ee:66:6d:03:1b:ba:4c:5f:0c:c2:92:
         cd:2c:99:4c:0c:8f:60:54:18:1a:c6:1b:72:36:ec:a1:61:ef:
         56:49:75:ac:28:9a:a9:69:d7:3b:9e:5e:b4:9d:5b:41:1a:e7:
         9a:5b:bd:c9:3c:27:76:42:02:87:f9:9a:62:2b:e3:28:a5:78:
         13:26:57:29:a6:21:e5:85:84:aa:f8:33:b1:dd:7a:a7:b0:37:
         36:d6:d6:0b:50:96:6c:2c:fe:f7:9b:2f:2a:f0:fe:90:94:63:
         6a:d9:e6:a6:fb:49:e1:8d:21:d9:f8:f0:eb:72:d3:23:08:a7:
         42:77:d7:f8
</pre>




## Kafka Digital Certificate
With the Kafka Digital Certificate issued, we're going to craft a ".jks" file to be used by Kafka Cluster.

1. Create the Truststore and Keystore for all brokers<p>
The command creates the "kafka.jks" file with the CA root Certificate. Use "kafkastore" as the keystore password.

<pre>
$ keytool -keystore kafka.jks -alias CARoot -import -file acquaCA.crt
Enter keystore password:  
Re-enter new password: 
Owner: EMAILADDRESS=acquaviva@uol.com.br, CN=acqua.com, OU=Technology, O=Acqua Corp, L=Sao Paulo, ST=Sao Paulo, C=BR
Issuer: EMAILADDRESS=acquaviva@uol.com.br, CN=acqua.com, OU=Technology, O=Acqua Corp, L=Sao Paulo, ST=Sao Paulo, C=BR
Serial number: 84e446db86c063e7
Valid from: Wed Dec 30 13:19:40 BRST 2020 until: Fri Jan 29 13:19:40 BRST 2021
Certificate fingerprints:
	 SHA1: 19:E9:EF:EB:48:DE:67:CF:59:19:93:6C:E3:9D:19:15:8C:EB:48:BA
	 SHA256: 97:B1:E9:03:5E:2B:D6:1A:0A:BA:F9:29:1B:5D:35:73:E0:D8:EB:72:DA:39:33:C2:14:24:86:5F:34:33:60:19
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 1
Trust this certificate? [no]:  yes
Certificate was added to keystore
</pre>

2. Issue a Kafka Private Key
<pre>
$ keytool -keystore kafka.jks -alias kafka -validity 365 -genkey -keyalg RSA -ext SAN=DNS:kafka.com
Enter keystore password:  
What is your first and last name?
  [Unknown]:  Claudio Acquaviva
What is the name of your organizational unit?
  [Unknown]:  Technology
What is the name of your organization?
  [Unknown]:  Kafka Corp
What is the name of your City or Locality?
  [Unknown]:  Sao Paulo
What is the name of your State or Province?
  [Unknown]:  Sao Paulo
What is the two-letter country code for this unit?
  [Unknown]:  BR
Is CN=Claudio Acquaviva, OU=Technology, O=Kafka Corp, L=Sao Paulo, ST=Sao Paulo, C=BR correct?
  [no]:  yes

Generating 2,048 bit RSA key pair and self-signed certificate (SHA256withRSA) with a validity of 365 days
	for: CN=Claudio Acquaviva, OU=Technology, O=Kafka Corp, L=Sao Paulo, ST=Sao Paulo, C=BR
</pre>

3. Issue a CSR (Certificate Signing Request) for the Kafka Digital Certificate<p>
The command creates the "kafka.csr" file. Use "kafkastore" as the keystore password.

<pre>
$ keytool -keystore kafka.jks -alias kafka -certreq -file kafka.csr
</pre>

4. Sign the Kafka broker's Certificate
<pre>
$ openssl x509 -req -CA acquaCA.crt -CAkey acquaCA.key -in kafka.csr -out kafka.crt -days 365 -CAcreateserial
Signature ok
subject=/C=BR/ST=Sao Paulo/L=Sao Paulo/O=Kafka Corp/OU=Technology/CN=Claudio Acquaviva
Getting CA Private Key
</pre>

5. Check the Kafka Certificate
<pre>
$ openssl x509 -in kafka.crt -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 17102241651444505035 (0xed575e2ba13d59cb)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: C=BR, ST=Sao Paulo, L=Sao Paulo, O=Acqua Corp, OU=Technology, CN=acqua.com/emailAddress=acquaviva@uol.com.br
        Validity
            Not Before: Dec 31 12:47:07 2020 GMT
            Not After : Dec 31 12:47:07 2021 GMT
        Subject: C=BR, ST=Sao Paulo, L=Sao Paulo, O=Kafka Corp, OU=Technology, CN=Claudio Acquaviva
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:8f:ee:f1:50:a4:43:53:6c:41:26:f5:10:88:cc:
                    fd:ba:6d:06:07:23:92:2f:39:49:d8:e0:c5:4f:a1:
                    44:9b:7d:2a:f1:c0:35:8c:60:a5:d1:58:91:9b:7e:
                    86:c8:32:49:28:98:03:37:50:2f:c7:76:6d:8d:b8:
                    cc:ad:89:86:46:bf:9e:e4:f1:bb:b2:d7:bd:d2:88:
                    fe:93:d8:8c:99:a2:bf:17:f2:93:b6:56:81:51:c1:
                    30:fe:60:04:62:60:de:07:bb:1f:4c:56:a5:5f:e0:
                    f4:06:35:f3:e4:92:aa:67:4a:ea:c4:49:3c:ea:47:
                    86:bd:9c:59:b4:fe:08:4e:ff:3a:9e:fd:92:07:34:
                    5b:6e:2a:f1:6b:5f:59:3e:ba:08:a6:46:26:56:e5:
                    58:b6:d2:dc:f0:44:ed:9c:cb:6e:6c:70:ee:0f:f2:
                    6c:5e:44:8d:ee:c2:33:e4:01:79:6d:6e:96:69:f5:
                    db:51:75:36:2c:3e:bb:13:6e:99:f2:c1:c3:57:17:
                    9c:94:91:14:94:bb:f7:e2:5e:b3:b8:e3:e7:e2:f1:
                    92:de:4a:e8:07:aa:67:6b:f9:31:8a:39:0c:ac:02:
                    b6:c9:cd:46:44:4e:aa:d5:a7:cb:54:d9:50:7f:a2:
                    04:5f:42:c9:70:18:99:25:8b:6b:e4:a4:27:e4:4d:
                    79:4d
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha1WithRSAEncryption
         a2:14:bf:dc:82:58:03:0a:2a:37:c1:f8:76:98:95:54:e6:31:
         81:9b:64:e1:1a:32:cc:04:91:78:17:40:41:40:0d:f5:2b:0e:
         4d:2c:c4:31:33:fa:01:da:a1:a9:de:9c:dd:61:b7:8d:c6:03:
         7b:3e:62:78:45:39:31:0f:f8:5d:50:66:bf:8e:90:78:05:38:
         2e:bd:65:28:f2:96:9c:7a:f2:b8:9c:93:1d:d5:04:64:ed:81:
         19:4b:c6:91:ff:ba:66:bf:31:d2:1f:cb:ca:aa:34:a5:23:b0:
         27:af:ff:54:80:b7:16:1c:db:7e:01:2e:97:69:f5:7c:1f:d9:
         1e:37:a9:04:e6:57:86:12:5d:16:8d:0b:77:53:1f:1d:7d:a3:
         41:1a:80:c7:76:5d:3e:3a:e1:6a:65:1e:c9:51:12:2a:4b:9d:
         79:59:80:19:ff:93:cb:4b:19:60:a8:b8:19:5b:a1:af:f2:3c:
         8e:ae:4d:a9:70:5f:1f:10:ec:85:a8:10:eb:8e:bf:23:40:9f:
         26:86:e6:84:a9:ad:6e:d4:8c:1b:dc:9c:ae:b0:de:91:ae:20:
         16:bf:26:9f:f7:37:f1:8d:c6:a1:56:12:ab:48:ca:99:0e:5f:
         4c:92:4f:c8:74:26:b9:57:3a:43:2a:cd:d3:fb:19:74:8c:56:
         32:81:99:6b
</pre>


## Kafka with mTLS on
1. Delete Kafka Container
<pre>
docker stop kafka
docker container rm kafka -v
</pre>

2. Start new Container<p>
The new Kafka container has two new settings:<p>
- SSL: it enables the specific 9093 port and requests for SSL Client Authentication<p>
- It sets all specific Kafka parameters related to the .jks file we crafted before.<p>

<pre>
docker run -d --name kafka -p 9092:9092 -p 9093:9093 --hostname kafka --network kong-net --link zookeeper:zookeeper \
-e KAFKA_LISTENERS=PLAINTEXT://:9092,SSL://:9093 \
-e KAFKA_SSL_TRUSTSTORE_LOCATION=/var/kafka.server.truststore.jks \
-e KAFKA_SSL_TRUSTSTORE_PASSWORD=kafkastore \
-e KAFKA_SSL_KEYSTORE_LOCATION=/var/kafka.server.keystore.jks \
-e KAFKA_SSL_KEYSTORE_PASSWORD=kafkastore \
-e KAFKA_KEY_PASSWORD=kafkastore \
-e KAFKA_SSL_CLIENT_AUTH=required \
-v /Users/claudio/kong/tech/Confluent/Plugin/CA/kafka.jks:/var/kafka.server.truststore.jks:ro \
-v /Users/claudio/kong/tech/Confluent/Plugin/CA/kafka.jks:/var/kafka.server.keystore.jks:ro \
confluent/kafka
</pre>

Test the connection
<pre>
openssl s_client -debug -connect localhost:9092 -tls1
openssl s_client -debug -connect localhost:9093 -tls1
</pre>


## Kafka mTLS Upstream plugin with mTLS on
1. Delete the mTLS plugin<p>
We're going to recreate the Kafka mTLS Upstram plugin to turn mTLS on:

<pre>
$ http :8001/plugins
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 879
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 16:54:30 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Admin-Latency: 3
X-Kong-Admin-Request-ID: TgYSBt0MfGcLx4HT8DH5vOFVfxDIbBd8
vary: Origin

{
    "data": [
        {
            "config": {
                "bootstrap_servers": [
                    {
                        "certificate_id": null,
                        "host": "kafka",
                        "port": 9092,
                        "tls_enable": false
                    }
                ],
…….
            "id": "468e6c2f-8c03-4a43-9b1d-dd69cc9ae0e8",
…….
}
</pre>

<pre>
http delete :8001/plugins/468e6c2f-8c03-4a43-9b1d-dd69cc9ae0e8
</pre>


2. Inject the CA Digital Certificate in Kong Enterprise
<pre>
curl -sX POST http://localhost:8001/ca_certificates -F "cert=@./acquaCA.pem"
</pre>


3. Inject the Kong Digital Certificate in Kong Enterprise
<pre>
curl -sX POST http://localhost:8001/certificates \
    -F "cert=@./kong.crt" \
    -F "key=@./kong.key"
</pre>


4. Check both Digital Certificates
<pre>
$ http :8001/ca_certificates
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 1273
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 16:58:42 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Admin-Latency: 2
X-Kong-Admin-Request-ID: VUGhZM2xeOZDQsQ8dYaZFCi5nZaWb6Es
vary: Origin

{
    "data": [
        {
            "cert": "-----BEGIN CERTIFICATE-----\nMIIC1zCCAb+gAwIBAgIJALVbmGbas7WsMA0GCSqGSIb3DQEBBQUAMCIxCzAJBgNV\nBAYTAkJSMRMwEQYDVQQKDApBY3F1YSBDb3JwMB4XDTIwMTIzMDE1MjgzMloXDTMw\nMTIyODE1MjgzMlowIjELMAkGA1UEBhMCQlIxEzARBgNVBAoMCkFjcXVhIENvcnAw\nggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDX7wHj990GOchFFz9bQdJw\nwSkzDM2Mp8TQEZuQ6z5WzXrHOVZGRNlov8MYVs80lXxXQNP//lJmvCMYdd9OihLR\n2qlSL9zQjZ+TsgVPdGbkyFwQYeH8/PqjH3O46SzQumn0Mtlgrk2PFhwAyVXtKZGg\nMwKuHmRXGl9GP6yCeEwc+5IuT/FjcFi6/3tvn5ralLjNLJGiczL6HKcRcsHKf7eI\n3PMofBYCclHRtsly4UoYNlgm7nL7qvlzLSaoRTdJhT7hI8X7H/nQ/20LKORFl/EN\nVgnJoZeAgMP48mLsw/gEfFHUoFXRFLJPMQIWHbTpAF67LKFpFkfDTVJgKg4uZefb\nAgMBAAGjEDAOMAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEFBQADggEBAHdkTCmL\nboUFsTRfT6Pv0oRJ9ZVqojB7Hi2tNY88Ftvu7tvy49wmMTmDNJwYjDF4jvp0jDh7\ndZMfirjcGq64dakRnMRbt96V+19UiKCQHfnJFe505viOmAiMR4pCk/G8ZV0XFRia\nrqTb5QKQmPSo/6b2iaoRiaXPQL16ZuyxMlt+b5/0YSBrw4I3ZpsohMHWwQGYgdRd\nTP8zL+i2uvaON65zzsbCaN2T2ikWmFdRcgeitYPyRNMdINYFvlYOj8aRBuky/p01\nkvvxzreCmrDeAeO6i/eZN+LmVAJ7jSbmOH5rWnsxYOAZDeZQ1sK52DDWfjQ8Zxhh\nRwcpgQqlZhAXunI=\n-----END CERTIFICATE-----\n",
            "cert_digest": "52d752773c4321d21292a3392b987ba859bffe5f32604373da7255de52df358c",
            "created_at": 1609347472,
            "id": "c8b462f2-3ecd-4737-bec6-fc6bc3274fed",
            "tags": null
        }
    ],
    "next": null
}
</pre>

<pre>
$ http :8001/certificates
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 3216
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 16:59:08 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Admin-Latency: 3
X-Kong-Admin-Request-ID: mXPvqolim0x5FCDbsk3RaNjXK93w9BBj
vary: Origin

{
    "data": [
        {
            "cert": "-----BEGIN CERTIFICATE-----\nMIIDrDCCApQCCQDtV14roT1ZyDANBgkqhkiG9w0BAQUFADCBmDELMAkGA1UEBhMC\nQlIxEjAQBgNVBAgMCVNhbyBQYXVsbzESMBAGA1UEBwwJU2FvIFBhdWxvMRMwEQYD\nVQQKDApBY3F1YSBDb3JwMRMwEQYDVQQLDApUZWNobm9sb2d5MRIwEAYDVQQDDAlh\nY3F1YS5jb20xIzAhBgkqhkiG9w0BCQEWFGFjcXVhdml2YUB1b2wuY29tLmJyMB4X\nDTIwMTIzMDE1NDEzMVoXDTMwMTIyODE1NDEzMVowgZYxCzAJBgNVBAYTAkJSMRIw\nEAYDVQQIDAlTYW8gUGF1bG8xEjAQBgNVBAcMCVNhbyBQYXVsbzESMBAGA1UECgwJ\nS29uZyBDb3JwMRMwEQYDVQQLDApUZWNobm9sb2d5MREwDwYDVQQDDAhrb25nLmNv\nbTEjMCEGCSqGSIb3DQEJARYUYWNxdWF2aXZhQHVvbC5jb20uYnIwggEiMA0GCSqG\nSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCpK3vqwzLm/vV2mbtd+3IdZKZvXwTG+yM0\nZWbM2wYkx7gULpRDrF4vtvg35ZzHqHn4YzhsGg4OO7SH7S02G2TqSM+qiPm+eE34\nh2gv13smxM7gut+fBsgTEVcYHMDv4q2Q3eOE9GlKwUdsHwnyAzRwVpSmMhp4ifn3\nZK8dCMG3CtKj6CqQpZFxKbfAAQfnkWQYzLIWRY707YRS0VFpTR0ryIqQtbyfYnGf\negKosepcsjBdKzKWWVoaWtUTnhDXCSk2vagx7OmkKooAMGJ6pr4OB2WU/n5ADBIC\n5cgjZz/+TsxyIYwceaew7OLA3REJ4vZrXDjbWHBUN9T3w79JI7e1AgMBAAEwDQYJ\nKoZIhvcNAQEFBQADggEBABS+5y180syWRFIO+jV+YZVBVzqzwW8VVIajTOe62PVw\nhEDT+1x6PobSpt53ewsZ9LbRpAA2oxr20Kl0r6KdOc20HFTN4+dO0sg0zCfmbNk+\nqs/QGsvbJHD9LUnKhWuiRpiCKSKHolBgYFBAK32tEdva2sAtcaVbx/Y4rWjQSQJJ\nkVhiS+/uZm0DG7pMXwzCks0smUwMj2BUGBrGG3I27KFh71ZJdawomqlp1zueXrSd\nW0Ea55pbvck8J3ZCAof5mmIr4yileBMmVymmIeWFhKr4M7HdeqewNzbW1gtQlmws\n/vebLyrw/pCUY2rZ5qb7SeGNIdn48Oty0yMIp0J31/g=\n-----END CERTIFICATE-----\n",
            "created_at": 1609347481,
            "id": "fa277570-141f-4a64-ba55-daff0823eadb",
            "key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqSt76sMy5v71dpm7XftyHWSmb18ExvsjNGVmzNsGJMe4FC6U\nQ6xeL7b4N+Wcx6h5+GM4bBoODju0h+0tNhtk6kjPqoj5vnhN+IdoL9d7JsTO4Lrf\nnwbIExFXGBzA7+KtkN3jhPRpSsFHbB8J8gM0cFaUpjIaeIn592SvHQjBtwrSo+gq\nkKWRcSm3wAEH55FkGMyyFkWO9O2EUtFRaU0dK8iKkLW8n2Jxn3oCqLHqXLIwXSsy\nlllaGlrVE54Q1wkpNr2oMezppCqKADBieqa+DgdllP5+QAwSAuXII2c//k7MciGM\nHHmnsOziwN0RCeL2a1w421hwVDfU98O/SSO3tQIDAQABAoIBAQCkWxLxauQxeNOS\nfpmDHaAo3ni1C2Pgzm3NohbWQJUfdsppETgK55Q6V1GhPPutHwohQIS4wjeVrHwg\n81VRlBvfYw4faST64HcgVq3qjTeg2uUDgYtxPW102Qv86TKp3VkzveAmdC836cAy\nU5WeA28XFYcmUNdW9PZeXPulAbTy14OUU7XzqsypZhctKP76RlRmMubAYTjrQgYo\nFKLRid4oGzlZj5eJDO46kFgATpMJdAR3Hd38So1iRkVvi3Mid8g8FnmH48+kHn1O\ncUuKZhBNTopFSWjiBqFEX3a8btMmBga3kUbBozFRZxyTd1LIWvnv933e32o4cKso\nt9a3EsGhAoGBAN+btRrVf/PMBQR6xt5SJB+idpMOvYWa/iSlJsLfpGfsj70LIz0h\ndywAa2pBM7bDc8ndFW1Aiy/0hsGRBtpXJfrpPePAzg35k/sCJgqOPn8Mc3M8oYeU\nrw7XQRpbSyOFPIRLRrYm3KoRDtUCydjJtJXK6lyOIEW+7w01sKSgsSQfAoGBAMGs\n+5mfzcW0osCCXzMfninPwer5N9gnOz6zcdaonWtbO17CS7LjgoOSpanNho0Mhr9b\n9Fz71o2Mxf06ibFgrQEgstrNkq+BGl1HH+LL4dGA5BpoPN7SSOVn6tFfW9TawFbD\nqJGhLDV58X1HoOF4o7ShJYrFNIb+Snh3V/iqBImrAoGANI3ZC9x//SHHUB03Hkt5\n+AFsEvYU7xDViHFUYdrEPjSoN8slVhnGc44JsOhwKhVX4mrWvV29GOFExru6O5jd\n8VHeXOgUxc4RzJ3dqP9zitK3U689W6tDVZ6by4EHcOrApWs3zFnn5QSrUr8cB5qo\nmcgeOvCgfyP39UfYI2ktGQsCgYBwf76V+dFZKhfvossRsyf4OYn2p1Tc5czwGuPh\nQIhQN+pAnLPD8Yt6SdCY1Z12iPQsa4mCCXcTOdY3xaz9r55OrWO23Pp7n45k6E+J\nOcyuGSRmgm35METPnJE1lSKOfZKD05szHF/FoFO55cV5ss3EumZIOUzNrSAs4YXk\nFz4TiQKBgQDLW/15YTwuTzpLyknZtwysbvnPpDiKHCF5SjzKW3k7hM7kr05YmdCm\naYfH3FMXoZ5kc+SqVUynm9PQkjgDZCU4mxAsSsOlthRYtQuNh8ElmFW84AQ1UmqK\nF66zokOl5Wcr5hzmtuAniy1iR1kDZjgOLIMYirQitsrUe4sWuf5vBw==\n-----END RSA PRIVATE KEY-----\n",
            "snis": [],
            "tags": null
        }
    ],
    "next": null
}
</pre>



5. Enable the Kafka mTLS Upstream plugin to the Route with mTLS on<p>
The new Kafka plugin settings include:<p>
- The 9093 port defined by the Kafka Cluster for SSL connections<p>
- Uses the Kong Enterprise Digital Certificate id injected previously.<p>
- All other settings remain the same

<pre>
curl -X POST http://localhost:8001/routes/httpbinroute/plugins \
    --data "name=kafka-upstream" \
    --data "config.bootstrap_servers[1].host=kafka" \
    --data "config.bootstrap_servers[1].port=9093" \
    --data "config.bootstrap_servers[1].tls_enable=true" \
    --data "config.bootstrap_servers[1].certificate_id=fa277570-141f-4a64-ba55-daff0823eadb" \
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

6. Check the plugin
<pre>
$ http :8001/plugins
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 912
Content-Type: application/json; charset=utf-8
Date: Thu, 31 Dec 2020 14:23:03 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Admin-Latency: 5
X-Kong-Admin-Request-ID: iRU7dbbEbjWhiQQiCBI5wjYg8Was5qJZ
vary: Origin

{
    "data": [
        {
            "config": {
                "bootstrap_servers": [
                    {
                        "certificate_id": "fa277570-141f-4a64-ba55-daff0823eadb",
                        "host": "kafka",
                        "port": 9093,
                        "tls_enable": true
                    }
                ],
                "forward_body": false,
                "forward_headers": true,
                "forward_method": false,
                "forward_uri": false,
                "keepalive": 60000,
                "producer_async": true,
                "producer_async_buffering_limits_messages_in_memory": 50000,
                "producer_async_flush_timeout": 1000,
                "producer_request_acks": 1,
                "producer_request_limits_bytes_per_request": 1048576,
                "producer_request_limits_messages_per_request": 200,
                "producer_request_retries_backoff_timeout": 100,
                "producer_request_retries_max_attempts": 10,
                "producer_request_timeout": 2000,
                "timeout": 10000,
                "topic": "test"
            },
            "consumer": null,
            "created_at": 1609419198,
            "enabled": true,
            "id": "9ccc36e0-661e-41f9-801f-fb96e2be03e3",
            "name": "kafka-upstream",
            "protocols": [
                "grpc",
                "grpcs",
                "http",
                "https"
            ],
            "route": {
                "id": "9e784488-29e2-4dfb-9707-289db87536f3"
            },
            "service": null,
            "tags": null
        }
    ],
    "next": null
}
</pre>


7. Consume the Route and check the Kafka Topic
<pre>
$ http :8000/httpbin/get bbb:555
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 26
Content-Type: application/json; charset=utf-8
Date: Wed, 30 Dec 2020 14:59:20 GMT
Server: kong/2.2.0.0-enterprise-edition
X-Kong-Response-Latency: 30

{
    "message": "message sent"
}
</pre>

The consumer should show the new message
<pre>
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
{"headers":{"host":"localhost:8000","accept-encoding":"gzip, deflate","user-agent":"HTTPie\/2.3.0","accept":"*\/*","bbb":"555","connection":"keep-alive"}}
</pre>
