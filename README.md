# DSN580S T2: Single-Node Kubernetes with Minikube on AWS

**Course:** DSN580S Distributed Systems & Software Defined Networks, CPUT
**Student number:** 213133369
**Lecturer:** Dr W Hendricks

This repository documents a single-node Kubernetes cluster built with Minikube on one Ubuntu EC2 instance in an AWS Academy Learner Lab, a three-replica nginx workload, and a Portainer dashboard reached through an SSH tunnel.

## Environment

| Item | Value |
|---|---|
| Platform | AWS Academy Learner Lab, region us-east-1 |
| Instance name | `DSN580S-213133369-minikube` |
| Instance ID | `i-0bcf8398b949502c8` |
| Instance type | t3.medium (2 vCPU, 4 GB RAM) |
| OS | Ubuntu Server 24.04.4 LTS |
| Disk | 30 GiB gp3 |
| Docker | 29.9.1 |
| kubectl | v1.37.1 |
| Minikube | v1.39.0 (Kubernetes v1.37.0, Docker driver) |
| Portainer | Community Edition 2.45.2 LTS (Helm chart `portainer-245.2.0`) |

The public IP changes every time the instance stops and starts. The commands below use `<public-ip>` as a placeholder.

## Repository contents

| Path | Purpose |
|---|---|
| `README.md` | Step-by-step commands and notes (this file) |
| `nginx-deployment.yaml` | Namespace, Deployment (3 replicas) and NodePort Service |
| `commands-history.txt` | Raw shell history from the server, unedited |
| `screenshots/` | Evidence, named by task |

## Evidence conventions

Every terminal screenshot starts with:

```bash
echo "213133369"; date; hostname
```

The namespace (`s213133369`), the deployment (`nginx-213133369`) and the service (`nginx-213133369-svc`) all carry the student number. No keys, tokens or passwords are stored in this repository.

---

## Task 1: Provision and secure the instance

1. In the Learner Lab, started the lab and opened the AWS console.
2. Launched an EC2 instance named `DSN580S-213133369-minikube` with Ubuntu Server 24.04 LTS, t3.medium, a new RSA `.pem` key pair and a 30 GiB gp3 volume.
3. Created a new security group with a single inbound rule: **SSH (TCP 22) from my own IP address only (/32)**. No Kubernetes API or dashboard port is open to `0.0.0.0/0`.
4. Connected from PowerShell on my own computer:

```bash
ssh -i dsn580s-key.pem ubuntu@<public-ip>
echo "213133369"; date; hostname; lsb_release -d
```

Screenshots: `screenshots/task1-ec2-instance.png`, `screenshots/task1-security-group.png`, `screenshots/task1-ssh.png`

## Task 2: Install Docker, kubectl and Minikube

All commands run on the instance as the normal `ubuntu` user (never as root for Minikube).

**Docker** (official apt repository method):

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker run hello-world
groups
```

**kubectl:**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Minikube:**

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
minikube version
```

**Start the cluster with the Docker driver and show the node is Ready:**

```bash
minikube start --driver=docker
echo "213133369"; date; hostname; whoami
minikube status
kubectl get nodes -o wide
```

Note: Minikube warned that its default 3072 MiB memory request leaves little headroom on a 3.8 GiB instance. The cluster started and ran normally.

Screenshots: `screenshots/task2-tools.png`, `screenshots/task2-node-ready.png`

## Task 3: Deploy and verify a workload

```bash
mkdir -p ~/dsn580s-minikube && cd ~/dsn580s-minikube
# nginx-deployment.yaml created here (contents are in this repository)
kubectl apply -f nginx-deployment.yaml
kubectl get all -n s213133369 -o wide
curl -s http://$(minikube ip):30080 | head -n 5
```

The manifest defines the namespace `s213133369`, the Deployment `nginx-213133369` (3 replicas of `nginx:stable`) and the NodePort Service `nginx-213133369-svc` (80:30080).

**Scale up and down:**

```bash
kubectl scale deployment nginx-213133369 -n s213133369 --replicas=5
kubectl get pods -n s213133369
kubectl scale deployment nginx-213133369 -n s213133369 --replicas=2
kubectl get pods -n s213133369
```

**Delete one pod and watch Kubernetes replace it:**

```bash
kubectl scale deployment nginx-213133369 -n s213133369 --replicas=3
POD=$(kubectl get pods -n s213133369 -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n s213133369
kubectl get pods -n s213133369
```

The replacement pod appeared within seconds with a new name and a new pod IP, while the Service address stayed the same.

Screenshots: `screenshots/task3-deploy-expose.png`, `screenshots/task3-scale.png`, `screenshots/task3-self-heal.png`

## Task 4: Dashboard (Portainer)

I chose **Portainer** and installed it with the official Helm chart, following the Portainer documentation (docs.portainer.io) and the `portainer/k8s` chart repository.

**Install Helm and Portainer on the instance:**

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm repo add portainer https://portainer.github.io/k8s/
helm repo update
helm install --create-namespace -n portainer portainer portainer/portainer --set service.type=NodePort
kubectl get pods,svc -n portainer
```

**Access through an SSH tunnel from my own computer** (the Portainer port is never exposed to the internet):

```powershell
ssh -i dsn580s-key.pem -L 9443:192.168.49.2:30779 ubuntu@<public-ip>
```

Then opened `https://localhost:9443` in the browser. Portainer's first-run setup needs a one-time token from its logs and must be completed within a few minutes of the pod starting:

```bash
kubectl logs -n portainer deployment/portainer | grep -i token
kubectl rollout restart deployment portainer -n portainer   # only if the setup page timed out
```

The admin password and setup token are not stored in this repository.

Screenshots: `screenshots/task4-portainer-install.png`, `screenshots/task4-tunnel.png`, `screenshots/task4-nodes.png`, `screenshots/task4-namespaces.png`, `screenshots/task4-deployments.png`, `screenshots/task4-pods.png`, `screenshots/task4-services.png`

## Task 5: Reflection

The reflection is in the PDF report submitted on Blackboard.

## Notes and problems encountered

- **Ubuntu version:** my first instance launched on Ubuntu 26.04. I relaunched on 24.04 LTS, the version most tools document, and terminated the first instance.
- **Two shells:** the Linux commands in this README run on the server (`ubuntu@...` prompt). Running them in my local PowerShell failed, as expected, with no effect on the cluster.
- **Portainer timeout:** the first-run setup page timed out once. Restarting the Portainer deployment reset it and issued a new setup token.
- **Lab sessions:** instances are shut down when a Learner Lab session ends, so the manifest and notes were copied to my computer with `scp` and committed here.

## References

- Kubernetes documentation: https://kubernetes.io/docs/
- Minikube documentation: https://minikube.sigs.k8s.io/docs/start/
- Docker Engine installation on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Portainer documentation: https://docs.portainer.io/
- Portainer Kubernetes Helm chart: https://github.com/portainer/k8s
