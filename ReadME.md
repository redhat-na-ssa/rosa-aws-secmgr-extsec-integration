# This project shows the steps needs to integrate AWS secret manager with ROSA Cluster using ExternalSecret Operator

## 1) ROSA Cluster Steps

1) Get External Token from console.redhat.com/openshift/token/rosa
2) Install ROSA CLI
3) Check rosa login works. These commands needs to be performed after `aws configure`
    ```
         rosa login
         rosa list clusters
    ```    
oc login to Rosa clusters

## 2) AWS Steps

### Set environment variables
```
    export REGION=$(oc get infrastructure cluster -o=jsonpath="{.status.platformStatus.aws.region}")
    export OIDC_ENDPOINT=$(oc get authentication.config.openshift.io cluster \
    -o jsonpath='{.spec.serviceAccountIssuer}' | sed  's|^https://||')
    export AWS_ACCOUNT_ID=`aws sts get-caller-identity --query Account --output text`
    export AWS_PAGER=""
```    

### Create secret in AWS secret manager

```    
    SECRET_ARN=$(aws --region "$REGION" secretsmanager create-secret \
        --name MySecret --secret-string \
        '{"acscentral":"value-acscentral", "acstoken":"value-acstoken"}' \
        --query ARN --output text);
    echo $SECRET_ARN    
```        



### Create IAM policy to access the secret manager

The sample policy.json is provided. replace policy.json file with value replaced from SECRET_ARN. The below command does that
```
    POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn \
    --output text iam create-policy \
    --policy-name openshift-access-to-mysecret-policy \
    --policy-document file://policy.json)

    echo $POLICY_ARN
```


### Create trust policy

create `trust-policy.json`. Sample file is provided in the repo Replace OIDC_ENDPOINT and AWS_ACCOUNT_ID in this file before creating it.

```
    ROLE_ARN=$(aws iam create-role --role-name openshift-access-to-mysecret \
    --assume-role-policy-document file://trust-policy.json \
    --query Role.Arn --output text)

    echo $ROLE_ARN
```

### Attach POLICY_ARN to ROLE_ARN

``` 
      aws iam attach-role-policy --role-name openshift-access-to-mysecret --policy-arn $POLICY_ARN
```     

## 3) Openshift Steps

### Install ExternalSecret operator and configure

    1) Install externalsecret operator from operatorhub
    2) Create new project to run operatorconfig
            ```
               oc new-project secret-manager
            ```

    3) Create OperatorConfig CR in secret-manager namespaces This annotates the service account to use ROLE_ARN. use the provided `OperatorConfig.yaml` and replace ROLE_ARN with value on line 105 of this file with value from Step 2.

    The Operator config creates 3 pods and 3 service accounts namely
       
        a) config
        b) certs
        c) webhook.

    The config pod takes care of synchorizing the externalsecret and secretstore resources with the provider.so we assume IAM role for the pod. Inorder the assume this IAM role we annotate the service account with the $ROLE_ARN. The service account this annotation is required is `externalsecret-operator-config-external-secrets`  

### Verify AWS injection is successful by following commands:

```
    oc get sa externalsecret-operator-config-external-secrets -o yaml -n secret-manager | grep eks

    oc describe pod externalsecret-operator-config-external-secrets-59bf6bc47d4b7vv -n secret-manager | grep AWS_ROLE_ARN
```

The above command should display values of ROLE_ARN 


### Fetch secrets from AWS provider (SecretsManager) and create them in Openshift.

1) Create actual project namespace `oc new-project php-demo`
2) Create SecretStore Custom Resource. This resource configures the provider to where it has to fetched from. see  `secret-store.yaml`. perform `oc apply -k secret-store.yaml`
3) Create ExternalSecret Custom Resource. This connect the secretstore and the actual secret to be created in openshift. please refer `external-secret.yaml`. replace the required values and perform `oc apply external-secret.yaml`

# References 

https://external-secrets.io/v0.5.1/provider-aws-secrets-manager/
https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
