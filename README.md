# Zero Trust LLM Gateway on EKS
 
A production-grade reference architecture for securing AI/LLM workloads on Amazon EKS. This project demonstrates defense in depth across every layer of the stack, from IAM and admission control to runtime threat detection and prompt injection filtering all provisioned with Terraform.
 
---
 
## Architecture Overview
 
```
Internet
    │
    ▼
[ AWS WAF ]
    │
    ▼
[ ALB Ingress ]
    │
    ▼
[ Kong / NGINX Gateway ]  ◄── Prompt Injection Sidecar
    │
    ▼
[ LLM Service Pod ]       ◄── Ollama (Mistral / LLaMA 3)
    │
    ▼
[ AWS Secrets Manager ]   ◄── Secrets Store CSI Driver
[ Amazon S3 / Bedrock ]   ◄── IRSA — scoped IAM role per pod
```
 
**Runtime:** Falco monitors all pods for anomalous behavior  
**Admission:** OPA/Gatekeeper enforces policy on every deployment  
**Network:** Cilium NetworkPolicies enforce zero-trust pod-to-pod traffic
 
---
 
## Security Layers
 
### 1. Workload Identity — IRSA
- IAM Roles for Service Accounts scoped to least privilege
- No static credentials in manifests, environment variables, or images
- Demonstrates a misconfigured role vs. a hardened one
### 2. Admission Control — OPA / Gatekeeper
- Rego policies enforce: no root containers, read-only root filesystem, required resource limits, no privileged escalation
- Blocked deployments shown with `kubectl` output examples
### 3. Network Segmentation — Cilium
- NetworkPolicies restrict LLM pod ingress to the gateway only
- Egress limited to Secrets Manager and S3 endpoints
- All other pod-to-pod traffic denied by default
### 4. Secret Management — Secrets Store CSI Driver
- API keys and model configuration mounted as volumes from AWS Secrets Manager
- No secrets in ConfigMaps, environment variables, or Helm values
### 5. Prompt Injection Detection
- Lightweight sidecar inspects incoming requests before they reach the model
- Pattern-based and heuristic filtering for common injection payloads
- Blocked requests logged to CloudWatch with request metadata
### 6. Runtime Threat Detection — Falco
- Custom rules alert on: shell exec inside LLM container, unexpected outbound connections, sensitive host path reads
- Alerts forwarded to CloudWatch Logs / SNS
---
 
## Repo Structure
 
```
.
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── modules/
│   │   ├── eks/              # EKS cluster, node groups, OIDC
│   │   ├── irsa/             # IAM roles for service accounts
│   │   ├── networking/       # VPC, subnets, security groups
│   │   ├── secrets/          # Secrets Manager + CSI driver
│   │   ├── gatekeeper/       # OPA Gatekeeper + Rego policies
│   │   ├── falco/            # Falco DaemonSet + custom rules
│   │   └── ingress/          # Kong/NGINX + prompt injection sidecar
├── k8s/
│   ├── llm/                  # Ollama deployment manifests
│   ├── network-policies/     # Cilium NetworkPolicy definitions
│   ├── gatekeeper-policies/  # ConstraintTemplates and Constraints
│   └── falco-rules/          # Custom Falco rule files
├── docs/
│   └── architecture.md       # Extended design notes
└── README.md
```
 
---
 
## Prerequisites
 
- AWS CLI configured with appropriate permissions
- Terraform >= 1.6
- `kubectl` >= 1.28
- `helm` >= 3.12
- An AWS account with EKS, IAM, Secrets Manager, and CloudWatch access
---
 
## Deployment
 
```bash
# 1. Clone the repo
git clone https://github.com/<your-handle>/zero-trust-llm-eks.git
cd zero-trust-llm-eks
 
# 2. Initialize Terraform
cd terraform
terraform init
 
# 3. Review the plan
terraform plan -var-file="vars.tfvars"
 
# 4. Apply
terraform apply -var-file="vars.tfvars"
 
# 5. Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name zero-trust-llm-cluster
 
# 6. Verify Gatekeeper policies are active
kubectl get constraints
 
# 7. Verify Falco is running
kubectl get pods -n falco
```
 
---
 
## Security Validation
 
Each layer includes a validation step to confirm controls are working.
 
**Test OPA policy enforcement — expect a rejection:**
```bash
kubectl apply -f k8s/tests/privileged-pod.yaml
# Error from server ([...]: pods "test-privileged" is forbidden)
```
 
**Test network policy — expect a timeout:**
```bash
kubectl exec -it <non-gateway-pod> -- curl http://llm-service:11434
# curl: (28) Connection timed out
```
 
**Test IRSA scoping — expect an access denied:**
```bash
kubectl exec -it <llm-pod> -- aws s3 ls s3://another-account-bucket
# An error occurred (AccessDenied)
```
 
**Trigger a Falco alert:**
```bash
kubectl exec -it <llm-pod> -- /bin/sh
# Falco alert fires → CloudWatch Logs / SNS notification
```
 
---
 
## Tools & Technologies
 
| Layer | Tool |
|---|---|
| Infrastructure | Terraform |
| Container Orchestration | Amazon EKS |
| LLM Runtime | Ollama (Mistral / LLaMA 3) |
| Workload Identity | IRSA (IAM Roles for Service Accounts) |
| Admission Control | OPA Gatekeeper |
| Network Policy | Cilium |
| Secret Management | AWS Secrets Manager + Secrets Store CSI Driver |
| Runtime Security | Falco |
| Ingress / Gateway | Kong or NGINX |
| Observability | CloudWatch Logs, CloudWatch Metrics, SNS |
 
---
 
## Blog Post
 
This project is documented in detail in the accompanying blog post:  
**[Securing LLM Workloads on EKS: A Zero Trust Approach](#)** *(link coming soon)*
 
 
---
 
## License
 
MIT