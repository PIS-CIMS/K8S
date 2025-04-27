# K8S

ansible-playbook -i hosts.ini docker_install.yml --ask-become-pass

ansible-playbook -i hosts.ini k8s_install.yml --ask-become-pass

🚀 Detailed Implementation Steps:
✅ 1. Provision Infrastructure
Ensure you have at least 3 machines (can be cloud VMs like AWS EC2, Azure, DigitalOcean, or local VMs with VirtualBox/VMware):

Hostname	Role	IP Address
k8s-master-01	Control	192.168.0.101
k8s-worker-01	Worker	192.168.0.102
k8s-worker-02	Worker	192.168.0.103
✅ 2. Ansible Configuration
Create a simple Ansible inventory file (hosts.ini):

ini
Kopiera
Redigera
[k8s-master]
k8s-master-01 ansible_host=192.168.0.101

[k8s-workers]
k8s-worker-01 ansible_host=192.168.0.102
k8s-worker-02 ansible_host=192.168.0.103

[k8s-cluster:children]
k8s-master
k8s-workers

[all:vars]
ansible_python_interpreter=/usr/bin/python3
✅ 3. Install Docker using Ansible
Create docker_install.yml:

yaml
Kopiera
Redigera
---
- hosts: k8s-cluster
  become: true
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install prerequisites
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    - name: Add Docker’s GPG key
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

    - name: Set up Docker repo
      shell: |
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
          https://download.docker.com/linux/ubuntu \
          $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    - name: Install Docker Engine
      apt:
        name: docker-ce
        update_cache: yes
        state: present

    - name: Enable Docker service
      systemd:
        name: docker
        state: started
        enabled: true
Run this playbook:

bash
Kopiera
Redigera
ansible-playbook -i hosts.ini docker_install.yml
✅ 4. Install Kubernetes components (kubeadm, kubelet, kubectl) with Ansible
Create k8s_install.yml:

yaml
Kopiera
Redigera
---
- hosts: k8s-cluster
  become: true
  tasks:
    - name: Disable Swap
      shell: |
        swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab

    - name: Install Kubernetes prerequisites
      apt:
        name:
          - apt-transport-https
          - curl
        state: present

    - name: Add Kubernetes GPG key
      shell: |
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /usr/share/keyrings/kubernetes-apt-keyring.gpg

    - name: Add Kubernetes repo
      shell: |
        echo "deb [signed-by=/usr/share/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    - name: Install kubeadm, kubelet, kubectl
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
        update_cache: yes

    - name: Hold Kubernetes packages
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
        mark_hold: yes

    - name: Enable kubelet
      systemd:
        name: kubelet
        enabled: yes
Run this playbook:

bash
Kopiera
Redigera
ansible-playbook -i hosts.ini k8s_install.yml
✅ 5. Initialize Kubernetes Cluster (Using kubeadm)
On Master node (k8s-master-01) run:

bash
Kopiera
Redigera
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
Configure kubectl (on master):

bash
Kopiera
Redigera
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
✅ 6. Setup Networking (e.g., Flannel)
On master node, run:

bash
Kopiera
Redigera
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
✅ 7. Join Worker Nodes
Run the join command (obtained from master node output of kubeadm init) on each worker:

bash
Kopiera
Redigera
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
✅ 8. Validate Your Kubernetes Setup
Check nodes status:

bash
Kopiera
Redigera
kubectl get nodes
Check cluster components:

bash
Kopiera
Redigera
kubectl get pods -n kube-system
🎯 (Optional) Automate the entire Kubernetes cluster setup in Ansible
You can also automate the kubeadm init and join with Ansible by capturing token output and distributing it securely using Ansible facts and templates. This is more advanced but very achievable.

🚩 Recommendations:
GitOps Workflow: Use GitOps tools (ArgoCD, FluxCD) later for deployments.

Infrastructure-as-Code: Store Ansible playbooks in Git.

🎖️ You now have a robust Kubernetes cluster built and managed with Docker and Ansible!

Let me know if you need advanced configurations or have specific customizations you'd like to implement!
