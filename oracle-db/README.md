# Consuming an External OracleDB Service using Openshift Service Mesh

This example shows how to consume external OracleDB services. 

You deploy a simple Java client microservice application which communicate with the external OracleDB using JDBC.

This microservice can also be used to validate connectivity with the database by looking at its log or issuing http requests.

## Creating the Oracle Database
According to your testing environment follow one of this two documentations to install the Oracle Database (12c R1):
- [Amazon Web Services (AWS) : Installation of Oracle on EC2](https://oracle-base.com/articles/vm/aws-ec2-installation-of-oracle)
- [Oracle Database 12c Release 1 (12.1) Installation On Oracle Linux 7 (OL7)](https://oracle-base.com/articles/12c/oracle-db-12cr1-installation-on-oracle-linux-7)

Database Creation (DBCA):
~~~
dbca -silent -createDatabase \
 -templateName General_Purpose.dbc \
 -gdbname <database_name> -sid <database_name> -responseFile NO_VALUE \
 -characterSet AL32UTF8 \
 -sysPassword <password> \
 -systemPassword <password> \
 -createAsContainerDatabase true \
 -numberOfPDBs 1 \
 -pdbName test \
 -pdbAdminPassword <password> \
 -databaseType MULTIPURPOSE \
 -automaticMemoryManagement false \
 -storageType FS \
 -datafileDestination "/u01/app/oracle/oradata/" \
 -redoLogFileSize 50 \
 -emConfiguration NONE \
 -ignorePreReqs
~~~

Create a User and grant Permissions:
~~~
export ORACLE_SID=test
alter session set "_ORACLE_SCRIPT"=true; 
create user <username> identified by <password>;
create role <username>_role;
GRANT ALL PRIVILEGES TO <username>_role; 
GRANT <username>_role TO <username>;
~~~

Test connection:
~~~
sqlplus <username>/<password>

SQL> SELECT SYSDATE from dual;
SYSDATE
---------
12-JAN-22
~~~

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
export ORACLEDB_PORT="<tcp_port/tcps_port>"
~~~

Exporting the environment variables to configure the Java microservice container
(or manually edit the [oracle-db-tcp.yaml](platform/ocp/oracle-db-tcp.yaml) file):
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@<database>"
export ORACLEDB_USERNAME="<username>"
export ORACLEDB_PASSWORD="<password>"
~~~

The container can also be configured to retrieve the environment variables from a secret. \
To create it you can use the following command as an example:

~~~bash
# Creating the secret exposing the evironment variables for the Java microservice
oc create secret generic oracle-jdbc-env \
  --from-literal=url=jdbc:oracle:thin:@<database> \
  --from-literal=user=<username> \
  --from-literal=password=<password>
~~~

### Oracle DB deployment with TCP connection

#### Create the deployment: 
-  using the exported environment variables:
   ~~~bash
   envsubst < platform/ocp/oracle-db-tcp.yaml | oc create -f -
   ~~~

- using the secret that stores the environment variables:
  ~~~bash
  oc patch -f platform/ocp/oracle-db-tcp.yaml --type merge --patch "$(cat platform/ocp/patch-file-secret-env.yaml)" --local=true -oyaml | oc create -f -
  ~~~
  
The following is the JDBC URL connection string with JDBC Thin Driver using the TCP protocol:
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@tcp://$ORACLEDB_HOST:$ORACLEDB_PORT/<service-name>"
~~~

Refer to the official [Oracle JDBC driver documentation](https://docs.oracle.com/en/database/oracle/oracle-database/21/jajdb/oracle/jdbc/OracleDriver.html)
for more information about the URL formats.

<!-- 
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=$$ORACLEDB_HOST)(PORT=$ORACLEDB_PORT))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=<service-name>)))" 
~~~
-->

### Oracle DB deployment with TCPS connection and Oracle Wallet
Create the secret using the wallet downloaded from the Oracle Console:

~~~bash
oc create secret generic oracledb-wallet --from-file=<path-to-wallets-unzipped-folder>
~~~

Export the **ORACLEDB_WALLET_LOCATION** variable (or manually edit the [oracle-db-tcps-wallet.yaml](platform/ocp/oracle-db-tcps-wallet.yaml) file) 
which specifies the *mountPath* where the Java microservice application can find the SSO Wallet file (*cwallet.sso*) 
required to start the SSL/TLS connection. 

~~~bash
export ORACLEDB_WALLET_LOCATION="<wallet_location>"
~~~

Then *mountPath* (i.e., the **ORACLEDB_WALLET_LOCATION**) must coincide with the one selected for the **wallet_location** 
variable (used to set the *oracle.net.wallet_location* variable) specified in the Oracle URL string. \
The following is the JDBC EZConnect URL connection string with JDBC Thin Driver using the TCPS protocol and the SSO Wallet:
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@tcps://$ORACLEDB_HOST:$ORACLEDB_PORT/<service-name>?wallet_location=$ORACLEDB_WALLET_LOCATION"
~~~

<!--
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=$ORACLEDB_HOST)(PORT=$ORACLEDB_PORT))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=<service-name>)))?WALLET_LOCATION=$ORACLEDB_WALLET_LOCATION"
~~~
-->

#### Create the deployment:
- using the exported environment variables:
   ~~~bash
   envsubst < platform/ocp/oracle-db-tcps-wallet.yaml | oc create -f -
   ~~~

- using the secret that stores the environment variables:
  ~~~bash
  oc patch -f platform/ocp/oracle-db-tcps-wallet.yaml --type merge --patch "$(cat platform/ocp/patch-file-secret-env.yaml)" --local=true -oyaml | oc create -f -
  ~~~

#### Using TNS alias
It is also possible to simplify the JDBC URL connection string using the TNS alias. \
The connection string is found in the file *tnsnames.ora* which is part of the client credentials download. 
The *tnsnames.ora* file contains the predefined service names. Each service has its own TNS alias and connection string.

A sample entry, with *oracldb_name* as the TNS alias and a connection string in tnsnames.ora follows:

~~~
# You should replace **<SERVER_ADDRESS>** and the **<TCPS_PORT>** with the *IP Address* or *FQDN* and the *TCPS* port
# of the server hosting your database, respectively. 
oracldb_name =
  (DESCRIPTION =
    (ADDRESS_LIST = (ADDRESS = (PROTOCOL = TCPS)(HOST = <SERVER_ADDRESS>)(PORT = <TCPS_PORT>)))
    (CONNECT_DATA = (SERVICE_NAME = oracldb_name))
  )
~~~

The following is a sample connection string using TNS alias:
~~~bash
export ORACLEDB_URL="jdbc:oracle:thin:@oracldb_name?TNS_ADMIN=/path/to/wallet"
~~~

The **TNS_ADMIN** connection property specifies the following:
1. The location of *tnsnames.ora*.
2. The location of *Oracle Wallet*.
3. The location of *ojdbc.properties*. This file contains the connection properties required to use Oracle Wallets.

To use the JDBC Thin Driver to connect with TNS alis and Oracle Wallet, do the following:

1. **Update the connection URL** to have the required TNS alias and pass *TNS_ADMIN*, providing the path for *tnsnames.ora* and the wallet files.
    ~~~bash
    export ORACLEDB_URL="jdbc:oracle:thin:@<alias-in-tnsnames.ora>?TNS_ADMIN=$ORACLEDB_WALLET_LOCATION"
    ~~~
    
2. **Set the wallet location**. The properties file *ojdbc.properties* is pre-loaded with the wallet related connection property.
    ~~~bash
    oracle.net.wallet_location=(SOURCE=(METHOD=FILE)(METHOD_DATA=(DIRECTORY=${TNS_ADMIN})))
    ~~~
   
3. Update the **oracle-db-wallet** secret by adding the *tnsnames.ora*, *sqlnet.ora* and the *ojdbc.properties* files to the *<path-to-wallets-unzipped-folder>*:

    ~~~bash
    oc set data secret/oracle-db-wallet --from-file=wallet/
    ~~~

4. Delete the pods to let the deployment recreate them with the updated **oracle-db-wallet** secret.
    ~~~bash
    oc delete pods -l app=oracledb-health-check
    ~~~

Refer to the official Oracle [documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/adbsa/connect-jdbc-thin-wallet.html#GUID-BE543CFD-6FB4-4C5B-A2EA-9638EC30900D)
for more information about JDBC Thin Connections with a Wallet.

## Service Mesh configuration
Apply one of the following configurations to consume the Oracle DB service by the in-mesh Java microservice application:

- Control TCP egress traffic without a gateway:
  ~~~
  envsubst < networking/oracle-db-external-se.yaml | oc create -f -
  ~~~

- Direct TCP Egress traffic through an egress gateway:

  ~~~bash
  # Export the istio egress gateway port:
  export EGRESS_GATEWAY_ORACLEDB_PORT=<istio_egress_gateway_port>

  envsubst < networking/oracle-db-mesh-with-egressgateway.yaml | oc create -f -
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
