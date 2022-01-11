# Connecting to an Oracle Database outside the Mesh

This example shows how to communicate with an Oracle Database which is external to the Mesh.

## Creating the Java microservice container
~~~bash
cd src/java/wallet-secret-sample 
mvn clean install
podman build -t <container-tag> target  # container-tag is quay.io/ugiordan/oracledb-health-check in the yaml 
podman push <container-tag>
~~~

## Configuration and deploy
Export the Oracle DB environment variables:
~~~bash
export ORACLEDB_HOST="<hostname>"
export ORACLEDB_IP="<ip>"
export ORACLEDB_TCP_PORT="<tcp_port>"
export ORACLEDB_TCPS_PORT="<tcps_port>"
~~~

Exporting the environment variables to configure the Java microservice container
(or manually edit the [oracle-db-tcp.yaml](platform/ocp/oracle-db-tcp.yaml) file):
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@<database>"
export ORACLEDB_USERNAME="<username>"
export ORACLEDB_PASSWORD="<username>"
~~~

The container can also be configured to retrieve the environment variables from a secret. \
To create it you can use the following command as an example:

~~~bash
# Creating the secret exposing the evironment variables for the Java microservice
oc create secret generic oracle-jdbc-env \
  --from-literal=url=jdbc:oracle:thin:@<database> \
  --from-literal=user=<username> \
  --from-literal=password=<username>
~~~

### Oracle DB deployment with TCP connection

#### Create the deployment: 
-  using the exported environment variables:
   ~~~bash
   envsubst < oracle-db-tcp.yaml | oc create -f -
   ~~~

- using the secret that stores the environment variables:
  ~~~bash
  oc patch -f oracle-db-tcp.yaml --type merge --patch "$(cat patch-file-secret-env.yaml)" --local=true -oyaml | oc create -f -
  ~~~
  
The following is an example URL for connecting to Oracle DB using TCP:
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@tcp://$ORACLEDB_HOST:$ORACLEDB_TCP_PORT/<service-name>"
~~~
<!-- 
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=$$ORACLEDB_HOST)(PORT=$ORACLEDB_TCP_PORT))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=<service-name>)))" 
~~~
-->

### Oracle DB deployment with TCPS connection and Oracle Wallet
Create the secret using the wallet downloaded from the Oracle Console:

~~~bash
oc create secret generic oracledb-wallet --from-file=<path-to-wallets-unzipped-folder>
~~~

Export the **ORACLEDB_WALLET_LOCATION** variable (or manually edit the [oracle-db-tcps-wallet.yaml](platform/ocp/oracle-db-tcps-wallet.yaml) file) 
which specifies the *mountPath* where the Java microservice application can find the SSO Wallet file (*cwallet.sso*) required to start the SSL/TLS connection. 

~~~bash
export ORACLEDB_WALLET_LOCATION="<wallet_location>"
~~~

Then *mountPath* (i.e., the **ORACLEDB_WALLET_LOCATION**) must coincide with the one selected for the **wallet_location** variable (used to set the *oracle.net.wallet_location* variable) specified in the Oracle URL string. \
The following is an example URL for connecting to Oracle DB using TCPS and wallet:
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@tcps://$ORACLEDB_HOST:$ORACLEDB_TCPS_PORT/<service-name>?wallet_location=$ORACLEDB_WALLET_LOCATION"
~~~
<!--
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=$ORACLEDB_HOST)(PORT=$ORACLEDB_TCPS_PORT))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=<service-name>)))?WALLET_LOCATION=$ORACLEDB_WALLET_LOCATION"
~~~
-->

It is also possible to simplify the Oracle URL string connection by adding the *tnsnames.ora* and *sqlnet.ora* files to the **oracle-db-wallet**.

The following is an example of *tnsnames.ora*, or TNS (Transparent Network Substrate) file. It is a text file used to configure a client-side connection to the Oracle database. \
You should replace **<SERVER_ADDRESS>** and the **<TCPS_PORT>** with the *IP Address* or *FQDN* and the *TCPS* port of the server hosting your database, respectively. \
You also need to replace the **<SERVICE_NAME>** with the name that uniquely identifies your instance/database.

~~~
SERVER =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCPS)(HOST = <SERVER_ADDRESS>)(PORT = <TCPS_PORT>))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = <SERVICE_NAME>)
    )
  )
~~~

To add the encryption options to the client you need the *sqlnet.ora*. \
As we are only concerned with enabling encrypted communication and not authentication, we will set **SSL_CLIENT_AUTHENTICATION** to **FALSE**.

~~~
SSL_CLIENT_AUTHENTICATION = FALSE
WALLET_LOCATION =
  (SOURCE =
    (METHOD = FILE)
    (METHOD_DATA =
      (DIRECTORY = $ORACLEDB_WALLET_LOCATION)
    )
  )
~~~

Using the *tnsnames.ora* and *sqlnet.ora* files, the URL connection string will be:
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@<SERVICE_NAME>"
~~~

#### Create the deployment:
- using the exported environment variables:
   ~~~bash
   envsubst < oracle-db-tcps-wallet.yaml | oc create -f -
   ~~~

- using the secret that stores the environment variables:
  ~~~bash
  oc patch -f oracle-db-tcps-wallet.yaml --type merge --patch "$(cat patch-file-secret-env.yaml)" --local=true -oyaml | oc create -f -
  ~~~

## Service Mesh configuration
Apply one of the following configurations to consume the Oracle DB service by the in-mesh Java microservice application:

- Control TCP egress traffic without a gateway:
  ~~~
  envsubst < oracle-db-external-se.yaml | oc create -f -
  ~~~

- Direct TCP Egress traffic through an egress gateway:

  ~~~bash
  # Export the istio egress gateway port:
  export EGRESS_GATEWAY_ORACLEDB_PORT=<istio_egress_gateway_port>

  envsubst < oracle-db-mesh-with-egressgateway.yaml | oc create -f -
  ~~~

## Usage
After successsful installation you can validate first connectivity through the Pod's log:

~~~bash
oc logs -l app=oracledb-health-check
'Connecting to jdbc:oracle:thin:@tcps://node-0.oracledb.lab.rdu2.cee.redhat.com:2484/test?wallet_location=/app/wallet/'
'Retrieving connections: true'
'Database version: 12.1'
~~~

And you can use the Pod's http listener to validate connectivity:
1. Determining the ingress IP and ports:
    ~~~
    export INGRESS_HOST=$(oc get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
    export INGRESS_PORT=$(oc -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    ~~~
   
2. Configuring ingress using an Istio gateway:
    ~~~bash
    oc apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: oracledb-health-check-gateway
    spec:
      selector:
        istio: ingressgateway 
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "oracledb-health-check.example.com"
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: oracledb-health-check
    spec:
      hosts:
      - "oracledb-health-check.example.com"
      gateways:
      - oracledb-health-check-gateway
      http:
      - match:
        - uri:
            prefix: /
        route:
        - destination:
            port:
              number: 8080
            host: oracledb-health-check
    EOF
    ~~~
   
3. Access the *oracledb-health-check* service using curl
    ~~~bash
    curl -X GET -s -w "\n" -HHost:oracledb-health-check.example.com "http://$INGRESS_HOST:$INGRESS_PORT"
    {"database-version": "12.1", "database-sysdate": "2022-01-11 10:39:52"}
    ~~~