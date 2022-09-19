# CP4BA - Oneshot GitOps Install


## Acknowledgements

This project was chiefly inspired off the good work done by the Apollo, GitOps and OTP teams as given below:

1) https://github.com/apollo-business-automation/ibm-cp4ba-enterprise-deployment 
2) https://github.com/one-touch-provisioning/otp-gitops
3) https://production-gitops.dev/ 


## Overview

Deploying CloudPaks into supported clouds in a repetable, automatable fashion is a non trivial, potentially time consuming process. This project aims to lay the foundation for future deployments, culminating in an end state which supports the following capabilities:

1) Ability to select CloudPaks, and which services within said CloudPak to deploy
2) Ability to deploy common, "big win" use cases (applications) using said framework
3) Native integration with a wide range of CI Tooling. Eg, Jenkins.
4) A range of target deployments options given, ranging from all key supported hyperscalers to on-premise deployments
5) Inbuilt monitoring and auditing framework to support day 2 operations 
6) And more

This is still a work in progress, we expect more work to be done here down the road.

## Shoutouts

This base implementation would not have been possible without the following individuals:

1) Leela Chitta
2) Ondrej Svec
3) Dalli Bagdi
4) Tim Quigly
5) Jan Dusek


## Usage

This automation involves, as one would expect, the creation of secrets. Creating plain Kubernetes secrets and storing the required K8's objects as plain text in a Git Repository is ill advised. One way to circumnavigate is through [SealedSecrets] (https://github.com/bitnami-labs/sealed-secrets). That said, using SealedSecrets to handle a large number of secrets can quite get clunky. A number of manual steps are involved which, from a maintainability perspective, does not scale so well. On the other hand, we may need to integrate with a client secret store - they may prefer to use their own instance of Vault for example. As a result, we have externalised the secrets to an external store. We use the [ExternalSecret] operator (https://external-secrets.io/v0.6.0-rc1/) to accomodate for this. As given in the link, the following (amongst others) secret manager instances are supported:

1) AWS Secrets Manager
2) Azure Key Vault
3) Google Secrets Manager
4) HashiCorp Vault

We leverage IBM Secrets Manager for [this] (https://cloud.ibm.com/docs/secrets-manager?topic=secrets-manager-vault-api), which uses a custom version of open source HashiCorp Vault for this. The secrets we need to create to standup CP4BA with FileNet and IER are the following:

1) A Universal Password
2) An IBM Entitlement Key
3) An LDAP secret (Admin and Config Password) assigned to the root user

We have, for sake of simplicity, created a secret - the Universal Password - and assigned this secret to the various different services constituting this CloudPak, as seen shortly. In a production setting, it is recommended to have a "one to one" mapping as opposed to a "many to one" mapping as we have done here.

The upshot of all this is one instance of a SealedSecret needs to be created, which is the secret required to access the external secret store.


### Secret Creation

As mentioned earlier, we will leverage IBM Secrets Manager for this. Non-privileged accounts do get a thirty day free trial to deploy an instance of Secrets Manager in IBM Cloud. Refer to the image below.

![IBM Cloud - Secrets Manager](Images/SecretsManagerIBMCloud.png)