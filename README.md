# Kubernetes

Step 1:
Create VPC for Kubernetes:
CloudFormation: https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml

Step 2: Follow the Instructions to install Kubectl outline here:
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

Step 3: Deploy EKS Cluster:

Step 4: Create IAM Policy for Nodes:
IAM roles

It is necessary to create an IAM role which can assume other roles and assign it to each kubernetes worker.

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

The roles that will be assumed must have a Trust Relationship which allows them to be assumed by the kubernetes worker role. 

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

Step 5: Launch nodes with IAM ROles

Step 6: Deploy kube2iam 

Step 7: Deploy Application to test





Securing pods on Kuberenetes:

Task 1 Create a Role for IAM:
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
