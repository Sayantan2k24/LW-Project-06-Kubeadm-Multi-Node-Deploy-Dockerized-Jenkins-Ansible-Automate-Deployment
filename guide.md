# Kubernetes Cluster with Jenkins + Ansible Setup

## 1. Kubernetes Cluster Setup with kubeadm

### Master Node Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# Install kubeadm, kubelet, kubectl
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Initialize cluster
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Setup kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico network plugin
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
```

### Worker Node Setup

```bash
# Same initial setup as master (Docker, kubeadm, etc.)
# Then join the cluster using the command from master node init output:
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

## 2. RBAC Configuration

### ServiceAccount and ClusterRole

```yaml
# rbac-config.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-ansible-sa
  namespace: devops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-ansible-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "persistentvolumeclaims", "events", "configmaps", "secrets", "nodes"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "replicasets", "statefulsets"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: ["extensions"]
  resources: ["ingresses", "deployments"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-ansible-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-ansible-role
subjects:
- kind: ServiceAccount
  name: jenkins-ansible-sa
  namespace: devops
```

## 3. Namespace Creation

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops
```

## 4. Jenkins Deployment

```yaml
# jenkins-deployment.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /var/jenkins-data
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: devops
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-ansible-sa
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        env:
        - name: JENKINS_OPTS
          value: "--httpPort=8080"
        volumeMounts:
        - name: jenkins-storage
          mountPath: /var/jenkins_home
        - name: docker-sock
          mountPath: /var/run/docker.sock
        - name: kubectl-binary
          mountPath: /usr/local/bin/kubectl
        securityContext:
          runAsUser: 0
      volumes:
      - name: jenkins-storage
        persistentVolumeClaim:
          claimName: jenkins-pvc
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
      - name: kubectl-binary
        hostPath:
          path: /usr/bin/kubectl
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops
spec:
  selector:
    app: jenkins
  ports:
  - name: web
    port: 8080
    targetPort: 8080
    nodePort: 30080
  - name: jnlp
    port: 50000
    targetPort: 50000
  type: NodePort
```

## 5. Ansible Deployment

```yaml
# ansible-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ansible-inventory
  namespace: devops
data:
  hosts: |
    [k8s-nodes]
    k8s-master ansible_host=<MASTER_IP>
    k8s-worker1 ansible_host=<WORKER1_IP>
    k8s-worker2 ansible_host=<WORKER2_IP>
    
    [k8s-nodes:vars]
    ansible_user=ubuntu
    ansible_ssh_private_key_file=/etc/ansible/ssh-keys/id_rsa
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ansible
  namespace: devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ansible
  template:
    metadata:
      labels:
        app: ansible
    spec:
      serviceAccountName: jenkins-ansible-sa
      containers:
      - name: ansible
        image: cytopia/ansible:latest
        command: ["/bin/bash"]
        args: ["-c", "while true; do sleep 30; done"]
        volumeMounts:
        - name: ansible-playbooks
          mountPath: /etc/ansible/playbooks
        - name: ansible-inventory
          mountPath: /etc/ansible/inventory
        - name: ssh-keys
          mountPath: /etc/ansible/ssh-keys
        - name: kubectl-binary
          mountPath: /usr/local/bin/kubectl
        env:
        - name: ANSIBLE_HOST_KEY_CHECKING
          value: "False"
      volumes:
      - name: ansible-playbooks
        configMap:
          name: ansible-playbooks
      - name: ansible-inventory
        configMap:
          name: ansible-inventory
      - name: ssh-keys
        secret:
          secretName: ansible-ssh-keys
          defaultMode: 0600
      - name: kubectl-binary
        hostPath:
          path: /usr/bin/kubectl
---
apiVersion: v1
kind: Service
metadata:
  name: ansible-service
  namespace: devops
spec:
  selector:
    app: ansible
  ports:
  - port: 22
    targetPort: 22
  clusterIP: None
```

## 6. Ansible Playbooks ConfigMap

```yaml
# ansible-playbooks-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ansible-playbooks
  namespace: devops
data:
  deploy-app.yml: |
    ---
    - name: Deploy Application to Kubernetes
      hosts: localhost
      connection: local
      tasks:
        - name: Create deployment
          kubernetes.core.k8s:
            name: sample-app
            api_version: apps/v1
            kind: Deployment
            namespace: default
            definition:
              spec:
                replicas: 3
                selector:
                  matchLabels:
                    app: sample-app
                template:
                  metadata:
                    labels:
                      app: sample-app
                  spec:
                    containers:
                    - name: app
                      image: nginx:latest
                      ports:
                      - containerPort: 80
        
        - name: Create service
          kubernetes.core.k8s:
            name: sample-app-service
            api_version: v1
            kind: Service
            namespace: default
            definition:
              spec:
                selector:
                  app: sample-app
                ports:
                - port: 80
                  targetPort: 80
                type: ClusterIP

  update-nodes.yml: |
    ---
    - name: Update Kubernetes Nodes
      hosts: k8s-nodes
      become: yes
      tasks:
        - name: Update package cache
          apt:
            update_cache: yes
        
        - name: Upgrade packages
          apt:
            upgrade: dist
        
        - name: Restart kubelet if needed
          systemd:
            name: kubelet
            state: restarted
          when: ansible_facts['packages']['kubelet'] is defined

  scale-deployment.yml: |
    ---
    - name: Scale Kubernetes Deployment
      hosts: localhost
      connection: local
      vars:
        deployment_name: "{{ deployment | default('sample-app') }}"
        replica_count: "{{ replicas | default(5) }}"
      tasks:
        - name: Scale deployment
          kubernetes.core.k8s:
            name: "{{ deployment_name }}"
            api_version: apps/v1
            kind: Deployment
            namespace: default
            definition:
              spec:
                replicas: "{{ replica_count }}"
```

## 7. SSH Keys Secret (Generate first)

```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ansible-key -N ""

# Create secret
kubectl create secret generic ansible-ssh-keys \
  --from-file=id_rsa=ansible-key \
  --from-file=id_rsa.pub=ansible-key.pub \
  -n devops
```

## 8. Jenkins Pipeline Configuration

Create this Jenkinsfile in your repository:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-ansible-sa
  containers:
  - name: ansible
    image: cytopia/ansible:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: kubectl-binary
      mountPath: /usr/local/bin/kubectl
  volumes:
  - name: kubectl-binary
    hostPath:
      path: /usr/bin/kubectl
"""
        }
    }
    
    environment {
        KUBECONFIG = '/var/run/secrets/kubernetes.io/serviceaccount'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("myapp:${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Deploy with Ansible') {
            steps {
                container('ansible') {
                    script {
                        sh '''
                        # Install kubernetes collection
                        ansible-galaxy collection install kubernetes.core
                        
                        # Run deployment playbook
                        ansible-playbook -i /etc/ansible/inventory/hosts \
                          /etc/ansible/playbooks/deploy-app.yml \
                          -e "image_tag=${BUILD_NUMBER}"
                        '''
                    }
                }
            }
        }
        
        stage('Scale Application') {
            when {
                branch 'production'
            }
            steps {
                container('ansible') {
                    sh '''
                    ansible-playbook /etc/ansible/playbooks/scale-deployment.yml \
                      -e "replicas=5"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
```

## 9. Deployment Commands

```bash
# Create namespace
kubectl apply -f namespace.yaml

# Apply RBAC
kubectl apply -f rbac-config.yaml

# Create SSH keys secret
kubectl create secret generic ansible-ssh-keys \
  --from-file=id_rsa=ansible-key \
  --from-file=id_rsa.pub=ansible-key.pub \
  -n devops

# Deploy Jenkins
kubectl apply -f jenkins-deployment.yaml

# Create ConfigMaps
kubectl apply -f ansible-playbooks-configmap.yaml

# Deploy Ansible
kubectl apply -f ansible-deployment.yaml

# Check deployments
kubectl get pods -n devops
kubectl get services -n devops
```

## 10. Integration Flow

### Workflow:
1. **Developer pushes code** to Git repository
2. **Jenkins webhook triggers** pipeline
3. **Jenkins builds** Docker image
4. **Jenkins calls Ansible** pod via Kubernetes API
5. **Ansible executes playbooks** using ServiceAccount
6. **Ansible deploys/updates** applications on cluster
7. **Ansible reports back** to Jenkins
8. **Jenkins sends notifications**

### Key Integration Points:

- **ServiceAccount**: Both pods use same SA with cluster-wide permissions
- **Shared Storage**: Jenkins and Ansible can share playbooks via ConfigMaps
- **API Communication**: Ansible uses kubectl and K8s API directly
- **Pipeline Orchestration**: Jenkins orchestrates the entire CI/CD flow

## 11. Accessing Jenkins

```bash
# Get Jenkins initial password
kubectl exec -it deployment/jenkins -n devops -- cat /var/jenkins_home/secrets/initialAdminPassword

# Access Jenkins UI
http://<NODE_IP>:30080
```

## 12. Testing the Setup

```bash
# Test Ansible connectivity
kubectl exec -it deployment/ansible -n devops -- ansible k8s-nodes -i /etc/ansible/inventory/hosts -m ping

# Test kubectl access from Ansible pod
kubectl exec -it deployment/ansible -n devops -- kubectl get nodes

# Test manual playbook execution
kubectl exec -it deployment/ansible -n devops -- ansible-playbook /etc/ansible/playbooks/deploy-app.yml
```

This setup provides a complete CI/CD pipeline where Jenkins orchestrates deployments and Ansible handles the actual deployment tasks with proper RBAC permissions.