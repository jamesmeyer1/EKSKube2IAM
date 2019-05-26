# Securing Pods for EKS

## Goals of the Workshop
This project will take you through the process of securing Kubernete Pods running on Amazon Elastic Kubernetes Service (EKS). We will setup a VPC with the proper tagging, configure kubectl locally, and create the trusts, roles, and IAM policies that are required to apply pod level security on EKS using [kube2iam](https://github.com/jtblin/kube2iam). 

![Overview of Architeture](https://github.com/meyjames/Kubernetes/blob/master/podlevel.png)

```
Things I would copy down as you see or create them.
1) EKS Cluster Name
        You will need this when you create your nodes
2) NodeInstanceRole
        This will be available in Outputs after you launch the nodes
3) Arn of the Role you create to access S3
        You will need this for the applicatioin testing
```

# Environment Setup:
## Step 1:
Create a VPC for Kubernetes. When you create your Amazon EKS cluster, Amazon EKS tags the VPC containing the subnets you specify in the following way so that Kubernetes can discover it. You can read about the subnet and VPC tagging performed here. You can include this address or upload the file above with same name. 

Launch Stack: [stack](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/template?stackName=stack_name&templateURL=http://dy9fuoxcazx2i.cloudfront.net/amazon-eks-vpc.yaml)

## Step 2: Deploy EKS Cluster:
1) Check the region in the Management Console. We are deploying to US East (Ohio) us-east-2. You will need to select the subnets for your cluster. Your subnets will have tags that will define them. The format will be: stackname-Subnet#. Please select the three subnets that were created for you. 

## Step 3: Update AWS ClI to latest version:
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html

## Step 4: Install Kubectl based on your OS:
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

## Step 5: Launch and Configure Instances:
Link to current EKS Optimized AMIs: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
Use the CloudFormation Stack called <final Name>
        
You must enable the worker nodes to join the cluster created. 

5A) Download, edit, and apply the AWS IAM Authenticator configuration map. Use the following command to download the configuration map:
```
curl -o aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/aws-auth-cm.yaml
```
5B) Open the file with your favorite text editor. Replace the <ARN of instance role (not instance profile)> snippet with the NodeInstanceRole value that you recorded in the previous procedure, and save the file.
        Important
        Do not modify any other lines in this file.
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

## Step 6: 
Use the AWS CLI to create or update your kubeconfig for your cluster. This will combine other contexts. You can read more about this process including troubleshooting tips here.
https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html

```
aws eks --region <region> update-kubeconfig --name <cluster_name>
```
```
kubectl get svc
```

## Step 7: Create IAM Policy for Nodes:
IAM roles

The Kuberentes workers will need to assume a role. Please include the following in a policy and create a role. Name it something you will remember. You will need it later. 
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

The roles that will be assumed must have a Trust Relationship which allows them to be assumed by the kubernetes worker role. 
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
        "AWS": "arn:aws:iam::123456789012:role/kubernetes-worker-role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create S3 Policy for Pod and update trust with role from nodes in the CloudFormation stack. 



Step 5: Launch nodes with IAM Role attached to node. This will be the IAM role the node uses. Another one will be created for Pods. You will need to enter the IAM role name to the nodes.  

Step 6: Deploy kube2iam 

Securing pods on Kuberenetes:

Task 1 Create a Role for IAM:

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
Step 7: Deploy Application to test
```
apiVersion: v1
kind: Pod
metadata:
  name: s3
  labels:
    name: s3
  annotations:
    iam.amazonaws.com/role: <arn application uses>
spec:
  containers:
  - image: fstab/aws-cli
    command:
      - "/home/aws/aws/env/bin/aws"
      - "s3"
      - "ls"
      - "<your bucket name here>"
    name: s3
```

