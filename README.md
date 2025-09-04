Objective

Deploy and scale a containerized NGINX application on Amazon EKS using:

eksctl for cluster provisioning

AWS Load Balancer Controller for Ingress

AWS Certificate Manager (ACM) for TLS (optional)

Horizontal Pod Autoscaler (HPA)

Helm for controller setup

curl, ab (Apache Bench) for testing scaling behavior

✅ Prerequisites

AWS CLI (aws configure must be set)

kubectl (v1.27+ recommended)

eksctl

helm

jq, curl, ab (Apache Bench)

Admin access on your system (to edit /etc/hosts on Linux/macOS or C:\Windows\System32\drivers\etc\hosts on Windows)

🏗️ Step-by-Step Setup
1. Set Environment Variables
export CLUSTER_NAME=demo-cluster
export AWS_REGION=us-west-2
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

2. Create EKS Cluster
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodegroup-name demo-nodegroup \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

3. Extract VPC ID for ALB Controller
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:eksctl.cluster.k8s.io/v1alpha1/cluster-name,Values=$CLUSTER_NAME" \
  --query "Vpcs[0].VpcId" \
  --output text)

4. Configure IAM OIDC Provider for EKS
Get OIDC ID:
oidc_id=$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

Check if already associated:
aws iam list-open-id-connect-providers | grep $oidc_id

If not, associate it:
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve

5. Download & Create IAM Policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

6. Create IAM Role & Kubernetes ServiceAccount
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

7. Deploy AWS Load Balancer Controller (Helm)
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName="$CLUSTER_NAME" --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region="$AWS_REGION" --set vpcId="$VPC_ID"

8. Install Metrics Server (for HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.0/components.yaml
kubectl get deployment metrics-server -n kube-system

9. Deploy App & Resources
kubectl create namespace demo

kubectl apply -f service.yaml
kubectl apply -f deployment.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml

10. Verify HPA Behavior
kubectl get hpa -n demo
kubectl describe hpa nginx-hpa -n demo


You should see currentAverageUtilization increase and replicas scale based on CPU.

11. Test DNS & Access
nslookup <your-load-balancer-dns>.elb.us-west-2.amazonaws.com
curl http://<resolved-ip>    # should show NGINX welcome page

12. Add Domain to Hosts File (for Ingress Host Mapping) # TLS certificate can be issued through ACM in AWS or through Lets Encrypt - will be shown in the live demo. 

Example:

192.168.104.252 example.com


Add the IP from nslookup to hosts file:

Windows: C:\Windows\System32\drivers\etc\hosts

Linux/macOS: /etc/hosts

Then visit: http://example.com

13. Load Test to Trigger Autoscaling
ab -n 100000 -c 500 http://<loadbalancer-dns>.us-west-2.elb.amazonaws.com:80/


Then check autoscaling:

kubectl describe hpa nginx-hpa -n demo
kubectl get pods -n demo

📤 Clean Up
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION

🛡️ Disaster Recovery Strategy (DR)

To extend this setup for disaster recovery:

Multi-AZ: EKS by default supports highly available nodegroups across multiple availability zones.

Multi-Region DR:

Create a secondary EKS cluster in a different region.

Mirror the IaC and Kubernetes resources via GitOps or CI/CD pipelines.

Use Route53 for DNS-level failover or latency-based routing.

Store container images in regional ECR for replication.

📁 File Structure
.
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── hpa.yaml
├── iam_policy.json
├── README.md

✅ Helpful Commands
kubectl config current-context
kubectl get pods -n demo
kubectl get svc -n demo
kubectl get ingress -n demo
kubectl describe hpa nginx-hpa -n demo

DR Strategy

To extend this Kubernetes deployment for disaster recovery, I would architect the setup to span multiple availability zones (AZs) within the chosen cloud provider. Deploying the application across multiple AZs ensures high availability and fault tolerance by distributing pods and services so that if one AZ experiences an outage, the workload continues running in the others. This involves configuring the Kubernetes cluster nodes across AZs and using cloud provider-specific load balancers that can route traffic to healthy pods in any AZ. Additionally, persistent storage (if needed) should leverage multi-AZ storage solutions, such as AWS EBS with replication or Azure Managed Disks with zone redundancy, to prevent data loss.

For more comprehensive disaster recovery, I would consider a multi-region deployment. This involves provisioning Kubernetes clusters in two or more geographically separated regions and replicating application state and data between them, for example using cross-region database replication or storage sync services. Traffic management can be handled using global DNS load balancing services like AWS Route 53 or Azure Traffic Manager to route users to the nearest healthy region. In case of a regional failure, failover to the secondary region would ensure business continuity with minimal downtime. Automating synchronization and failover with infrastructure-as-code tools and CI/CD pipelines would also be critical to maintain consistency and rapid recovery.
