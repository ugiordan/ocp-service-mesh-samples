# Connecting to an Oracle Database outside the Mesh

This example shows how to communicate with an Oracle Database which is external to the Mesh.

## Configuration and deploy

### Service Mesh configuration
Export the environment variables for the OSSM configuration:
~~~
export ORACLEDB_HOST=<oracle_db_hostname>
export ORACLEDB_IP=<oracle_db_ip>
export ORACLEDB_PORT=<oracle_db_port>
export EGRESS_GATEWAY_ORACLEDB_PORT=<istio_egress_gateway_port>

oc create -f egress-rule-oracledb.yaml
~~~

### Application deployment configuration
Create the secret using the wallet downloaded from the Cloud Console:

~~~
oc create secret generic oracle-db-wallet --from-file=<path-to-wallets-unzipped-folder>
~~~

The deployment file uses the same mount path that the container configures in the **oracle.net.wallet_location** 
VM parameter [Dockerfile](src/java/wallet-secret-sample/src/main/docker/Dockerfile). \
Export the environment variables for the Java microservice and create the deployment:
~~~
# exporting the evironment variables for the Java microservice
export ORACLEDB_URL="jdbc:oracle:thin:@<alias-in-tnsnames.ora>"
export ORACLEDB_USERNAME="<username>"
export ORACLEDB_PASSWORD="<password>"
export ORACLEDB_WALLET_LOCATION="<wallet_location>"

# Creating the deployment 
envsubst < oracle-db-deployment.yaml | oc create -f -
~~~

The Java microservice pod can also be configured to retrieve the environment variables from a secret. \
To create it you can use the following command as an example:

~~~
# Creating the secret exposing the evironment variables for the Java microservice
oc create secret generic oracle-jdbc-env \
  --from-literal=url=$ORACLEDB_URL \
  --from-literal=user=$ORACLEDB_USERNAME \
  --from-literal=password=$ORACLEDB_PASSWORD \
  --from-literal=wallet_location=$ORACLEDB_WALLET_LOCATION
 
# Creating the deployment 
envsubst < oracle-db-deployment-env-secret.yaml | oc create -f -
~~~