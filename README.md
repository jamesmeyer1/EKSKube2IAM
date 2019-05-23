# Securing Pods for EKS

## Goals of the Workshop
This workshop will walk through the process of securing Kubernete Pods running on Amazon Elastic Kubernetes Service (EKS). We will setup a VPC with the proper tagging, configure kubectl locally, and create the trusts, role, and IAM policies that are required to apply IAM to a pod. 

## Step 1:
Create a VPC for Kubernetes. When you create your Amazon EKS cluster, Amazon EKS tags the VPC containing the subnets you specify in the following way so that Kubernetes can discover it. You can read about the subnet and VPC tagging performed here.

CloudFormation: https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml 

## Step 2: Deploy EKS Cluster:
1) Check the region in the Management Console. We are deploying in US-EAST-2. 

## Step 3: Update AWS ClI to latest version:
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html

## Step 4: Follow the Instructions to install Kubectl outline here:
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

## Step 5: 
Use the AWS CLI update-kubeconfig command to create or update your kubeconfig for your cluster. 
aws eks --region region update-kubeconfig --name cluster_name
kubectl get svc

## Step 6: Create IAM Policy for Nodes:
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

Step 5: Launch nodes with IAM Role attached to node. This will be the IAM role the node uses. Another one will be created for Pods. You will need to enter the IAM role name to the nodes.  

Step 6: Deploy kube2iam 

Step 7: Deploy Application to test





Securing pods on Kuberenetes:

Task 1 Create a Role for IAM:
```
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
            - "--app-port=8181"
            - "--base-role-arn=arn:aws:iam::794843820546:role/kube2iamrole"
            - "--iptables=true"
            - "--host-ip=$(HOST_IP)"
            - "--host-interface=weave"
            - "--verbose"
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          ports:
            - containerPort: 8181
              hostPort: 8181
              name: http
          securityContext:
            privileged: true
```
