OAuth for CFK with Azure
Confluent Platform 7.7 and CFK 2.9.0 released OAuth support. This repository demo up a basic CP cluster via CFK running on Azure Kubernetes Service (AKS) with mtls migrate to external clients to use OAuth and mtls for authentication.Also demonstrates migrating ACLs from mTLS to OAuth.

In technical detail it deploys:

1 Zookeeper
3 Kafka brokers
1 Control Center

General resources:
* [Quickstart: Deploy an Azure Kubernetes Service (AKS) cluster using Azure portal](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal?tabs=azure-cli)
* [Confluent for Kubernetes Quick Start](https://docs.confluent.io/operator/current/co-quickstart.html)
* [CP Mtls OAuth setup](https://github.com/confluentinc/confluent-kubernetes-examples/blob/master/security/userprovided-tls_mtls_kafka-acls/README.md)


## Deploy Confluent for Kubernetes

Set up the Helm Chart:

```
helm repo add confluentinc https://packages.confluent.io/helm
```

Install Confluent For Kubernetes using Helm:

```
helm upgrade --install operator confluentinc/confluent-for-kubernetes --namespace confluent
```
  
Check that the Confluent For Kubernetes pod comes up and is running:

```
kubectl get pods --namespace confluent
```

## Create TLS certificates

In this scenario, you'll configure authentication using the mTLS mechanism. With mTLS, Confluent components and clients use TLS certificates for authentication. The certificate has a CN that identifies the principal name.

Each Confluent component service should have it's own TLS certificate. In this scenario, you'll
generate a server certificate for each Confluent component service. Follow [these instructions](../../assets/certs/component-certs/README.md) to generate these certificates.The component certs are provided for the example in this repo as well. 


These TLS certificates include the following principal names for each component in the certificate Common Name:
- Kafka: `kafka` (Take a look at CN in component-certs/kafka-server-domain.json , this is where the principal would be extracted from according to the mapping rule specified in below sections.)
- Schema Registry: `sr`
- Kafka Connect: `connect`
- Kafka Rest Proxy: `krp`
- ksqlDB: `ksql`
- Control Center: `controlcenter`
     
## Deploy configuration secrets

you'll use Kubernetes secrets to provide credential configurations.

With Kubernetes secrets, credential management (defining, configuring, updating)
can be done outside of the Confluent For Kubernetes. You define the configuration
secret, and then tell Confluent For Kubernetes where to find the configuration.

To support the above deployment scenario, you need to provide the following
credentials:

* Component TLS Certificates

* Authentication credentials for Zookeeper, Kafka

### Provide component TLS certificates

Set the tutorial directory for this tutorial under the directory you generated the certificates and pem files:

```
export TUTORIAL_HOME=<Set this to current directory>
```

In this step, you will create secrets for each Confluent component TLS certificates.For the demo purpose we will be going ahead with kafka and zookeeper.

```
kubectl create secret generic tls-zookeeper \
  --from-file=fullchain.pem=$TUTORIAL_HOME/component-certs/generated/zookeeper-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/component-certs/generated/zookeeper-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-kafka \
  --from-file=fullchain.pem=$TUTORIAL_HOME/component-certs/generated/kafka-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/component-certs/generated/kafka-server-key.pem \
  --namespace confluent
```

## Deploy Confluent Platform

Deploy Confluent Platform:

```
kubectl apply -f $TUTORIAL_HOME/mtls-oauth-migration/onlymtls.yaml --namespace confluent
```

Check that all Confluent Platform resources are deployed:

```
kubectl get pods --namespace confluent
```

## Create the ACLs for each component

You'll see that Schema Registry, Connect, ksqlDB, Control Center - all fail to come up. This is because 
the Kafka ACLs that let these components create, read and write to their required topics have not been 
created.

Read up on the ACL format and concepts here: https://docs.confluent.io/platform/current/kafka/authorization.html#acl-format

In this step, you'll create the required ACLs to start each Confluent component.

### Create ACLs using tooling on Kafka pod

Note: Bashing to the Broker pod is ok in order to test functionality.  
For production scenarios you'll want to run the CLI or call the Admin API from outside the Kafka cluster and either connect over the internal or external Kubernetes network.  

Open an interactive shell session in the Kafka broker container:

```
kubectl -n confluent exec -it kafka-0 -- bash
```

Create the client configuration to connect to the Kafka cluster over
the internal Kubernetes network:

```
cat <<-EOF > /opt/confluentinc/kafka.properties
bootstrap.servers=kafka.confluent.svc.cluster.local:9071
security.protocol=SSL
ssl.keystore.location=/mnt/sslcerts/keystore.p12
ssl.keystore.password=mystorepassword
ssl.truststore.location=/mnt/sslcerts/truststore.p12
ssl.truststore.password=mystorepassword
EOF
```

## Validate 
Test the certificates creating topics and producing to them

```
/bin/kafka-topics --create --bootstrap-server kafka.confluent.svc.cluster.local:9092 --topic test-initial --command-config /opt/confluentinc/kafka.properties

```

## Create ACLs to test the migration:
For client principals extracted from other tls certificates , below is the mapping rule specified for a certificate

```
principalMappingRules:
          - RULE:^CN=([a-zA-Z0-9]*).*$/$1/

```

A distinguished name (DN) is the concatenation of the X.500 attributes that uniquely identify the subject of a TLS certificate. Here’s an example of a DN:

CN=writeuser,OU=Unknown,O=Unknown,L=Unknown,ST=Unknown,C=Unknown

For above example , the principal extracted from the rule would be "writeuser" , take note of how principals are extracted. 

For more info on this please refer here :https://docs.confluent.io/platform/current/security/authentication/mutual-tls/tls-principal-mapping.html


Create ACLs for the principal 
```

/bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
 --command-config /opt/confluentinc/kafka.properties \
 --add \
 --allow-principal "User:writeuser" \
 --operation Read --operation Write --operation Create --operation Alter --operation AlterConfigs --operation Delete \
 --topic test \
 --resource-pattern-type prefixed

```
# Setup Oauth 

## Azure AD endpoints

For the later configuration we need to set the `token_endpoint`, the `jwks_uri`, and the `issuer`.
We can obtain all information via

```shell
curl https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration | jq
```

Generally, those are
```
token_endpoint = https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
jwks_uri = https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys
issuer = https://login.microsoftonline.com/<tenant-id>/v2.0
```

## Azure AD applications

To retrieve the JWT token, CP is using the client credentials grant flow. So, we need to register an application in Azure AD
and create a secret. 
We can get a JWT token via: 
```
curl -X POST -H "Content-Type: application/x-www-form-urlencoded" \
-d 'client_id=[client_id]&client_secret=[client_secret value]&grant_type=client_credentials' \
https://login.microsoftonline.com/[tenant_id]/oauth2/token
```

> [!NOTE]
> In this example, we only register one application in Azure AD. Consider different applications with its secret per CP component
> and client.



## CFK cluster

* [CFK server-side OAuth/OIDC authentication](https://docs.confluent.io/operator/current/co-authenticate-kafka.html#server-side-oauth-oidc-authentication-for-ak-and-kraft)

Store the clientId and secret in a file and deploy it as a k8s secret. 
```shell
kubectl create -n confluent secret generic oauth-jass --from-file=oauth.txt=client-credentials.txt
```

Afterwards, we configure the Kafka CR
```yaml
listeners:
  external:
    authentication:
      type: oauth
      jaasConfig:
        secretRef: oauth-jass
      oauthSettings:
        groupsClaimName: groups
        subClaimName: sub
        audience: <client-id>
        expectedIssuer: see above
        jwksEndpointUri: see above
        tokenEndpointUri: see above
configOverrides:
  server:
    - listener.name.external.oauthbearer.sasl.server.callback.handler.class=org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerValidatorCallbackHandler
    - listener.name.external.oauthbearer.sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
```

Once the `confluent-operator` is running, we deploy the cluster with 

```
kubectl apply -f ./mtlswithoauth.yaml -n confluent
```


## Test By Producing 

```
cat <<-EOF > /opt/confluentinc/kafka.properties
bootstrap.servers=kafka.confluent.svc.cluster.local:9092
ssl.keystore.location=/mnt/sslcerts/keystore.p12
ssl.keystore.password=mystorepassword
ssl.truststore.location=/mnt/sslcerts/truststore.p12
ssl.truststore.password=mystorepassword
sasl.mechanism=OAUTHBEARER
security.protocol=SASL_SSL
sasl.login.callback.handler.class=org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler
sasl.login.connect.timeout.ms=15000
sasl.oauthbearer.token.endpoint.url=https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required clientId='<clientId>' clientSecret='<clientSecret>' scope='<clientId>/.default';
EOF
```
```
/bin/kafka-topics --create --bootstrap-server kafka.confluent.svc.cluster.local:9092 --topic test-later --command-config /opt/confluentinc/kafka.properties

```

In OAuth, the username is typically extracted from a claim in the OAuth token (for example, the "sub" claim specified under subClaimName in oauth configs).
Your Confluent Server broker configuration determines how these identities are mapped to the User:<username> format. If configured correctly, the principal can remain the same when migrating from mTLS to OAuth, meaning no changes to ACLs are necessary.

If the subClaimName (sub) field value from jwt token matches the principal extracted from tls certificate which we created acl for , this should run successfully. 

If not , take a look at adding optional claims for users and groups in azure AD and map that field to subClaimName : https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims?tabs=appui 


Alternatively you can use Microsoft identity platform application authentication certificate credentials.
[An application can use for authentication a JSON Web Token (JWT) assertion signed with a certificate that the application owns.](https://learn.microsoft.com/en-us/entra/identity-platform/certificate-credentials)
[Access token request with a certificate](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow#second-case-access-token-request-with-a-certificate)
Some more reference can be found here : https://gist.github.com/phillipharding/d6cc4b84221cf5304e41d97beb62ea6c
