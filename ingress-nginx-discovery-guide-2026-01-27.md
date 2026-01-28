# Ingress-NGINX Discovery Guide for AWS EKS
**Created:** January 27, 2026  
**Purpose:** Find all ingress-nginx deployments in AWS environment before March 2026 migration deadline

---

## Prerequisites

Ensure you have the following tools installed:
- AWS CLI
- kubectl
- (Optional) jq for JSON parsing

---

## Step 1: Discover All EKS Clusters

### List all EKS clusters in your default region
```bash
aws eks list-clusters
# This shows all EKS cluster names in your default AWS region
```

### List clusters in a specific region
```bash
aws eks list-clusters --region us-east-1
# Replace us-east-1 with your desired region (us-west-2, eu-west-1, etc.)
```

### List clusters across ALL regions
```bash
for region in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "Checking region: $region"
  aws eks list-clusters --region $region --output table
done
# This loops through every AWS region and lists EKS clusters in each
# Helps ensure you don't miss clusters in unexpected regions
```

---

## Step 2: Connect to Each Cluster

### Update kubeconfig for a specific cluster
```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
# Example: aws eks update-kubeconfig --name production-cluster --region us-east-1
# This configures kubectl to communicate with the specified EKS cluster
# Replace <cluster-name> with actual cluster name from Step 1
```

### Verify connection
```bash
kubectl cluster-info
# Confirms you're connected to the correct cluster
# Shows the Kubernetes master and services endpoints
```

### Check current context
```bash
kubectl config current-context
# Shows which cluster you're currently connected to
# Useful when switching between multiple clusters
```

---

## Step 3: Find Ingress-NGINX Installations

### Check for ingress-nginx pods
```bash
kubectl get pods --all-namespaces | grep -i nginx
# Searches for any pods with "nginx" in their name across all namespaces
# Ingress-nginx pods typically have names like "ingress-nginx-controller-xxxxx"
```

### Find ingress-nginx specifically by label
```bash
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
# Uses Kubernetes labels to find official ingress-nginx installations
# More accurate than grep as it uses the standard ingress-nginx labels
```

### Check all namespaces for ingress-nginx
```bash
kubectl get all --all-namespaces -l app.kubernetes.io/name=ingress-nginx
# Shows ALL resources (pods, services, deployments) related to ingress-nginx
# Gives a complete picture of the ingress-nginx installation
```

### List ingress-nginx deployments
```bash
kubectl get deployments --all-namespaces | grep -i nginx
# Shows deployment configurations for nginx-based controllers
# Deployments manage the pod replicas and updates
```

---

## Step 4: Check Ingress Resources

### List all Ingress resources
```bash
kubectl get ingress --all-namespaces
# Shows all ingress rules across all namespaces
# Each ingress defines routing rules for external traffic
```

### Get detailed ingress information
```bash
kubectl get ingress --all-namespaces -o wide
# Shows additional details like hosts, addresses, and ports
# The "CLASS" column shows which ingress controller handles each resource
```

### Describe specific ingress resources
```bash
kubectl describe ingress --all-namespaces
# Provides detailed information about each ingress including:
# - Rules and paths
# - Backend services
# - TLS configuration
# - Events and status
```

---

## Step 5: Check Ingress Classes

### List all ingress classes
```bash
kubectl get ingressclass
# Shows available ingress controllers in the cluster
# Look for "nginx" or "ingress-nginx" in the NAME or CONTROLLER columns
```

### Get detailed ingress class information
```bash
kubectl get ingressclass -o yaml
# Shows full configuration of each ingress class
# The "controller" field indicates if it's using ingress-nginx
# Example: controller: k8s.io/ingress-nginx
```

---

## Step 6: Check Helm Installations

### List all Helm releases
```bash
helm list --all-namespaces
# Shows all applications installed via Helm
# Look for releases named "ingress-nginx" or similar
```

### Search for nginx in Helm releases
```bash
helm list --all-namespaces | grep -i nginx
# Filters Helm releases to show only nginx-related installations
```

### Get details of ingress-nginx Helm release
```bash
helm get values ingress-nginx -n <namespace>
# Shows the configuration values used for the Helm installation
# Replace <namespace> with the namespace where ingress-nginx is installed
# Common namespaces: ingress-nginx, kube-system, default
```

---

## Step 7: Check Services and Load Balancers

### List services associated with ingress-nginx
```bash
kubectl get svc --all-namespaces | grep -i nginx
# Shows Kubernetes services that route traffic to nginx pods
# Type "LoadBalancer" indicates an AWS ELB/NLB is created
```

### Get detailed service information
```bash
kubectl get svc --all-namespaces -l app.kubernetes.io/name=ingress-nginx -o wide
# Shows services with their external IPs and load balancer details
# The EXTERNAL-IP column shows the AWS load balancer DNS name
```

### Find AWS load balancers created by ingress-nginx
```bash
kubectl get svc --all-namespaces -o json | grep -i "hostname"
# Extracts load balancer hostnames from service configurations
# These correspond to actual AWS ELB/NLB resources
```

---

## Step 8: Check ConfigMaps

### List ingress-nginx ConfigMaps
```bash
kubectl get configmap --all-namespaces | grep -i nginx
# ConfigMaps store ingress-nginx configuration settings
# Common names: ingress-nginx-controller, nginx-configuration
```

### View ingress-nginx configuration
```bash
kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml
# Shows the actual configuration parameters for ingress-nginx
# Replace namespace (-n) if ingress-nginx is in a different namespace
```

---

## Step 9: Generate Comprehensive Report

### Create a detailed inventory of all ingress-nginx resources
```bash
# Save cluster name
CLUSTER_NAME=$(kubectl config current-context)

# Create output file
OUTPUT_FILE="ingress-nginx-inventory-$(date +%Y%m%d-%H%M%S).txt"

# Gather all information
echo "=== CLUSTER: $CLUSTER_NAME ===" > $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

echo "=== INGRESS-NGINX PODS ===" >> $OUTPUT_FILE
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

echo "=== ALL INGRESS RESOURCES ===" >> $OUTPUT_FILE
kubectl get ingress --all-namespaces -o wide >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

echo "=== INGRESS CLASSES ===" >> $OUTPUT_FILE
kubectl get ingressclass >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

echo "=== INGRESS-NGINX SERVICES ===" >> $OUTPUT_FILE
kubectl get svc --all-namespaces -l app.kubernetes.io/name=ingress-nginx >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

echo "=== HELM RELEASES ===" >> $OUTPUT_FILE
helm list --all-namespaces | grep -i nginx >> $OUTPUT_FILE

echo "Report saved to: $OUTPUT_FILE"
```
**This script creates a timestamped report file containing:**
- Current cluster context
- All ingress-nginx pods
- All ingress resources
- Ingress classes
- Related services
- Helm releases

---

## Step 10: Cross-Reference with AWS Resources

### Find ELB/NLB created by ingress-nginx
```bash
aws elbv2 describe-load-balancers --region <region> | grep -i kubernetes
# Lists all load balancers in the region
# Ingress-nginx typically creates load balancers with "kubernetes" in tags
```

### Find Classic Load Balancers
```bash
aws elb describe-load-balancers --region <region>
# Lists older Classic Load Balancers (ELB)
# Some older ingress-nginx installations use these
```

### Check load balancer tags
```bash
aws elbv2 describe-tags --resource-arns <load-balancer-arn>
# Shows tags on a specific load balancer
# Look for tags like "kubernetes.io/service-name" or "ingress-nginx"
```

---

## Quick Discovery Script (Run This First)

### All-in-one discovery script
```bash
#!/bin/bash
# Quick discovery script for ingress-nginx across all EKS clusters

echo "Starting ingress-nginx discovery..."
echo "=================================="

# Get all regions
for region in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo ""
  echo "Checking region: $region"
  
  # Get clusters in region
  clusters=$(aws eks list-clusters --region $region --query 'clusters' --output text)
  
  if [ -n "$clusters" ]; then
    for cluster in $clusters; do
      echo "  Cluster: $cluster"
      
      # Update kubeconfig
      aws eks update-kubeconfig --name $cluster --region $region --alias "$cluster-$region" > /dev/null 2>&1
      
      # Check for ingress-nginx
      nginx_pods=$(kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --context "$cluster-$region" 2>/dev/null | grep -v NAME)
      
      if [ -n "$nginx_pods" ]; then
        echo "    ⚠️  FOUND ingress-nginx!"
        echo "    Pods:"
        echo "$nginx_pods" | sed 's/^/      /'
      else
        echo "    ✓ No ingress-nginx found"
      fi
    done
  fi
done

echo ""
echo "=================================="
echo "Discovery complete!"
```
**What this script does:**
1. Loops through all AWS regions
2. Finds all EKS clusters in each region
3. Connects to each cluster
4. Checks for ingress-nginx installations
5. Reports findings with clear indicators

**To use:** Save as `discover-ingress-nginx.sh`, make executable with `chmod +x discover-ingress-nginx.sh`, then run `./discover-ingress-nginx.sh`

---

## Common Namespaces to Check

Ingress-nginx is commonly installed in these namespaces:
- `ingress-nginx` (most common)
- `kube-system`
- `default`
- `ingress`
- `nginx-ingress`

### Check specific namespace
```bash
kubectl get all -n ingress-nginx
# Shows all resources in the ingress-nginx namespace
# Replace with other namespace names as needed
```

---

## Troubleshooting

### If you don't see any results:

1. **Verify kubectl connection:**
   ```bash
   kubectl get nodes
   # Should list cluster nodes - if not, check kubeconfig
   ```

2. **Check if you're in the right context:**
   ```bash
   kubectl config get-contexts
   # Lists all available cluster contexts
   # Asterisk (*) indicates current context
   ```

3. **Switch context if needed:**
   ```bash
   kubectl config use-context <context-name>
   # Switches to a different cluster
   ```

4. **Verify AWS credentials:**
   ```bash
   aws sts get-caller-identity
   # Shows current AWS identity and account
   # Confirms AWS CLI is properly configured
   ```

---

## Next Steps After Discovery

Once you've identified where ingress-nginx is used:

1. **Document findings:** Create an inventory of all clusters, namespaces, and ingress resources
2. **Identify applications:** Map which applications/teams use each ingress
3. **Plan migration:** Schedule migration windows with application owners
4. **Choose alternative:** Decide between AWS Load Balancer Controller or Istio (see alternatives below)
5. **Test in non-prod:** Migrate dev/staging environments first
6. **Monitor:** Set up alerts for any remaining ingress-nginx usage

---

## Migration Alternatives

### Option 1: AWS Load Balancer Controller (NLB/ALB) - **Recommended for Most EKS Workloads**

The AWS Load Balancer Controller is the recommended replacement for most use cases. It provides native integration with AWS load balancing services.

#### Why Choose AWS Load Balancer Controller?
- Native AWS integration
- Better performance and cost optimization
- Automatic integration with AWS services (WAF, Shield, Certificate Manager)
- Officially supported by AWS
- Simpler architecture (no extra pods managing load balancers)

#### Sub-options:

**A. Ingress - AWS Load Balancer Controller with NLB (Network Load Balancer)**
- **Use for:** Layer 4 (TCP/UDP) load balancing
- **Best for:** High performance, low latency requirements
- **Features:** Ultra-low latency, static IP addresses, handles millions of requests per second
- **Documentation:** [Ingress - AWS Load Balancer Controller with NLB](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/#nlb)

```bash
# Check if you need NLB characteristics:
# - Need static IP addresses?
# - Require extreme performance (millions of requests/sec)?
# - Using non-HTTP protocols (TCP/UDP)?
```

**B. Ingress - AWS Load Balancer Controller with ALB (Application Load Balancer)**
- **Use for:** Layer 7 (HTTP/HTTPS) load balancing
- **Best for:** Most web applications and microservices
- **Features:** 
  - Advanced routing (host-based, path-based)
  - SSL/TLS termination
  - WAF integration
  - Authentication (Cognito, OIDC)
  - Better for cost optimization with HTTP traffic
- **Documentation:** [Ingress - AWS Load Balancer Controller with ALB](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/#alb)

```bash
# ALB is recommended if you:
# - Route HTTP/HTTPS traffic
# - Need advanced routing rules
# - Want WAF protection
# - Need authentication at load balancer level
```

**C. AWS Load Balancer Controller for EKS (General Documentation)**
- **Overview:** Complete guide for setting up the controller
- **Documentation:** [AWS Load Balancer Controller for EKS](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)

#### Installation Steps for AWS Load Balancer Controller:

```bash
# 1. Create IAM policy for the controller
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# 2. Create IAM role and service account
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# 3. Install the controller using Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# 4. Verify installation
kubectl get deployment -n kube-system aws-load-balancer-controller
```

#### Migration Example - From ingress-nginx to ALB:

**Before (ingress-nginx):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

**After (AWS ALB):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

---

### Option 2: Istio Ingress Gateway - **For Advanced Traffic Management**

Istio is recommended for complex use cases requiring advanced traffic management, observability, and service mesh capabilities.

#### Why Choose Istio?
- Advanced traffic management (canary deployments, A/B testing, traffic splitting)
- Built-in observability and tracing
- Service-to-service security (mTLS)
- Gateway API support (newer Kubernetes standard)
- Circuit breaking and resilience patterns
- Multi-cluster support

#### When to Use Istio:
- You need sophisticated traffic routing (blue-green, canary)
- Require service mesh features (mTLS, circuit breaking)
- Need advanced observability across services
- Planning to implement Gateway API
- Have microservices that need traffic management between services

#### When NOT to Use Istio:
- Simple web applications with basic routing needs
- Small teams without service mesh expertise
- Limited operational capacity (Istio adds complexity)
- Cost-sensitive environments (higher resource overhead)

#### Documentation:
- **[Istio Guide](https://istio.io/latest/docs/)** - Official Istio documentation
- **[Istio on EKS](https://aws.amazon.com/blogs/opensource/getting-started-with-istio-on-amazon-eks/)** - AWS-specific guide

#### Installation Steps for Istio:

```bash
# 1. Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# 2. Install Istio with default profile
istioctl install --set profile=default -y

# 3. Enable sidecar injection for your namespace
kubectl label namespace default istio-injection=enabled

# 4. Verify installation
kubectl get pods -n istio-system
```

#### Migration Example - From ingress-nginx to Istio Gateway:

**Before (ingress-nginx):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

**After (Istio Gateway + VirtualService):**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-app-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.example.com"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-vs
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - my-app-gateway
  http:
  - route:
    - destination:
        host: my-app-service
        port:
          number: 80
```

---

## Decision Matrix: Which Alternative to Choose?

| Criteria | AWS Load Balancer Controller | Istio Ingress Gateway |
|----------|------------------------------|----------------------|
| **Complexity** | Low - Simple setup | High - Requires service mesh knowledge |
| **Cost** | Lower - Pay for AWS LB only | Higher - Additional infrastructure |
| **Performance** | Excellent - Native AWS | Good - Additional proxy layer |
| **Use Case** | Standard web apps & APIs | Advanced microservices |
| **Traffic Management** | Basic routing | Advanced (canary, traffic splitting) |
| **Observability** | Basic | Advanced (built-in tracing) |
| **Team Expertise Required** | AWS & Kubernetes | Istio & Service Mesh |
| **Maintenance Overhead** | Low | Medium to High |
| **AWS Integration** | Native | Requires additional config |
| **Recommendation** | ✅ **Most teams** | Specific advanced use cases |

---

## Recommended Approach

### For Most Teams:
1. **Start with AWS Load Balancer Controller (ALB)** for 80-90% of use cases
2. Use NLB only when you specifically need Layer 4 or static IPs
3. Evaluate Istio only if you have clear requirements for:
   - Advanced traffic management
   - Service mesh features
   - Complex microservices architecture

### Migration Priority:
1. **Phase 1:** Simple applications → AWS ALB (quickest wins)
2. **Phase 2:** Performance-critical apps → AWS NLB if needed
3. **Phase 3:** Complex microservices → Evaluate Istio if requirements justify it

---

## Important Deadline

**⚠️ All migrations must be completed by March 2026**

After this date, ingress-nginx will no longer receive security updates or bug fixes.

---

## Resources

- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Ingress-NGINX to ALB Migration Guide](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
- [Istio Ingress Gateway Guide](https://istio.io/latest/docs/tasks/traffic-management/ingress/)

---

**Document Version:** 1.0  
**Last Updated:** January 27, 2026
