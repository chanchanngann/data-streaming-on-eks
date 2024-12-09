# data-streaming-on-k8s
# Intro
This exercise is to build a data streaming pipeline using Kubernetes via AWS EKS. The components of the data pipeline are:
- Nifi
- Kafka
- Spark Streaming
- Snowflake
- ...
# Architecture

Data flow:
![](../images/flow2.jpg) 

AWS components:
![](../images/flow1.jpg)
**VPC:**
    - **4 subnets (2 public, 2 private)** are set up for high availability and redundancy across 2 different Availability Zones (AZ).
    - **Internet Gateway (IGW)** enables internet connectivity for resources in the public subnets.
	- The EKS cluster spans across 4 subnets (2 public + 2 private subnets).
**Public Subnets:**
	- used for internet-facing resources.
	- **Application load balancer (ALB)** gets direct internet access through IGW.
	- **NAT Gateway** to allow internet access for the resources in the private subnets. For example, EBS CSI driver requires outbound internet access to interact with AWS API to provision EBS volumes; load balancer controller communicates with AWS API to provision ALB.
**Private Subnets:**
	- to ensure **Node groups** (k8s worker nodes) are not exposed directly to the internet.
	- **Application pods** (Nifi, Kafka, Spark) are deployed on the private worker nodes
	- Common resources **EBS CSI driver pods** and **load balancer controller pods** are also deployed on the private worker nodes.
# Prerequisites
Install the followings in advance.
- aws CLI
- eksctl
- kubectl
- helm

# Part 1 - EKS cluster + Node Group

1. Launch EKS cluster using the given template `eks.yaml`, replacing `ap-northeast-2` with desired aws region.
```
aws cloudformation create-stack --stack-name MyEKSClusterStack --template-body file://project_folder/cloudformation/eks.yaml --capabilities CAPABILITY_NAMED_IAM --region ap-northeast-2
```

2. Wait until EKS cluster is ready. From its Cloudformation output, we copy a few values:
   `VpcId, ControlPlaneSecurityGroup, PrivateSubnetIds`
   Then paste the values to `params.json`. This file contains the input parameters required to create the node group.  

3. Launch the node group using the given template `nodegroup-private.yaml`.
```
aws cloudformation create-stack --stack-name MyPrivateNodeGroupStack --template-body file://project_folder/cloudformation/nodegroup-private.yaml --parameters file://project_folder/cloudformation/params.json --capabilities CAPABILITY_NAMED_IAM --region ap-northeast-2
```

4. Wait until the node group is ready. Then create kubeconfig file. This file enables the `kubctl` CLI to communicate w/ the EKS cluster.
```
aws eks update-kubeconfig --region ap-northeast-2 --name MyEKSCluster
```

5. To enable the nodes to join the cluster, prepare an aws-auth ConfigMap. Follow aws doc instruction to create `aws-auth-cm.yaml`. Replace the `rolearn` value with the `NodeGroupRole Arn` value obtained from Cloudformation output (MyPrivateNodeGroupStack). Then check and wait until the nodes are ready.
```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml

kubectl apply -f project_folder/auth/aws-auth-cm.yaml

kubectl get nodes --watch
```
![](../images/nodes.PNG)

---
# Part 2 - EBS CSI driver and Load Balancer Controller
### EBS CSI driver
To enable Dynamic Volume Provisioning, install EBS CSI driver in advance.

1. Create OIDC for the EKS cluster
```
eksctl utils associate-iam-oidc-provider --cluster MyEKSCluster --approve
```
2. Create service account
```
eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster MyEKSCluster  --role-name AmazonEKS_EBS_CSI_DriverRole --role-only --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy  --approve   
```
3. Helm install EBS CSI Driver.
```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update  

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver --namespace kube-system  --set serviceAccount.controller.create=false  --set serviceAccount.controller.name=ebs-csi-controller-sa
```
### AWS Load Balancer Controller
The controller manages AWS Elastic Load Balancers for a Kubernetes cluster. We need this controller to help create ALB so that nifi can be accessed via ingress entry.

1. Create OIDC (can skip since we have created the OIDC in CSI Driver section.)
```
eksctl utils associate-iam-oidc-provider --cluster MyEKSCluster --approve
```
2. Download & create IAM policy required for the load balancer controller to interact with AWS.
   (Go  [[#Troubleshoot]] section, there are missing policies in the aws provided policy.)
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://project_folder/auth/iam_policy.json
```
3. Create service account. The eksctl command will create service account `aws-load-balancer-controller` and IAM role `AmazonEKSLoadBalancerControllerRole`. The service account will be linked up with the IAM role, attaching the IAM policy `AWSLoadBalancerControllerIAMPolicy`(copy the policy ARN from aws IAM).
```
eksctl create iamserviceaccount  --cluster=MyEKSCluster --namespace=kube-system --name=aws-load-balancer-controller  --role-name AmazonEKSLoadBalancerControllerRole  --attach-policy-arn=arn:aws:iam::xxxxxxxx:policy/AWSLoadBalancerControllerIAMPolicy  --approve  
```

4. Helm install aws-load-balancer-controller
```
helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller  -n kube-system --set clusterName=MyEKSCluster  --set serviceAccount.create=false  --set serviceAccount.name=aws-load-balancer-controller
```

5. Verify if the controllers are installed properly. Check out `aws-load-balancer-controller` and `aws-ebs-csi-driver` from the output.
```
kubectl get deployment -n kube-system 
```
![](../images/controllers.PNG)

### Troubleshoot
Permission error: Missing IAM policies for Load Balancer Controller
```
Failed deploy model due to operation error Elastic Load Balancing v2: DescribeListenerAttributes, https response error StatusCode: 403, RequestID: xxxxx-xxx-xxx-xxx-xxxx, api error AccessDenied: User: arn:aws:sts::xxxxxxxx:assumed-role/AmazonEKSLoadBalancerControllerRole/xxxxxxxxxx is not authorized to perform: elasticloadbalancing:DescribeListenerAttributes because no identity-based policy allows the elasticloadbalancing:DescribeListenerAttributes action
```

Solution:
This seems a bug (not sure). You need to add the following policies to `iam_policy.json` , recreating the IAM policy and recreating service account `aws-load-balancer-controller`.
```
Action:
    - elasticloadbalancing:DescribeListenerAttributes
    - elasticloadbalancing:ModifyListenerAttributes
Resource: "*"
```
ref: https://github.com/eksctl-io/eksctl/issues/7987

(Details: Delete the existing `eksctl-MyEKSCluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller` cloudformation stack -> delete & recreate IAM policy -> eksctl create service account with IAM role attached with updated policy -> recreate AWS Load Balancer Controller using `helm` with the updated service account)

---
# Part 3 - Nifi
Set up a standalone Nifi application and here I am gonna use HTTPS approach to access Nifi.
My hostname is `rachel.nifi.com`.

1. Create namespace
```
kubectl create namespace nifi
```
2. Create secret with `tls.key` and `tls.cert`.
```
openssl genrsa -out tls.key 2048

openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj "/CN=rachel.nifi.com"

kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key -n nifi
```
Remarks: If `openssl.cnf` is not avail, you can download a sample OpenSSL configuration file from [OpenSSL GitHub](https://github.com/openssl/openssl/blob/master/apps/openssl.cnf) and add the configuration file using `-config` flag.
```
openssl req -new -x509 -key tls.key -out tls_r.cert -days 360 -subj "/CN=rachel.nifi.com" -config folder/to/openssl.cnf
```

3. Create the essential objects: statefulset, service, pvc, sc, ingress.
```
kubectl apply -f nifi/
```
![](../images/nifi_objects.PNG)
4. Review the setup and copy username and password from the pod logs
```
kubectl -n nifi get pods
kubectl -n nifi logs nifi-0 --tail 50
```
![](../images/nifi_pod.PNG)

From pod logs:
![](../images/nifi_credentials.PNG)

5. Get the DNS name of the ingress (ALB) and check for the public IPs of the ALB. 
```
kubectl get ingress -n nifi
nslookup k8s-nifi-xxxxx-xxxxx-xxxxx.ap-northeast-2.elb.amazonaws.com
```
Let say the public IPs are 12.34.56.78 and 22.34.56.78 from `nslookup` output.
Copy the IPs and the Nifi hostname to the local hosts file (located at `/etc/hosts` for mac)
```
12.34.56.78 rachel.nifi.com
22.34.56.78 rachel.nifi.com
```
6. @AWS console, check out the `Load Balancers` page. (we have implemented the load balancer for nifi in step 3.)
- make sure listener has included `443`
- import certificate `tls.key` & `tls.cert` to the listener 443 (import to ACM option)
- make sure load balancer security group inbound rule included HTTPS 443 traffic

7. All done. Go to the chrome browser and login with the generated username and password obtained from the pod logs.
```
https://rachel.nifi.com/nifi
```
![](../images/nifi_login_page.PNG)


### Troubleshoot

Permission error: failed to create data folder within nifi pod container. 
Details later.
# Part 4 - Kafka
Details later.

# Part 5 - Spark Streaming
Details later.

# Part 6 - Write to Snowflake
Details later.

# References
- https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
- https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
- https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html