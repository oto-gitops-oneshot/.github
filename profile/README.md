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

## Setup

Please ensure you follow the steps outlined in the [main repository](https://github.com/oto-gitops-oneshot/otp-gitops). For those of you not familiar with the pattern, the sections starting with [Elevator Pitch](https://github.com/oto-gitops-oneshot/otp-gitops#elevator-pitch) and ending in [Use Cases](https://github.com/oto-gitops-oneshot/otp-gitops#use-cases-for-different-git-repository-organisation) establish the context and motivation quite well.

With that out of the way, please ensure the commands contained in the sections starting with [Setup Git Repositories](https://github.com/oto-gitops-oneshot/otp-gitops#setup-git-repositories) and ending (**inclusive**) in the **third** command in [Bootstrap the OpenShift Cluster](https://github.com/oto-gitops-oneshot/otp-gitops#bootstrap-the-openshift-cluster-) are executed successfully.

For sake of completeness, the third command stated above is:

```
oc apply -f 0-bootstrap/hub/bootstrap.yaml
```

Once you've executed the above command anc wuccessfully obtained the Argo URL and password, you may proceed with the next section below.

## Prerequisites

This automation involves, as one would expect, the creation of secrets. Creating plain Kubernetes secrets and storing the corresponding K8's YAMLs as plain text in a Git Repository is ill advised. One way to circumnavigate this is through [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets). That said, using SealedSecrets to handle a large number of secrets can quite get clunky. It incurs a management overhead which, from a maintainability perspective, does not scale so well. In practice, integrating with a pre-existing client secret store is the most likely scenario - as opposed to internally managing secrets ourselves. As a result, we have externalised the secrets to an external store. We use the [ExternalSecret operator](https://external-secrets.io/v0.6.0-rc1/) to accomodate for this. As given in the link, the following (amongst others) secret manager instances are supported:

1) Google Secrets Manager
2) AWS Secrets Manager
3) Azure Key Vault
4) HashiCorp Vault

This is great. Customers on AWS, Azure and Google will (most likely) use the corresponding secret providers given above. HashiCorp Vault is fairly popular too. This way, we "meet customers in the middle". 

In this project, we leverage IBM Secrets Manager for [this](https://cloud.ibm.com/docs/secrets-manager?topic=secrets-manager-vault-api), which uses a custom version of open source HashiCorp Vault, one of the supported instance types as mentioned above. The secrets we need to create to standup CP4BA with FileNet and IER are the following:

1) A Universal Password
2) An IBM Entitlement Key
3) An LDAP secret (Admin and Config Password) assigned to the root user

We have, for sake of simplicity, created a secret, aptly called universalPassword, and assigned this secret to the various different services constituting this CloudPak, as we shall see in the next section. In a production setting, it is recommended to have a "one to one" mapping between secret and corresponding service as opposed to a "many to one" mapping as we have done here. This is part of our roadmap.

The upshot of all this is one (as opposed to many) instance of a SealedSecret needs to be created, which is the secret required to access the aforementioned external secret store.

### Prerequisite - Secret Creation

As mentioned earlier, we will leverage IBM Secrets Manager for this. Non-privileged accounts do get a thirty day free trial to deploy an instance of Secrets Manager in IBM Cloud. Refer to the image below.

![IBM Cloud - Secrets Manager](Images/SecretsManagerIBMCloud.png)

First and foremost, you will need an API Key to access this instance programmatically (this API Key is our SealedSecret).

![IBM Cloud - Secrets Manager - API Key](Images/API_KEY.png)

For more information on how to obtain an API Key, please refer to the following [link](https://cloud.ibm.com/docs/account?topic=account-userapikey&interface=ui). I do encourage you to read the third paragraph found in the link should you wish to deploy this asset in a production setting. In particular: **"it is recommended that you create an API key that is associated with a functional ID that is assigned the minimum level of access that is required to work with the service".** This is a no brainer. Abiding by the principles of least privilege (or common sense), your API Key should only have READ access to said instance, no more and no less.

With that out of the way, we can now go ahead and create the secrets required to standup the CloudPak. Your UI should resemble the following once you are done with the procedure.

![IBM Cloud - Secrets Manager - UI](Images/SM_UI.png)

Take note of the key names here. It is recommended to leave them as such, otherwise you will have to update the names upstream in the relevant YAML files (which is dealt with in the upcoming section). Please mind the camelCase format should you choose to change the name, do not use snake_case or kebab-cases. These special characters are interpreted and parsed differently and will result in erraneous behaviour. 

Follow the steps given below to create the adminPassword, universalPassword and configPasswords. The corresponding secrets are of type string. Feel free to choose any string value you please. It is recommended to choose a random secure password. This [link](https://www.helperset.com/tools/generate-secure-string) generates a random secure string on demand. 15 or so characters should suffice. The default (32) is overkill.

Click "Add", located towards the right of the screen. You will be presented with the following options:

![IBM Cloud - Secrets Manager - Options](Images/SM_Options.png)

Click "Other Secret Type". You will be given the following:

![IBM Cloud - Secrets Manager - Options](Images/SM_Other.png)

The name should correspond to the names given earlier. Bear in mind the strict camelCase convention imposed in the event you do choose to use a different name. Populate the secret value with the random secure string mentioned earlier. Take note of the ID associated with this password and store it somewhere for the time being. Refer to the image below in the event you don't have the ID's handy. Simply click the "Details" section and copy the ID presented to you.

![IBM Cloud - Secrets Manager - Details](Images/SM_Details.png)

Rinse and repeat till the following passwords are created in secrets manager:

1) universalPassword
2) adminPassword
3) configPassword

The IBM Entitlement Key creation is slightly more involved. You will have to obtain the JSON representation of the entitlement key. 

First and foremost, navigate to this [site](https://myibm.ibm.com/products-services/containerlibrary) to obtain your key. Once the key is obtained, run the following bad boy of a command:

```
oc create secret docker-registry ibm-entitlement-key --dry-run=client -o json \
--docker-username=cp \
--docker-password="your_ibm_entitlement_key_here" \
--docker-server=cp.icr.io \
--docker-email=your_email_here@ibm.com | jq '.data.".dockerconfigjson"' |  tr -d '"' | base64 --decode
```

The following assumptions are made:

1) The "oc" binary is installed and present in the path variable
2) The "jq" binary is installed and present in the path variable
3) Unix/Linux workstation. TODO: Find equivalent windows command.

You need not login to an openshift cluster for this thanks to the dry-run parameter present in the command.

Be sure to replace the values associated with the docker-password and docker-emails fields accordingly. Copy the output and follow the same procedure for secret creation as given above for the ibm entitltment key, except this time you will be pasting the output of the command given above as opposed to generating a random secure string. Please make note of the ID here as well.

Your UI should resemble the following now.

![IBM Cloud - Secrets Manager - Final](Images/SM_Final.png)

Before proceeding with the next section, navigate to the "Endpoints" tab as given below, and note down the Public Endpoint (highlighted) below:

![IBM Cloud - Secrets Manager - Public Endpoint](Images/SM_Public_Endpoints.png)

Well done! Give yourself a pat in the back. 

### Prerequisite - Secret Updates

Within the services repository you cloned, the following files need to be updated:

1) SECRET_PATH/ban.yaml
2) SECRET_PATH/fncm.yaml
3) SECRET_PATH/ibm-entitlement-key.yaml
4) SECRET_PATH/ier.yaml
5) SECRET_PATH/ldap-bind.yaml
6) SECRET_PATH/rr.yaml
7) SECRET_PATH/universal-password.yaml
8) DB2_PATH/create/base/pull-secret.yaml
9) LDAP_PATH/templates/external-admin-secret.yaml

Where:
```
SECRET_PATH=instances/cloudpak/cp4ba/predeploy/secrets
DB2_PATH=instances/db2
LDAP_PATH=instances/openldap
```

Specifically, the files given in list elements 1, 2, 4, 6 and 7 need to have their spec.data.remoteRef.key value updated with the id associated with the universalPassword secret you created in the previous section for the list entry with name universalPassword. If you used a different name, this name field would also need to be updated accordingly. Please refer to the image below. I've highlighted the relevant field to update. (Hint: this field is found in line 10 of each file)

![IBM Cloud - Secrets Manager - Details](Images/ES_Field.png)

Update list elements 3 and 8 with the id associated with the ibmEntitlementKey. If you used a different name, this name field would also need to be updated. (Hint: this field is found in line 10 of each file)

Update list elements 5 and 9 with the LDAP admin and config passwords created in the previous section. It should be fairly straighforward to conclude as to what this procedure entails.

Finally, modify the serviceUrl field in the **"cluster-secret-store.yaml"** file found in **"instances/external-secrets-instance/overlays/default/"** directory to coincide with the public endpoint of your secrets manager (vault) instance you noted down in the previous section. Please refer to the image below.

![IBM Cloud - Secrets Manager - Public Endpoint - YAML](Images/SM_ServiceUrl.png)

Make sure you commit and push your changes accordingly.

Please do note, we do ultimately want to provide automation around this, when this is consumable via TechZone. For now, bear with us. This is the automation we foresee being carried out behind the scenes in any case.

## Usage

Please ensure you have completed the steps given in the [Setup](#Setup) before proceeding with the upcoming sections.

### Usage - Infra

By default, lines 2 through to 6 (inclusive) found in the file **"0-bootstrap/kustomization.yaml"** are commented out. This is to ensure nothing is provisioned initially. See the image below.

![GitOps - Main Repo - Kustomize - Outer](Images/Bootstrap_Kustomization.png)

Simply uncomment line 2 (the infra line), save, commit and push. Within a matter of a few minutes, the infrastructure components required to stand up CP4BA with FileNet and IER are provisioned. The upshot of all this is that the infra app, app-project and child applications are created successfully and displaying a healthy status according to Argo.

![GitOps - Main Repo - Parent - Main App](Images/Parent-Infra-Service.png)

Note the services application and appProject should not have been stood up at this stage. Use your power of imagination to fool you into believing they are not there yet.

![GitOps - Main Repo - Parent - Infra App](Images/Infra.png)

For more information on the infrastructure components stood up, please refer to the README provided in the [infra repository](https://github.com/oto-gitops-oneshot/otp-gitops-infra).

### Usage - Service

The infrastructure components stood up in the previous section included the sealed secret operator, thereby allowing downstream applications to create sealed secrets. As such, prior to uncommenting the services line in "0-bootstrap/kustomization.yaml", the secret granting access to the external secret store must be committed first to the repository. Please complete the following steps:

1) Navigate to your cloned sevices repository, specifically to the "instances/api-key-sealedsecret" directory. Therein lies a script. It is pretty self explanatory. A sealedSecret is created from your ibm cloud api key you created in [Secrets](#prerequisite---secret-creation)
2) Export your api key as an environment variable as given in the command below.
3) Run the script. The command is provided below.
4) Save, commit and push your changes.

The commands are as follows:

```
export API_KEY="your_api_key_here"
./api-key-sealedsecret.sh
```

Now, navigate back to the main repo. Uncomment the services entry found in the "0-bootstrap/kustomization.yaml" as given below. Save, commit your changes and push.

![GitOps - Main Repo - Parent - Kustomize File](Images/Bootstrap-uncomented.png)

The services app and appProject should have now been stood up as given below.

![GitOps - Main Repo - Parent - Main App](Images/Parent-Infra-Service.png)

And this kickstarts the provisioning of the child applications defined within the service application. The whole process should take approximately an hour (or perhaps a little more) to complete. This is the perfect time to sit back, reflect and question your life decisions that led you to this very moment. 

![GitOps - Main Repo - Parent - Service App](Images/Services.png)

The README provided in the [services repository](https://github.com/oto-gitops-oneshot/otp-gitops-services) offers a deeper insight into the resources provisioned.

## Verification

Congratulations! Give yourself another pat in the back. 

The exposted routes for the various applications stood up can be found in the ConfigMap called **icp4adeploy-cp4ba-access-info** in the **cp4ba** namespace. For instance, the content platform engine route is as such:

```
Content Platform Engine administration: https://domain_here/cpe/acce/
```

And the navigator route:

```
Business Automation Navigator for CP4BA: https://domain_here/icn/navigator/
```

Needless to say, replace **domain_here** accordingly.

The username and password are **cpadmin** and the value assigned to the **universalPassword** secret you created earlier.


## Roadmap

As this was intially driven by a client POC, we do plan to iterate on this, and make this a fully fledged consumable product in the upcoming future. We do plan to tackle the following points progressively:

1) TODO: Fill this out