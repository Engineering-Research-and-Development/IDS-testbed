# IDSA Reference Testbed Installation (CA-DAPS-TC-MDB)

## Index
- [Execution modes](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#execution-modes)
    - [Hardware Requirements](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#hardware-requirements)
- [Target View: Preconfigured testbed](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#target-view-preconfigured-testbed)
- [Target View: Manual testbed set up](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#target-view-manual-testbed-set-up)
    - [Certificate Authority](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#certificate-authority)
    - [DAPS](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#daps)
    - [TRUE Connector](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#true-connector)
    - [Metadata Broker](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#metadata-broker)
- [Stop and delete testbed set up](https://github.com/Engineering-Research-and-Development/true-connector-mvds/blob/master/InstallationGuide.md#stop-and-delete-testbed-set-up)

The installation and configuration process is explained below for each of the components. To further support this document, the links to the official installation guides will be linked.

# Execution modes
You may either run the preconfigured testbed offered in this repo or follow the instructions for the manual setup below to set it up on your own and possibly adjust it to your needs.

## Hardware Requirements

In this section the minimum requirements required for operating the IDS-testbed are detailed.

The current minimum requirements for the IDS-Testbed are:
- 4 GB RAM (however 8GB RAM is recommended)
- 50 GB storage

It is recommended to use 64bit quad core processor to provide enough processing power for all docker containers.

Take into account that if more components are included at the IDS-testbed or a huge amount of data is uploaded it is possible to run out of disk free space. In this cases it is recommended to provide more free disk storage.

# Preconfigured testbed
The Testbed is already preconfigured out-of-the-box for testing purposes and you can start it by executing in the root folder where the docker-compose.yml is located:

```
docker-compose up -d
```

To see the log lines:

```
docker-compose logs -f
```

Follow this set up of the Testbed to configure it to your needs.

# CERTIFICATE AUTHORITY
Move to right directory, and make the files executable in CertificateAuthority.

The official documentation will cover the scope, dependencies and usage of the component.

Official documentation: [CertificateAuthority/README.md](./CertificateAuthority/README.md)

The preconfigured setup includes certificates for:
* a root CA called "ReferenceTestbedCA"
* a subCA called "ReferenceTestbedSubCA" and
* certificates for devices called "testbed1", ... , "testbed4"

## Continue here after the official documentation has been followed

The Certificate Authority provides {CERT_FILENAME}.crt and {CERT_FILENAME}.key formats. Keep in mind that other formats will be required for the different components. Those have to be created.

Now convert the generated certificates in `data/cert` using openssl to then use in the Connector and DAPS.

```
## navigate to the following directory data/cert
cd data/cert
ls
```
The output should look similar to
```
{CERT_FILENAME}.crt  {CERT_FILENAME}.key
```
Obtain a `.p12` file format from the current `.crt` and `.key` formats:
```
## .crt + .key -> .p12
openssl pkcs12 -export -out {CERT_FILENAME}.p12 -inkey {CERT_FILENAME}.key -in {CERT_FILENAME}.crt -passout pass:password
## .p12 -> .cert
openssl pkcs12 -in {CERT_FILENAME}.p12 -out {CERT_FILENAME}.cert -nokeys -nodes -passin pass:password

```

You should now have two additional files in data/cert

```
{CERT_FILENAME}.cert  {CERT_FILENAME}.crt  {CERT_FILENAME}.key  {CERT_FILENAME}.p12
```

The certificate chain (CA, SubCA, Certs) has been created and the user should be able to create as many certificates as they need for their environment.

# DAPS

The official documentation of the Omejdn DAPS is here: https://github.com/International-Data-Spaces-Association/omejdn-daps

## Adding the keys to the DAPS

Every client that wants to use the local Omejdn DAPS must place their `{CERTFILE}.cert` file in the `keys` directory.

The directory can be found in

```
DAPS/keys
```

Add the certificate provided by the local CA, newly created by the local CA or provided by Fraunhofer AISEC. Place the certificate at the folder `DAPS/keys/omejdn/` with name `omejdn.key` to avoid dependency issues later on.

## Adding the clients to the DAPS

**Note:** The user must execute the `register_connector.sh` file in order to add the client to the Omejdn DAPS. Once executed, the certificate will be included in the DAPS's list of clients.

To execute the script

```
cd DAPS
./register_connector.sh {CERT_FILENAME} 
```

It could look something like this
```
./register_connector.sh testbed1
```

The certificate will be added to the list of DAPS's clients. You can check it at the file `DAPS/config/clients.yml`

## Required changes in the configuration

Change the configuration file `.env` with your favorite editor, e.g. `nano`.

```
nano .env
```

**Note** The file could be hidden. Select the option `show hidden files` and it should be placed at IDS-testbed root directory.

Replace the following lines with the necessary configuration. It could look something like this

```
COMPOSE_PROJECT_NAME=testbed
OMEJDN_ENVIRONMENT="production"
OMEJDN_PROTOCOL="https"
OMEJDN_VERSION="1.6.0"
OMEJDN_DOMAIN="omejdn"
OMEJDN_PATH="/auth"

ADMIN_USERNAME="admin"
ADMIN_PASSWORD="password"

TLS_KEY="${PWD}/DAPS/keys/TLS/daps.key"
TLS_CERT="${PWD}/DAPS/keys/TLS/daps.cert"
```

Configure the `docker-compose.yml` file with your configuration. Then run the Omejdn DAPS server.

The `docker-compose.yml` could look something like this

```
services

  omejdn:
    image: nginx:1.21.6
    container_name: omejdn
    ports:
      - 80:80
      - 443:443      
    environment:
      - OMEJDN_DOMAIN=${OMEJDN_DOMAIN}
      - OMEJDN_PATH=${OMEJDN_PATH}
      - UI_PATH=${UI_PATH}
    volumes:
      - ./DAPS/nginx.conf:/etc/nginx/templates/default.conf.template
      - ./DAPS/keys/TLS/daps.cert:/etc/nginx/daps.cert
      - ./DAPS/keys/TLS/daps.key:/etc/nginx/daps.key
    networks:
      - local

  omejdn-server:
    image: ghcr.io/fraunhofer-aisec/omejdn-server:${OMEJDN_VERSION}
    container_name: omejdn-server
    environment:
      - OMEJDN_ISSUER=${OMEJDN_ISSUER}
      - OMEJDN_FRONT_URL=${OMEJDN_ISSUER}
      - OMEJDN_OPENID=true
      - OMEJDN_ENVIRONMENT=${OMEJDN_ENVIRONMENT}
      - OMEJDN_ACCEPT_AUDIENCE=idsc:IDS_CONNECTORS_ALL
      - OMEJDN_DEFAULT_AUDIENCE=idsc:IDS_CONNECTORS_ALL
      - OMEJDN_ADMIN=${ADMIN_USERNAME}:${ADMIN_PASSWORD}
    volumes:
      - ./DAPS/config:/opt/config
      - ./DAPS/keys:/opt/keys
    networks:
      - local
      
networks:
  local:
    driver: bridge
```

Place the local CA created certificate at the folder `DAPS/keys/TLS/` and name it as `daps.crt` and `daps.key` to match the above mentioned `docker-compose.yml` file configuration.

# TRUE CONNECTOR

The testbed will have two built-in Connectors which are already preconfigured and ready out-of-the-box. To tailor the TRUE Connector setup to your needs follow the next steps.

For a deep dive in to the TRUE Connector see [here](https://github.com/Engineering-Research-and-Development/true-connector/releases/tag/v1.0.0)

## Generate DAPS certificate using Omejdn DAPS

This step is not mandatory, since Testbed comes with some generated certificates which can be used temporarily.

If you want to create new certificate, please follow [instructions](https://github.com/Engineering-Research-and-Development/true-connector-mvds/tree/master/CertificateAuthority) provided by Testbed, on how to achieve this.


## DAPS certificate

You can use certificate already provided in Testbed project, in following location - *CertificateAuthority\data\cert* . We need to make some small adjustments, to generate valid p12 file. For this purpose, we need to copy *ReferenceTestbedCA.crt* file from ca folder (one level above) and execute following command:

```
openssl pkcs12 -export -out testbed3.p12 -inkey testbed3.key -in testbed3.crt -certfile ReferenceTestbedCA.crt
```
As Export password insert ***password***, and confirm it.


This will generate valid testbed3.p12 file. Copy this file to certificate folder of the connector.


## TRUE Connector properties

Make sure that following properties are configured for Testbed environment:

**.env** 

```
### PROVIDER Configuration
PROVIDER_DAPS_KEYSTORE_NAME=testbed1.p12
PROVIDER_DAPS_KEYSTORE_PASSWORD=password
PROVIDER_DAPS_KEYSTORE_ALIAS=1

PROVIDER_MULTIPART_EDGE=form

CONSUMER_DAPS_KEYSTORE_NAME=testbed2.p12
CONSUMER_DAPS_KEYSTORE_PASSWORD=password
CONSUMER_DAPS_KEYSTORE_ALIAS=1
```

(for the moment, we will use same DAPS certificate for both Provider and Consumer)

**application.properties** for both consumer and provider

```
application.isEnabledDapsInteraction=true

#Omejdn 
application.dapsUrl=http://omejdn/auth/token
application.dapsJWKSUrl=http://omejdn/auth/jwks.json

application.isEnabledUsageControl=true
application.usageControlVersion=platoon

```

### Enabling hostname validation

In order to enable this functionality, following changes are required:

**.env** file 

```
DISABLE_SSL_VALIDATION=false

TRUSTORE_NAME=truststoreEcc.jks
TRUSTORE_PASSWORD=allpassword

```

This will load certificates from truststore and when making https call towards target system, certificate will be validated against truststore, and if certificate is present and valid, call will be successful, otherwise you will get PKIX exception.

### Export TRUE Connector certificate

Open *ssl-server.jks* file from TRUEConnector/ecc_cert folder using KeyStore Explorer and export certificate (right click on entry name):

![Certificate_1](Export_TC_Certificate_1.jpg "Export TRUE Connector Certificate")

and provide location where to save exported certificate. It will be needed in next step, to update the external connector truststore.

![Certificate_2](Export_TC_Certificate_2.jpg "Export TRUE Connector Certificate 2")

Or you can use already extracted *execution_core_container.cer* file from **ecc_cert** folder.

### Updating external connector truststore

Open truststore file *truststore.p12* using KeyStore Explorer and import TRUE Connector certificate, so that the external connector can make https calls towards TRUE Connector provider

![Truststore](Import_TC_Certificate.jpg "Import TRUE Connector Certificate")

and provide alias *true-connector*

![Truststore Alias](Import_TC_Certificate_alias.jpg "Import TRUE Connector Certificate alias")

This will be used when external connector makes https request towards ecc-provider, to check hostname with imported certificate.


## Testbed interaction

### TRUE Connector as consumer

To perform contract negotiation with DataSpaceConnector and get an artifact. For this purpose, you can use ![TC as consumer](TC_Consumer.png "TC as consumer")

You can execute the whole folder, same as before, or execute each request in that order.

If everything was successful, it should look like the following:

![TC as consumer result](TC_Consumer_result.png "TC as consumer result")


### TRUE Connector as provider

TRUE Connector comes with predefined Self Description document. You can get more information about it by expecting the document itself on URL:

```
https://localhost:8090/

```

Same goes as before, execute each request or the whole TC as provider folder:

![TC as provider](TC_Provider.png "TC as provider")

### Broker interaction

The interaction with MetadataBroker can be checked with the requests from the Broker interaction folder in the same way as the steps from above.

![Broker interaction](Broker_interaction.png "Broker interaction")

### Self Description API

And finally, the Self Description API, for modifying the Self Description document can be accessed via the 

![Self Description API](Self_Description_API.png "Self Description API")

# METADATA BROKER

## Component Documentation
The official documentation will cover the pre-requisites, installation and deployment of the component.

Official documentation: https://github.com/International-Data-Spaces-Association/metadata-broker-open-core

## Continue here after reading the official documentation

Download the component from the official repository

```
cd IDS-testbed
git clone -b 5.0.3 https://github.com/International-Data-Spaces-Association/metadata-broker-open-core.git
```

Use the downloaded component to build the broker-core image.

## Changes to the application.properties file
Use nano or your most favourite editor.
```
nano broker-core/src/main/resources/application.properties
```
### DAPS
This will make use of the locally installed DAPS.

```
# DAPS
# daps.url=https://daps.aisec.fraunhofer.de
daps.url=https://omejdn/auth/token
daps.validateIncoming=true
```

### Security-related
Add the local DAPS to the trusted hosts

```
# Security-related
...
jwks.trustedHosts=daps.aisec.fraunhofer.de,omejdn
ssl.certificatePath=/etc/cert/server.crt
ssl.javakeystore=/etc/cert/isstbroker-keystore.jks
```

## Changes to the component's keystore
At the folder `broker-core/src/main/resources/` add the certificate provided by the local CA, newly created by the local CA or provided by Fraunhofer AISEC. If it is NOT provided by the local CA, make sure it is correctly added to the local DAPS.

```
keytool -importkeystore -srckeystore {SRCKEYSTORE} -srcstoretype {STORETYPE} -srcstorepass {SRCSTOREPASS} -destkeystore {DESTKEYSTORE} -deststoretype {DESTSTORETYPE} -deststorepass {DESTSTOREPASS}
```
It could look something like this

```
keytool -importkeystore -srckeystore testbed3.p12 -srcstoretype pkcs12 -srcstorepass password -destkeystore isstbroker-keystore.jks -deststoretype jks -deststorepass password
```

Expected outcome:
```
"Import command completed:  1 entries successfully imported, 0 entries failed or cancelled"
```

To check the content of the created keystore, use the following command:

```
keytool -v -list -keystore {KEYSTORE}
```

It could look something like this

```
keytool -v -list -keystore isstbroker-keystore.jks
```

## Build the broker-core image

Go to the main directory and build the project with `maven`:

```
cd metadata-broker-open-core
mvn clean package
```

This will create a `.jar` file in `broker-core/target` that will have to be copied into `docker/broker-core`.

```
cp broker-core/target/broker-core-5.0.3.jar docker/broker-core
```

Once the file is copied, move to the `docker/broker-core` directory and place there the TLS certificate that corresponds to the DAPS. For the IDS-testbed it is located at `DAPS/keys/TLS/daps.cert` and use the following command to change the certificate format to `daps.crt`

```
openssl x509 -inform PEM -in daps.cert -out daps.crt
```

Then build the `core` image locally using the following command.

```
cd docker/broker-core
docker build -t registry.gitlab.cc-asp.fraunhofer.de/eis-ids/broker-open/core:5.0.3 .
```

## Adding the TLS certificates

At the `IDS-testbed/MetadataBroker/` folder place the TLS certificates created by the local CA together with the keystore.
* `server.crt`
* `server.key`
* `isstbroker-keystore.jks`

## Usage

Take the content from the file `metadata-broker-open-core/docker/composefiles/broker-localhost/docker-compose.yml` and copy it at your docker-compose.yml file. Use nano or your most favourite editor.
```
nano docker-compose.yml
```

Use the TLS certificates and ensure the container names are consistent with other dependencies by adding `container_name:`.

If port 443 is already in use, the `reverseproxy` container will exit with code 1. Follow the steps in the next block to get around this:

```
services:
  broker-reverseproxy:
    image: registry.gitlab.cc-asp.fraunhofer.de/eis-ids/broker-open/reverseproxy
    container_name: broker-reverseproxy
    volumes:
      - ./MetadataBroker/server.crt:/etc/cert/server.crt
      - ./MetadataBroker/server.key:/etc/cert/server.key
    ports:
      - "443:443" # Change to a port of your choosing if taken: "{PORT}:443"
      - "80:80" # Change to a port of your choosing if taken: "{PORT}:80"
    networks:
      - local

  broker-core:
    image: registry.gitlab.cc-asp.fraunhofer.de/eis-ids/broker-open/core:5.0.3
    container_name: broker-core
    volumes:
      - ./MetadataBroker/isstbroker-keystore.jks:/etc/cert/isstbroker-keystore.jks
    environment:
      - SPARQL_ENDPOINT=http://broker-fuseki:3030/connectorData
      - ELASTICSEARCH_HOSTNAME=broker-elasticsearch
      - SHACL_VALIDATION=true
      - DAPS_VALIDATE_INCOMING=true
      - COMPONENT_URI=https://localhost/
      - COMPONENT_CATALOGURI=https://localhost/connectors/
      - DAPS_URL=https://omejdn/auth/token
    expose:
      - "8080" 
    networks:
      - local

  broker-fuseki:
    image: registry.gitlab.cc-asp.fraunhofer.de/eis-ids/broker-open/fuseki
    container_name: broker-fuseki
    volumes:
      - broker-fuseki:/fuseki
    expose:
      - "3030"
    networks:
      - local
      
volumes:
  broker-fuseki:

networks:
  local:
    driver: bridge
```

Go to the compose file and build the Metadata Broker

```
docker-compose up
```

## Stopping the testbed

To stop the testbed just use the following command:

```
docker-compose down
```