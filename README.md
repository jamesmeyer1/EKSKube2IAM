# Securing Pods for EKS

## Goal of the Builder Session
This builder session will take you through the process of securing Kubernete Pods running on Amazon Elastic Kubernetes Service (EKS). We will setup a VPC with proper tagging, configure kubectl, create the required trust, roles, and IAM policies that are required to apply pod level security using [kube2iam](https://github.com/jtblin/kube2iam) for EKS. We will also apply IAM roles to a namespace to restrict pods to a specific IAM role.  

![Overview of Architeture](https://github.com/meyjames/Kubernetes/blob/master/podlevel.png)

```
Things I would copy down as you see them.
1) EKS Cluster Name
        You will need this when you create your nodes
2) NodeInstanceRole
        This will be available in Outputs after you launch the nodes from CloudFormation
3) Arn of the Role you create to access S3
        You will need this for the application testing and to configure the test namespace permissions
```

# Environment Setup:
## Step 1:
Download this repository so you can edit and deploy stacks locally. 

We begin by creating a VPC for Kubernetes in the US East (Ohio) region. Please check the region in the Management Console before you begin. When you create your Amazon EKS cluster, Amazon EKS tags the VPC containing the subnets you specify in the appropriate way so Kubernetes can discover them. You can read about the subnet and VPC tagging performed [here](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html#vpc-tagging). For this example the default IP range will work.

Create the VPC using the CloudFormation stack listed below.
```
amazon-eks-vpc.yaml
```  

## Step 2:
If you have never launched and EKS instance you will need to create a role as outlined [here](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html). It provides the IAM policies you need to associate to your role, which are below too.
* AmazonEKSServicePolicy
* AmazonEKSClusterPolicy

## Step 3: 
Deploy EKS Cluster:
1) Select EKS from Services and deploy your EKS Cluster. Make sure to check the region US East (Ohio) us-east-2 in the AWS Management Console. The format for you subnets will be: stackname-Subnet#. Please select the three subnets that were created for you based on your stackname.  

* Select the VPC with stackname-VPC
* Select the security group with stackname-ControlPlaneSecurityGroup-####

Creating an EKS cluster can take up to 15 minutes. We can use this time to update our CLI, install Kubectl, and create a key pair which are required for this exercise. 

## Step 4: 
Update AWS ClI to latest version:
Update or install the Latest [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) for your operating system.

## Step 5: 
Install Kubectl based on your OS:
EKS uses a command line utility called kubectl for communicating with the cluster API server. The instructions for installing your specific operating system or package mananager are [here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

## Step 6:
Create a key pair before launching the instances as outlined [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)

## Step 7: 
Launch Instances:
After the EKS cluster becomes available you can launch your instances. You need the current optimized AMI for the [Amazon EKS worker nodes](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)

Use the CloudFormation Stack below to build out the work nodes. 
```
nodesWorkshop.yaml 

Please name your cluster the same as in the previous steps!!
```

## Step 8: 
Configure Instances to Join Cluster:
        
You must enable the worker nodes to join the cluster you created. 

* Download, edit, and apply the AWS IAM Authenticator configuration map. Use the following command to download the configuration map:
```
curl -o aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/aws-auth-cm.yaml
```
* Open the file with your favorite text editor. Replace the <ARN of instance role (not instance profile)> snippet with the NodeInstanceRole value that you recorded in the previous procedure and save the file. This ARN can be found under Outputs in CloudFormation or from the Description section of one of the EC2 instances created for you.  

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <ARN of instance role (not instance profile)>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

## Step 9: 
Use the AWS CLI to create or update your kubeconfig for your cluster. This will combine other contexts. You can read more about this process including troubleshooting tips [here](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)

```
aws eks --region <region> update-kubeconfig --name <cluster_name>
```

```
kubectl apply -f aws-auth-cm.yaml
```

```
kubectl get svc
```
When that command returns your cluster, add a namespace for testing pod level IAM control.
```
kubectl create namespace test
```
## Step 10: 
Create IAM Policy for Nodes:
IAM roles

Create a role with S3 permissions and edit the Trust Relationship to include the following. The role that will be assumed must have a Trust Relationship which allows them to be assumed by the Kubernetes worker node. Replace the Principal below with the role's ARN assigned to the worker nodes. 
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "<replace with NodeInstanceRole>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

The following policy was created in the CloudFormation stack for you. This is required and allows the worker nodes to assume the role you assign to the pods. 
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sts:AssumeRole"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

## Step 11: 
Deploy kube2iam to your cluster. You do not need to edit anything here. You will notice we have configured the flag --namespace-restrictions=true.

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube2iam
  namespace: kube-system
---
apiVersion: v1
items:
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube2iam
    rules:
      - apiGroups: [""]
        resources: ["namespaces","pods"]
        verbs: ["get","watch","list"]
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube2iam
    subjects:
    - kind: ServiceAccount
      name: kube2iam
      namespace: kube-system
    roleRef:
      kind: ClusterRole
      name: kube2iam
      apiGroup: rbac.authorization.k8s.io
kind: List
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube2iam
  namespace: kube-system
  labels:
    app: kube2iam
spec:
  selector:
    matchLabels:
      name: kube2iam
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: kube2iam
    spec:
      serviceAccountName: kube2iam
      hostNetwork: true
      containers:
        - image: jtblin/kube2iam:latest
          imagePullPolicy: Always
          name: kube2iam
          args:
            - "--auto-discover-base-arn"
            - "--namespace-restrictions=true"
            - "--auto-discover-default-role=true"
            - "--iptables=true"
            - "--host-ip=$(HOST_IP)"
            - "--node=$(NODE_NAME)"
            - "--host-interface=eni+"
            - "--verbose"
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          ports:
            - containerPort: 8181
              hostPort: 8181
              name: http
          securityContext:
            privileged: true
```
## Step 12: 
I have included a way to test your S3 permissions. Replace the role with the role ARN you created above with S3 permissions.   

```
apiVersion: v1
kind: Pod
metadata:
  name: s3
  namespace: test
  labels:
    name: s3
  annotations:
    iam.amazonaws.com/role: <replace with the ARN of the role with S3 permissions>
spec:
  containers:
  - image: fstab/aws-cli
    command:
      - "/home/aws/aws/env/bin/aws"
      - "s3"
      - "ls"
    name: s3
```

```
kubectl apply -f pathto/s3.yaml
```

If you execute the command below will you see your buckets? Why or why not?   
```
kubectl logs s3 --namespace=test
```

## Namespace Restrictions
By using the flag --namespace-restrictions you can enable a mode in which the roles that pods can assume is restricted by the annotation on the pod's namespace. This annotation should be in the form of a json array and an example is below. We will add the role ARM we created earlier to the namespace so it has an allowed role. We will be allowing access to S3.

## Step 13: 
To allow the aws-cli pod specified above to run in the test namespace you should replace the ARN with the S3 role created earlier. This file is called namespace.yaml. Replace the ARN and deploy.

```
apiVersion: v1
kind: Namespace
namespace: test
metadata:
  annotations:
    iam.amazonaws.com/allowed-roles: |
      ["put s3 arn here"]
  name: test
```

```
kubectl apply -f <pathto/namespace.yaml>
```
Redploy S3.yaml now that a role has been configured. Can you see your buckets? 

## Step 14:
Create another role with the same permissions and trust policy as above and redeploy s3.yaml with that ARN. 
* Can you make it work with the new ARN?

You can read about path-based and glob-based matching for additional namespace restriction approaches on the [kube2iam site](https://github.com/jtblin/kube2iam#namespace-restrictions). This was one approach to apply IAM role namespace restrictions on a pod. 

IAM roles for pods provides the level of security certain workloads require. Instead of assuming the worker node role for all pods you can customize the permissions a pod can assume. Hopefully this workshop provides the foundation for you to extend new levels of controls to your EKS environment at the pod level.  
