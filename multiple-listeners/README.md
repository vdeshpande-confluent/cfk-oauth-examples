# Deploy Confluent Platform with multiple listeners

This guide provides step-by-step instructions to deploy Kafka and Zookeeper on Kubernetes using the Confluent Platform Operator. The Kafka deployment supports multiple listeners, including mTLS and Azure AD OAuth authentication.

## Prerequisites
Before you start, ensure the following:

- Kubernetes Cluster: A running Kubernetes cluster with access to deploy Confluent components.

- Confluent Operator: The Confluent Platform Operator must be installed in the cluster.

- Azure AD Configuration: Azure AD set up for OAuth with required client ID, tenant ID, and access to the JWKS and token endpoints. [Ref](https://github.com/vdeshpande-confluent/cfk-oauth/blob/main/README.md#setup-oauth)

- TLS Certificates: TLS secrets (tls-zookeeper and tls-kafka) are created in the confluent namespace.[Ref](https://github.com/vdeshpande-confluent/cfk-oauth/blob/main/README.md#create-tls-certificates)

Azure AD App Registration: A registered app in Azure AD with the necessary API permissions to generate OAuth tokens.[Ref]()

## Overview
This deployment includes:

- Zookeeper: A stateful Zookeeper cluster using mTLS for secure communication.
- Kafka: A Kafka cluster with three listeners for different authentication mechanisms:
- Custom Listener (OAuth): Secured with Azure AD OAuth for clients shifted to oauth.
- Internal Listener (mTLS): For internal cluster communication.
- External Listener (mTLS): For external client communication which will use still mtls.

## Custom Listeners
 
 ```
listeners:
    custom:
      - name: customlistener-oauth 
        port: 9205
        authentication:
          type: oauth
          jaasConfig:
            secretRef: oauth-jass
          oauthSettings:
            groupsClaimName: groups
            subClaimName: sub  #This config will determine the principal / username from the oauth configs , which will help keep the acls consistent
            audience:  <client-id>
            expectedIssuer: https://login.microsoftonline.com/<tenant-id>/v2.0
            jwksEndpointUri: https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys
            tokenEndpointUri: https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
        tls:
          enabled: true

 ```

## Review the additional listeners

Review Kafka ConfigMap to check the additional custom listeners
```
kubectl get configmap kafka-shared-config -o jsonpath="{.data.kafka\.properties}" | grep -i listener 

```
You should see the following, indicating the new listeners 

```

inter.broker.listener.name=REPLICATION
listener.security.protocol.map=CUSTOMLISTENER-MTLS:SSL,EXTERNAL:SASL_PLAINTEXT,INTERNAL:SSL,OAUTH:SASL_SSL,REPLICATION:SSL
listeners=CUSTOMLISTENER-MTLS://:9204,EXTERNAL://:9092,INTERNAL://:9071,OAUTH://:9205,REPLICATION://:9072

```
