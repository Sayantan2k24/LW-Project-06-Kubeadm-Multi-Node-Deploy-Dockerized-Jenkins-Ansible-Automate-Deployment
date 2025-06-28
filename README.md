# LW-Project-06-Kubeadm-Multi-Node-Deploy-Dockerized-Jenkins-Ansible-Automate-Deployment
## Check the Detailed BLOG for understanding the hands on part:
### https://sayantansamanta098.medium.com/project-06-end-to-end-kubernetes-kubeadm-ci-cd-with-dockerized-jenkins-and-ansible-e2a7b8dd5169

## Architecture Flow
Here's how the components will work together:
Flow: Developer pushes code → Jenkins (CI/CD orchestrator) → Triggers Ansible playbooks → Ansible deploys to Kubernetes → Uses ServiceAccount with proper RBAC

## Integration Flow Summary
Here's how everything works together:

1. Architecture Overview:
Jenkins Pod: CI/CD orchestrator, triggers on code changes
Ansible Pod: Configuration management and deployment executor
ServiceAccount: Shared identity with cluster-wide permissions
RBAC: ClusterRole with necessary permissions for both pods

2. Deployment Flow:
Code Push → Jenkins Pipeline → Ansible Playbook → Kubernetes Deployment

4. Key Integration Points:
Shared ServiceAccount: Both Jenkins and Ansible pods use jenkins-ansible-sa with proper RBAC permissions
API Server Access: ServiceAccount token automatically mounted at /var/run/secrets/kubernetes.io/serviceaccount/
kubectl Access: Both pods have kubectl binary mounted from host
Ansible Playbooks: Stored in ConfigMaps, accessible to both pods
Communication: Jenkins triggers Ansible via Kubernetes Jobs or direct pod execution

4. Why This Design:
Jenkins: Handles CI/CD orchestration, webhooks, notifications, and pipeline management
Ansible: Executes actual deployments, configuration management, and infrastructure tasks
Separation of Concerns: Jenkins focuses on workflow, Ansible on execution
Scalability: Can spin up multiple Ansible execution pods as needed



