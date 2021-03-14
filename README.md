# tf-aws-eks-secrets

Use Case:

One of our clients is running Kubernetes on AWS (EKS + Terraform). At the moment, they store secrets like database passwords in a configuration file of the application, which is stored along with the code in Github. The resulting application pod is getting an ENV variable with the name of the environment, like staging or production, and the configuration file loads the relevant secrets for that environment.

We would like to help them improve the way they work with this kind of sensitive data.

Please also note that they have a small team and their capacity for self-hosted solutions is limited.

Provide one or two options for how would you propose them to change how they save and manage their secrets.


# Proposal
**Usage of AWS SSM parameter store and Kubernetes KMS**

The client can leverage the ssm parameter store service in AWS and can use the same in terraform for when creating or referencing infra on AWS. For accessing the secrets stored in SSM parameter store from EKS, they can use kubernetes-external-secrets(https://github.com/external-secrets/kubernetes-external-secrets).

The kubernetes-external-secrets project extends the Kubernetes API by adding a ExternalSecrets object using Custom Resource Definition and a controller to implement the behavior of the object itself.

An ExternalSecret declares how to fetch the secret data, while the controller converts all ExternalSecrets to Secrets. The conversion is completely transparent to Pods that can access Secrets normally.

By default Secrets are not encrypted at rest and are open to attack, either via the etcd server or via backups of etcd data. To mitigate this risk, use an external secret management system with a KMS plugin to encrypt Secrets stored in etcd.

System architecture

![image](https://user-images.githubusercontent.com/13218652/111053221-cf4fc780-8459-11eb-940e-9e48a412c5f4.png)

1. ExternalSecrets are added in the cluster (e.g., kubectl apply -f external-secret-example.yml)
2. Controller fetches ExternalSecrets using the Kubernetes API
3. Controller uses ExternalSecrets to fetch secret data from external providers (e.g, AWS Secrets Manager)
4. Controller upsert Secrets
5. Pods can access Secrets normally

**For using it with SSM parameter store**

User can scrape values from SSM Parameter Store individually or by providing a path to fetch all keys inside.
Additionally they can also scrape all sub paths (child paths) if needed. The default is not to scrape child paths
Below is an example for creating "External secret resource":
```
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: hello-service
spec:
  backendType: systemManager
  # optional: specify role to assume when retrieving the data
  roleArn: arn:aws:iam::123456789012:role/test-role
  # optional: specify region
  region: us-east-1
  data:
    - key: /foo/name
      name: fooName
    - path: /extra-people/
      recursive: false
```

**Here is an implementation plan for this proposal**
1) Deploy https://github.com/external-secrets/kubernetes-external-secrets using Helm and Integrate it with SSM
2) Deploy https://github.com/stakater/Reloader using Helm
3) Create ExternalSecret Resources for each application
4) Test the creation of Secrets by Updating SSM paramters
5) Add Annotations on Deployment to reload on changing the secrets
6) Create a script to push all env from files to SSM paramters Store
7) Push Env from files to SSM
8) remove all secrets from files and rewrite git history
9) They may also use https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/ for secret encryption within the EKS cluster
