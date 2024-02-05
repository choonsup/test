# MLOps Bundle Package Implementation (Air-Gap)

## Contents

- [MLOps Bundle Package Implementation (Air-Gap)](#mlops-bundle-package-implementation-air-gap)
  - [Contents](#contents)
  - [Environment](#environment)
  - [1 Prerequisites](#1-prerequisites)
    - [1-1 Local RPM Repository](#1-1-local-rpm-repository)
    - [1-2 Remote RPM Repository](#1-2-remote-rpm-repository)
    - [1-3 Helm Chart](#1-3-helm-chart)
    - [1-4 Git Repo](#1-4-git-repo)
    - [1-5 Docker Images](#1-5-docker-images)
    - [1-6 RKE2 and Rancher](#1-6-rke2-and-rancher)
  - [2 Docker](#2-docker)
  - [3 Repository Server](#3-repository-server)
    - [3-1 Nexus](#3-1-nexus)
    - [3-2 Create Repository](#3-2-create-repository)
    - [3-3 Upload Images](#3-3-upload-images)
    - [3-4 Upload Helm Charts](#3-4-upload-helm-charts)
    - [3-5 Upload Raw Files](#3-5-upload-raw-files)
  - [4 Rancher Kubernetes](#4-rancher-kubernetes)
    - [4-1 Master Node](#4-1-master-node)
    - [4-2 Worker Node](#4-2-worker-node)
    - [4-3 Helm](#4-3-helm)
    - [4-4 Cert-Manager](#4-4-cert-manager)
    - [4-5 Rancher](#4-5-rancher)
  - [5 Gitlab](#5-gitlab)
  - [6 MLflow](#6-mlflow)
  - [7 Kubeflow](#7-kubeflow)
  - [8 Grafana and Prometheus](#8-grafana-and-prometheus)
  - [9 Images](#9-images)
  - [10 Sample Pipeline](#10-sample-pipeline)
  - [11 Monitoring Dashboard](#11-monitoring-dashboard)
  
  
## Environment

- bootstrap (internet accessable PC)
- Repository Server
- Gitlab Server
- RKE2 Master
- RKE2 Worker

## 1 Prerequisites

### 1-1 Local RPM Repository

> **Environment:** bootstrap

(1) Download RHEL iso  
(2) Copy iso file into Repository server  
(3) Mount iso file

```bash
mount -t udf -o loop rhel8.iso /mnt
```

(4) Copy all contents in iso file to Repository server

```bash
sudo mkdir /iso
sudo cp /mnt/* /iso/
```

(5) Configure local repo

```bash
sudo vi /etc/yum.repo.d/local.repo
```

```
[BaseOS]
name=RHEL 8 - BaseOS
gpgcheck=0
enabled=1
baseurl=file:///iso/BaseOS

[InstallMedia-AppStream]
name=RHEL 8 - AppStream
gpgcheck=0
enabled=1
baseurl=file:///iso/AppStream/
```

### 1-2 Remote RPM Repository

> **Environment:** bootstrap

Install httpd

```bash
sudo yum install httpd
sudo systemctl enable httpd
sudo systemctl start httpd
sudo echo Apache on RHEL {ProductNumber} > /var/www/html/index.html
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=http
```

Copy ISO Files

```bash
sudo mkdir /mnt/rhel{ProductNumber}
sudo mount -o loop,ro rhel-{ProductNumber}-x86_64-dvd.iso /mnt/rhel{ProductNumber}/
sudo cp -r /mnt/rhel{ProductNumber}/ /var/www/html/
sudo umount  /mnt/rhel{ProductNumber}
```

Create Repo File

```bash
vi /etc/yum.repos.d/rhel_http_repo.repo
```

```bash
[BaseOS_repo_http]
name=RHEL_8_x86_64_HTTP BaseOS
baseurl="http://myhost/rhel8/BaseOS"
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[AppStream_repo_http]
name=RHEL_8_x86_64_HTTP AppStream
baseurl="http://myhost/rhel8/AppStream"
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

### 1-3 Helm Chart

> **Environment:** bootstrap

Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh
```

Helm Binary

```bash
mkdir -p ~/bundle/helm
curl -OLs https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
mv helm-v3.14.0-linux-amd64.tar.gz ~/bundle/helm/
```

Cert Manager

```bash
mkdir -p ~/bundle/raw
cd ~/bundle/raw
curl -OLs https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml
cd ~/bundle/helm/
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm pull jetstack/cert-manager --version v1.13.3
```

Rancher

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
helm pull rancher-latest/rancher --version 2.8.0
```

NVIDIA Operator

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm pull nvidia/gpu-operator --version v23.9.1
```

NFS Server Provisioner

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm pull stable/nfs-server-provisioner --version 1.1.3
```

Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/kube-prometheus-stack --version 56.2.1
```

### 1-4 Git Repo

> **Environment:** bootstrap

Kubeflow

```bash
cd ~/bundle/raw/
curl -L https://api.github.com/repos/kubeflow/manifests/tarball/master -o kubeflow.tgz
```

Kustomize

```bash
cd ~/bundle/raw/
curl -OLs https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.3.0/kustomize_v5.3.0_linux_amd64.tar.gz
```

### 1-5 Docker Images

> **Environment:** bootstrap

Install Docker

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker <userid>
```

Configure Insecure Registry

```bash
sudo vi /etc/docker/daemon.json
```

```bash
{
  "insecure-registries" : ["nexus url"]
}
```

```bash
sudo systemctl restart docker
```

Create archive directory

```bash
mkdir -p ~/bundle/images
cd ~/bundle/images
```

Get docker images

```bash
# nexus
docker save sonatype/nexus3:3.64.0 -o nexus3-3.64.0.tar
# gitlab
docker save gitlab/gitlab-ee:16.7.4-ee.0 -o gitlab-ee-16.7.4-ee.0.tar
# mlflow
docker save amd64/python:3.9-slim -o python-3.9-slim.tar
docker save postgres:14.0 -o postgres-14.0.tar
docker save minio/minio -o minio.tar
# cert manager
docker save quay.io/jetstack/cert-manager-controller -o cert-manager-controller.tar
docker save quay.io/jetstack/cert-manager-webhook -o cert-manager-webhook.tar
docker save quay.io/jetstack/cert-manager-cainjector -o cert-manager-cainjector.tar
docker save quay.io/jetstack/cert-manager-acmesolver -o cert-manager-acmesolver.tar
docker save quay.io/jetstack/cert-manager-ctl -o cert-manager-ctl.tar
# nvidia operator
docker save nvcr.io/nvidia/cuda:12.3.1-base-ubi8 -o cuda-12.3.1.tar
docker save nvcr.io/nvidia/driver:535.129.03 -o driver-535.129.03.tar
docker save nvcr.io/nvidia/gpu-operator:v23.9.1 -o gpu-operator-v23.9.1.tar
docker save nvcr.io/nvidia/gpu-feature-discovery:v0.8.2-ubi8 -o gpu-feature-discovery-v0.8.2-ubi8.tar
docker save nvcr.io/nvidia/k8s/container-toolkit:v1.14.3-ubuntu20.04 -o container-toolkit-v1.14.3.tar
docker save nvcr.io/nvidia/k8s/dcgm-exporter:3.3.0-3.2.0-ubuntu22.04 -o dcgm-exporter-3.3.0-3.2.0.tar
docker save nvcr.io/nvidia/k8s-device-plugin:v0.14.3-ubi8 -o k8s-device-plugin-v0.14.3.tar
docker save nvcr.io/nvidia/kubevirt-gpu-device-plugin:v1.2.4 -o kubevirt-gpu-device-plugin-v1.2.4.tar
docker save nvcr.io/nvidia/cloud-native/dcgm:3.3.0-1-ubuntu22.04 -o dcgm-3.3.0-1.tar
docker save nvcr.io/nvidia/cloud-native/gpu-operator-validator:v23.9.1 -o gpu-operator-validator-v23.9.1.tar
docker save nvcr.io/nvidia/cloud-native/k8s-mig-manager:v0.5.5-ubuntu20.04 -o k8s-mig-manager-v0.5.5.tar
docker save nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.5 -o k8s-driver-manager-v0.6.5.tar
docker save nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.4 -o k8s-driver-manager-v0.6.4.tar
docker save nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.2 -o k8s-driver-manager-v0.6.2.tar
docker save nvcr.io/nvidia/cloud-native/k8s-kata-manager:v0.1.2 -o k8s-kata-manager-v0.1.2.tar
docker save nvcr.io/nvidia/cloud-native/k8s-cc-manager:v0.1.1 -o k8s-cc-manager-v0.1.1.tar
docker save nvcr.io/nvidia/cloud-native/nvidia-fs:2.17.5 -o nvidia-fs-2.17.5.tar
docker save nvcr.io/nvidia/cloud-native/vgpu-device-manager:v0.2.4 -o vgpu-device-manager-v0.2.4.tar
# nfs server provisioner
docker save quay.io/kubernetes_incubator/nfs-provisioner:v2.3.0 -o nfs-provisioner-v2.3.0.tar
# prometheus stack
docker save quay.io/prometheus/alertmanager:v0.26.0 -o alertmanager-v0.26.0.tar
docker save quay.io/prometheus-operator/admission-webhook:v0.71.2 -o admission-webhook-v0.71.2.tar
docker save quay.io/prometheus-operator/prometheus-operator:v0.71.2 -o prometheus-operator-v0.71.2.tar
docker save quay.io/prometheus-operator/prometheus-config-reloader:v0.71.2 -o prometheus-config-reloader-v0.71.2.tar
docker save quay.io/prometheus/prometheus:v2.49.1 -o prometheus-v2.49.1.tar
docker save quay.io/thanos/thanos:v0.33.0 -o thanos-v0.33.0.tar
docker save registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20221220-controller-v1.5.1-58-g787ea74b6 -o kube-webhook-certgen-v20221220-controller-v1.5.1-58.tar
```

Build MLflow Image

```bash
mkdir -p ~/bundle/mlflow
cd ~/bundle/mlflow
vi dockerfile
```

```shell
FROM amd64/python:3.9-slim

RUN apt-get update && apt-get install -y \
    git \
    wget \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -U pip &&\
    pip install boto3 mlflow psycopg2-binary

RUN cd /tmp && \
    wget https://dl.min.io/client/mc/release/linux-amd64/mc && \
    chmod +x mc && \
    mv mc /usr/bin/mc
```

```bash
docker build -t <nexus url>/mlflow:latest .
```

### 1-6 RKE2 and Rancher

> **Environment:** bootstrap

Get images in Racher category

```bash
mkdir -p ~/bundle/rancher
cd ~/bundle/rancher
curl -OLs https://github.com/rancher/rancher/releases/download/v2.8.0/rancher-images.txt
curl -OLs https://github.com/rancher/rancher/releases/download/v2.8.0/rancher-load-images.sh
curl -OLs https://github.com/rancher/rancher/releases/download/v2.8.0/rancher-save-images.sh
sort -u rancher-images.txt -o rancher-images.txt
chmod +x rancher-save-images.sh
./rancher-save-images.sh --image-list ./rancher-images.txt
# rancher-images.tar.gz
```

Get RKE2 Install Script

```bash
curl -sfL https://get.rke2.io --output install.sh
```

## 2 Docker

> **Environment:** Reposiroty Server, Gitlab Server

Install Docker

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker <userid>
```

Configure Insecure Registry

```bash
sudo vi /etc/docker/daemon.json
```

```bash
{
  "insecure-registries" : ["nexus url"]
}
```

```bash
sudo systemctl restart docker
```

## 3 Repository Server

### 3-1 Nexus

> **Environment:** Reposiroty Server

Install Nexus

```bash
mkdir ~/nexus-data
sudo chown -R 200 ~/nexus-data
docker load -i ~/bundle/images/nexus3-3.64.0.tar
docker run -d -p 8081:8081 --name nexus -v ~/nexus-data:/nexus-data sonatype/nexus3:3.64.0
```

Check Password

```bash
docker exec nexus cat /nexus-data/admin.password && echo
```

Init Nexus

```
Login Nexus3 WEB and change admin password(hpinvent)
```

Enable Scripting in Nexus

```bash
docker exec nexus bash -c 'echo nexus.scripts.allowCreation=true >> /nexus-data/etc/nexus.properties'
docker restart nexus
```

Install nexus3-cli

```bash
sudo pip install nexus3-cli
nexus3 login -U http://192.168.0.81:8081 -u admin -p hpinvent --no-x509_verify
```

### 3-2 Create Repository

> **Environment:** Reposiroty Server

Create Docker Registry(Hosted)

```bash
nexus3 repository create hosted docker \
--blob-store-name default \
--strict-content \
--write-policy allow \
--no-v1-enabled \
--http-port 8082 \
docker-airgap
```

Enable Docker Bearer Token Realm

```bash
nexus3 security realm activate DockerToken
```

Create Raw Repository(Hosted)

```bash
nexus3 repository create hosted raw \
--blob-store-name default \
--strict-content \
--write-policy allow \
artifacts
```

Create Helm Repository(Hosted)

```
Menu: Repository - Repositories - Create repository - helm (hosted)
Name: nexus-helm
```

### 3-3 Upload Images

> **Environment:** bootstrap

Rancher Images

```bash
cd ~/bundle/rancher
chmod +x rancher-load-images.sh
./rancher-load-images.sh --image-list ./rancher-images.txt --registry <nexus dokcer repo>
```

Helm Chart Images

```bash
cd ~/bundle/images
docker login <nexus url>
# mlflow
docker load python-3.9-slim.tar
docker tag  amd64/python:3.9-slim <nexus url>/amd64/python:3.9-slim
docker push <nexus url>/amd64/python:3.9-slim
docker load postgres-14.0.tar
docker tag  postgres:14.0 <nexus url>/postgres:14.0
docker push <nexus url>/postgres:14.0
docker load minio.tar
docker tag  minio/minio:latest <nexus url>/minio/minio:latest
docker push <nexus url>/minio/minio:latest
docker push <nexus url>/mlflow:latest
# gitlab
docker load gitlab-ee-16.7.4-ee.0.tar
docker tag  gitlab/gitlab-ee:16.7.4-ee.0 <nexus url>/gitlab/gitlab-ee:16.7.4-ee.0
docker push <nexus url>/gitlab/gitlab-ee:16.7.4-ee.0
# cert manager
docker load cert-manager-controller.tar
docker tag  quay.io/jetstack/cert-manager-controller:latest <nexus url>/quay.io/jetstack/cert-manager-controller:latest
docker push <nexus url>/quay.io/jetstack/cert-manager-controller:latest
docker load cert-manager-webhook.tar
docker tag  quay.io/jetstack/cert-manager-webhook:latest <nexus url>/quay.io/jetstack/cert-manager-webhook:latest
docker push <nexus url>quay.io/jetstack/cert-manager-webhook:latest
docker load cert-manager-cainjector.tar
docker tag  quay.io/jetstack/cert-manager-cainjector:latest <nexus url>/quay.io/jetstack/cert-manager-cainjector:latest
docker push <nexus url>quay.io/jetstack/cert-manager-cainjector:latest
docker load cert-manager-acmesolver.tar
docker tag  quay.io/jetstack/cert-manager-acmesolver:latest <nexus url>/quay.io/jetstack/cert-manager-acmesolver:latest
docker push <nexus url>quay.io/jetstack/cert-manager-acmesolver:latest
docker load cert-manager-ctl.tar
docker tag  quay.io/jetstack/cert-manager-ctl:latest <nexus url>/quay.io/jetstack/cert-manager-ctl:latest
docker push <nexus url>quay.io/jetstack/cert-manager-ctl:latest
# nvidia operator
docker load cuda-12.3.1.tar
docker tag  nvcr.io/nvidia/cuda:12.3.1-base-ubi8 <nexus url>/nvcr.io/nvidia/cuda:12.3.1-base-ubi8
docker push <nexus url>nvcr.io/nvidia/cuda:12.3.1-base-ubi8
docker load driver-535.129.03.tar
docker tag  nvcr.io/nvidia/driver:535.129.03 <nexus url>/nvcr.io/nvidia/driver:535.129.03
docker push <nexus url>nvcr.io/nvidia/driver:535.129.03
docker load gpu-operator-v23.9.1.tar
docker tag  nvcr.io/nvidia/gpu-operator:v23.9.1 <nexus url>/nvcr.io/nvidia/gpu-operator:v23.9.1
docker push <nexus url>nvcr.io/nvidia/gpu-operator:v23.9.1
docker load gpu-feature-discovery-v0.8.2-ubi8.tar
docker tag  nvcr.io/nvidia/gpu-feature-discovery:v0.8.2-ubi8 <nexus url>/nvcr.io/nvidia/gpu-feature-discovery:v0.8.2-ubi8
docker push <nexus url>nvcr.io/nvidia/gpu-feature-discovery:v0.8.2-ubi8
docker load container-toolkit-v1.14.3.tar
docker tag  nvcr.io/nvidia/k8s/container-toolkit:v1.14.3-ubuntu20.04 <nexus url>/nvcr.io/nvidia/k8s/container-toolkit:v1.14.3-ubuntu20.04
docker push <nexus url>nvcr.io/nvidia/k8s/container-toolkit:v1.14.3-ubuntu20.04
docker load dcgm-exporter-3.3.0-3.2.0.tar
docker tag  nvcr.io/nvidia/k8s/dcgm-exporter:3.3.0-3.2.0-ubuntu22.04 <nexus url>/nvcr.io/nvidia/k8s/dcgm-exporter:3.3.0-3.2.0-ubuntu22.04
docker push <nexus url>nvcr.io/nvidia/k8s/dcgm-exporter:3.3.0-3.2.0-ubuntu22.04
docker load k8s-device-plugin-v0.14.3.tar
docker tag  nvcr.io/nvidia/k8s-device-plugin:v0.14.3-ubi8 <nexus url>/nvcr.io/nvidia/k8s-device-plugin:v0.14.3-ubi8
docker push <nexus url>/nvcr.io/nvidia/k8s-device-plugin:v0.14.3-ubi8
docker load kubevirt-gpu-device-plugin-v1.2.4.tar
docker tag  nvcr.io/nvidia/kubevirt-gpu-device-plugin:v1.2.4 <nexus url>/nvcr.io/nvidia/kubevirt-gpu-device-plugin:v1.2.4
docker push <nexus url>/nvcr.io/nvidia/kubevirt-gpu-device-plugin:v1.2.4
docker load dcgm-3.3.0-1.tar
docker tag  nvcr.io/nvidia/cloud-native/dcgm:3.3.0-1-ubuntu22.04 <nexus url>/nvcr.io/nvidia/cloud-native/dcgm:3.3.0-1-ubuntu22.04
docker push <nexus url>/nvcr.io/nvidia/cloud-native/dcgm:3.3.0-1-ubuntu22.04
docker load gpu-operator-validator-v23.9.1.tar
docker tag  nvcr.io/nvidia/cloud-native/gpu-operator-validator:v23.9.1 <nexus url>/nvcr.io/nvidia/cloud-native/gpu-operator-validator:v23.9.1
docker push <nexus url>/nvcr.io/nvidia/cloud-native/gpu-operator-validator:v23.9.1
docker load k8s-mig-manager-v0.5.5.tar
docker tag  nvcr.io/nvidia/cloud-native/k8s-mig-manager:v0.5.5-ubuntu20.04 <nexus url>/nvcr.io/nvidia/cloud-native/k8s-mig-manager:v0.5.5-ubuntu20.04
docker push <nexus url>/nvcr.io/nvidia/cloud-native/k8s-mig-manager:v0.5.5-ubuntu20.04
docker load k8s-driver-manager-v0.6.5.tar
docker tag  nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.5 <nexus url>/nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.5
docker push <nexus url>/nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.5
docker load k8s-driver-manager-v0.6.4.tar
docker tag  nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.4 <nexus url>/nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.4
docker push <nexus url>/nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.4
docker load k8s-driver-manager-v0.6.2.tar
docker tag  nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.2 <nexus url>/nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.2
docker push <nexus url>/nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.2
docker load k8s-kata-manager-v0.1.2.tar
docker tag  nvcr.io/nvidia/cloud-native/k8s-kata-manager:v0.1.2 <nexus url>/nvcr.io/nvidia/cloud-native/k8s-kata-manager:v0.1.2
docker push <nexus url>/nvcr.io/nvidia/cloud-native/k8s-kata-manager:v0.1.2
docker load k8s-cc-manager-v0.1.1.tar
docker tag  nvcr.io/nvidia/cloud-native/k8s-cc-manager:v0.1.1 <nexus url>/nvcr.io/nvidia/cloud-native/k8s-cc-manager:v0.1.1
docker push <nexus url>/nvcr.io/nvidia/cloud-native/k8s-cc-manager:v0.1.1
docker load nvidia-fs-2.17.5.tar
docker tag  nvcr.io/nvidia/cloud-native/nvidia-fs:2.17.5 <nexus url>/nvcr.io/nvidia/cloud-native/nvidia-fs:2.17.5
docker push <nexus url>/nvcr.io/nvidia/cloud-native/nvidia-fs:2.17.5
docker load vgpu-device-manager-v0.2.4.tar
docker tag  nvcr.io/nvidia/cloud-native/vgpu-device-manager:v0.2.4 <nexus url>/nvcr.io/nvidia/cloud-native/vgpu-device-manager:v0.2.4
docker push <nexus url>/nvcr.io/nvidia/cloud-native/vgpu-device-manager:v0.2.4
# nfs server provisioner
docker load nfs-provisioner-v2.3.0.tar
docker tag  quay.io/kubernetes_incubator/nfs-provisioner:v2.3.0 quay.io/kubernetes_incubator/nfs-provisioner:v2.3.0
docker push quay.io/kubernetes_incubator/nfs-provisioner:v2.3.0
# prometheus stack
docker load alertmanager-v0.26.0.tar
docker tag  quay.io/prometheus/alertmanager:v0.26.0 <nexus url>/quay.io/prometheus/alertmanager:v0.26.0
docker push <nexus url>/quay.io/prometheus/alertmanager:v0.26.0
docker load admission-webhook-v0.71.2.tar
docker tag  quay.io/prometheus-operator/admission-webhook:v0.71.2 <nexus url>/quay.io/prometheus-operator/admission-webhook:v0.71.2
docker push <nexus url>/quay.io/prometheus-operator/admission-webhook:v0.71.2
docker load prometheus-operator-v0.71.2.tar
docker tag  quay.io/prometheus-operator/prometheus-operator:v0.71.2 <nexus url>/quay.io/prometheus-operator/prometheus-operator:v0.71.2
docker push <nexus url>/quay.io/prometheus-operator/prometheus-operator:v0.71.2
docker load prometheus-config-reloader-v0.71.2.tar
docker tag  quay.io/prometheus-operator/prometheus-config-reloader:v0.71.2 <nexus url>/quay.io/prometheus-operator/prometheus-config-reloader:v0.71.2
docker push <nexus url>/quay.io/prometheus-operator/prometheus-config-reloader:v0.71.2
docker load prometheus-v2.49.1.tar
docker tag  quay.io/prometheus/prometheus:v2.49.1 <nexus url>/quay.io/prometheus/prometheus:v2.49.1
docker push <nexus url>/quay.io/prometheus/prometheus:v2.49.1
docker load thanos-v0.33.0.tar
docker tag  quay.io/thanos/thanos:v0.33.0 <nexus url>/quay.io/thanos/thanos:v0.33.0
docker push <nexus url>/quay.io/thanos/thanos:v0.33.0
docker load kube-webhook-certgen-v20221220-controller-v1.5.1-58.tar
docker tag  registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20221220-controller-v1.5.1-58-g787ea74b6 <nexus url>/registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20221220-controller-v1.5.1-58-g787ea74b6
docker push <nexus url>/registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20221220-controller-v1.5.1-58-g787ea74b6
```

### 3-4 Upload Helm Charts

> **Environment:** bootstrap

Upload Helm Chart to Repository

```bash
cd ~/bundle/helm
curl -u admin:hpinvent http://<nexus url>/repository/nexus-helm/ --upload-file cert-manager-v1.13.3.tgz
curl -u admin:hpinvent http://<nexus url>/repository/nexus-helm/ --upload-file rancher-2.8.0.tgz
curl -u admin:hpinvent http://<nexus url>/repository/nexus-helm/ --upload-file gpu-operator-v23.9.1.tgz
curl -u admin:hpinvent http://<nexus url>/repository/nexus-helm/ --upload-file nfs-server-provisioner-1.1.3.tgz
curl -u admin:hpinvent http://<nexus url>/repository/nexus-helm/ --upload-file kube-prometheus-stack-56.2.1.tgz
```

### 3-5 Upload Raw Files

> **Environment:** bootstrap

Upload Raw Files to Repository

```bash
cd ~/bundle/helm/
curl -u admin:hpinvent --upload-file helm-v3.14.0-linux-amd64.tar.gz http://<nexus url>/repository/artifacts/helm-v3.14.0-linux-amd64.tar.gz
cd ~/bundle/raw
curl -u admin:hpinvent --upload-file cert-manager.crds.yaml http://<nexus url>/repository/artifacts/cert-manager.crds.yaml
curl -u admin:hpinvent --upload-file kubeflow.tgz http://<nexus url>/repository/artifacts/kubeflow.tgz
curl -u admin:hpinvent --upload-file kustomize_v5.3.0_linux_amd64.tar.gz http://<nexus url>/repository/artifacts/kustomize_v5.3.0_linux_amd64.tar.gz
curl -u admin:hpinvent --upload-file mc http://<nexus url>/repository/artifacts/mc
cd ~/bundle/rancher
curl -u admin:hpinvent --upload-file rancher-images.txt http://<nexus url>/repository/artifacts/rancher/rancher-images.txt
curl -u admin:hpinvent --upload-file rancher-load-images.sh http://<nexus url>/repository/artifacts/rancher/rancher-load-images.sh
curl -u admin:hpinvent --upload-file rancher-save-images.sh http://<nexus url>/repository/artifacts/rancher/rancher-save-images.sh
curl -u admin:hpinvent --upload-file install.sh http://<nexus url>/repository/artifacts/rancher/install.sh
```

## 4 Rancher Kubernetes

### 4-1 Master Node

> **Environment:** RKE2 Master

Configure Private Registry

```bash
sudo vi /etc/rancher/rke2/registries.yaml
```

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://<nexus url>"
configs:
  "<nexus url>":
    auth:
      username: admin
      password: hpinvent
```

Configure Default Registry

```bash
sudo vi /etc/rancher/rke2/config.yaml
```

```yaml
system-default-registry: "<nexus url>"
```

Install RKE2

```bash
curl -OLs http://<nexus url>/repository/artifacts/rancher/install.sh
./install.sh
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

Check Logs

```bash
sudo journalctl -u rke2-server -f
# Wait for message below
"Tunnel authorizer set Kubelet Port 10250"
```

Configure kubectl

```bash
mkdir ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown stack:stack ~/.kube/config
sudo cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
sudo cp /var/lib/rancher/rke2/bin/kubectl ~/.kube/
sudo chown stack:stack ~/.kube/kubectl
```

Check RKE2 Server Token

```bash
sudo cat /var/lib/rancher/rke2/server/node-token
```

### 4-2 Worker Node

> **Environment:** RKE2 Worker

Configure Private Registry

```bash
sudo vi /etc/rancher/rke2/registries.yaml
```

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://<nexus url>"
configs:
  "<nexus url>":
    auth:
      username: admin
      password: hpinvent
```

Configure Agent

```bash
sudo vi /etc/rancher/rke2/config.yaml
```

```yaml
system-default-registry: "<nexus url>"
server: https://<server-ip>:9345
token: <RKE2 Server Token>
```

Install RKE2

```bash
INSTALL_RKE2_TYPE="agent" sh install.sh
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

Check Logs

```bash
sudo journalctl -u rke2-agent -f
# Wait for message below
"Tunnel authorizer set Kubelet Port 10250"
```

### 4-3 Helm

> **Environment:** Repository Server

Configure kubectl

```bash
mkdir ~/.kube
scp <master>:~/.kube/config ~/.kube/config
vi ~/.kube/config
# Change IP 127.0.0.1 -> master node ip
scp <master>:~/.kube/kubectl ~/.kube/kubectl
sudo cp ~/.kube/kubectl /usr/local/bin/
```

Install Helm

```bash
curl -OLs http://<nexus url>/repository/artifacts/helm-v3.14.0-linux-amd64.tar.gz
tar zxvf helm-v3.14.0-linux-amd64.tar.gz
sudo mv ./linux-amd64/helm /usr/local/bin/
sudo chmod +x /usr/local/bin/helm
sudo rm -rf linux-amd64
sudo rm helm-v3.14.0-linux-amd64.tar.gz
```

Add nexus-helm repo

```bash
helm repo add nexus-helm http://<nexus url>/repository/nexus-helm --username admin --password hpinvent
helm repo update
```

### 4-4 Cert-Manager

> **Environment:** Repository Server

Create Chart Directory

```bash
mkdir -p ~/charts
cd ~/charts
```

Download Installation Files

```bash
helm fetch nexus-helm/cert-manager
tar zxvf cert-manager-v1.13.3.tgz
cd ~/charts/cert-manager
curl -OLs http://<nexus url>/repository/artifacts/cert-manager.crds.yaml
```

Change Repository URL

```bash
vi values.yaml
```

Before change

```yaml
  repository: quay.io/jetstack/cert-manager-controller
    repository: quay.io/jetstack/cert-manager-webhook
    repository: quay.io/jetstack/cert-manager-cainjector
    repository: quay.io/jetstack/cert-manager-acmesolver
    repository: quay.io/jetstack/cert-manager-ctl
```

After change

```yaml
  repository: <nexus url>/quay.io/jetstack/cert-manager-controller
    repository: <nexus url>/quay.io/jetstack/cert-manager-webhook
    repository: <nexus url>/quay.io/jetstack/cert-manager-cainjector
    repository: <nexus url>/quay.io/jetstack/cert-manager-acmesolver
    repository: <nexus url>/quay.io/jetstack/cert-manager-ctl
```

Install Cert-Manager

```bash
cd ~/charts/cert-manager
kubectl create namespace cert-manager
kubectl apply -f cert-manager-crd.yaml
cd ~/charts
helm install -n cert-manager cert-manager ./cert-manager
```

### 4-5 Rancher

> **Environment:** Repository Server

Install Rancher(Air-Gap)

```bash
helm install rancher nexus-helm/rancher \
--namespace cattle-system \
--set hostname=rancher.stack.com \
--set certmanager.version=v1.13.3 \
--set rancherImage=<nexus url>/rancher/rancher \
--set systemDefaultRegistry=192.168.0.81:8082 \
--set useBundledSystemChart=true \
--createnamespace
```

## 5 Gitlab

> **Environment:** Gitlab Server

Create Docker Compose Directory

```bash
mkdir -p ~/docker-compose
```

Create Gitlab Directory

```bash
mkdir -p ~/gitlab/data
mkdir -p ~/gitlab/log
mkdir -p ~/gitlab/config
```

Create docker-compose.yml

```bash
cd ~/docker-compose
vi docker-compose.yml
```

```yaml
version: "3.6"
services:
  gitlab:
    image: <nexus url>gitlab/gitlab-ee:16.7.4-ee.0
    container_name: gitlab
    restart: always
    hostname: "gitlab.example.com"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        # Add any other gitlab.rb configuration here, each on its own line
        external_url 'https://gitlab.example.com'
        gitlab_rails['gitlab_shell_ssh_port'] = 2424
    ports:
      - "80:80"
      - "443:443"
      - "2424:2424"
    volumes:
      - "/home/stack/gitlab/config:/etc/gitlab"
      - "/home/stack/gitlab/logs:/var/log/gitlab"
      - "/home/stack/gitlab/data:/var/opt/gitlab"
    shm_size: "256m"
```

Run Gitlab Server

```bash
docker compose up -d
```

Get Initial Admin Password

```bash
docker exec gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

## 6 MLflow

> **Environment:** Gitlab Server

Create Gitlab Directory

```bash
mkdir -p ~/mlflow
cd ~/mlflow
```

Create docker-compose.yaml

```bash
vi docker-compose.yaml
```

```yaml
version: "3"

services:
  mlflow-backend-store:
    image: <nexus url>/postgres:14.0
    container_name: mlflow-backend-store
    environment:
      POSTGRES_USER: mlflowuser
      POSTGRES_PASSWORD: mlflowpassword
      POSTGRES_DB: mlflowdatabase
    healthcheck:
      test:
        ["CMD", "pg_isready", "-q", "-U", "mlflowuser", "-d", "mlflowdatabase"]
      interval: 10s
      timeout: 5s
      retries: 5

  mlflow-artifact-store:
    image: <nexus url>/minio/minio
    container_name: mlflow-artifact-store
    ports:
      - 9000:9000
      - 9001:9001
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: miniostorage
    command:
      server /data/minio --console-address :9001
      #healthcheck:
      #test: ["CMD", "curl", "-I", "http://localhost:9000/minio/health/live"]
      #interval: 30s
      #timeout: 20s
      #retries: 3
    healthcheck:
      test: timeout 5s bash -c ':> /dev/tcp/127.0.0.1/9000' || exit 1
      start_period: 5s
      interval: 10s
      timeout: 5s
      retries: 2

  mlflow-server:
    image: <nexus url>/mlflow
    container_name: mlflow-server
    depends_on:
      mlflow-backend-store:
        condition: service_healthy
      mlflow-artifact-store:
        condition: service_healthy
    ports:
      - 5001:5000
    environment:
      AWS_ACCESS_KEY_ID: minio
      AWS_SECRET_ACCESS_KEY: miniostorage
      MLFLOW_S3_ENDPOINT_URL: http://mlflow-artifact-store:9000
    command:
      - /bin/sh
      - -c
      - |
        mc config host add mlflowminio http://mlflow-artifact-store:9000 minio miniostorage &&
        mc mb --ignore-existing mlflowminio/mlflow
        mlflow server \
        --backend-store-uri postgresql://mlflowuser:mlflowpassword@mlflow-backend-store/mlflowdatabase \
        --default-artifact-root s3://mlflow/ \
        --host 0.0.0.0
```

Run MLflow

```bash
docker compose up -d
```

## 7 Kubeflow

> **Environment:** Repository Server

Install Kustomize

```bash
curl -Ls http://<nexus url>/repository/artifacts/kustomize_v5.3.0_linux_amd64.tar.gz -o kustomize
sudo mv kustomize /usr/local/bin/
```

Download NFS Server Provisioner Chart

```bash
helm fetch nexus-helm/nfs-server-provisioner
tar zxvf nfs-server-provisioner-1.1.3.tgz
cd ~/charts/nfs-server-provisioner
```

Change Repository URL

```bash
vi values.yaml
```

Before change

```yaml
repository: quay.io/kubernetes_incubator/nfs-provisioner
```

After change

```yaml
repository: <nexus url>quay.io/kubernetes_incubator/nfs-provisioner
```

Install NFS Server Provisioner

```bash
cd ~/charts
helm install nfs-server-provisioner ./nfs-server-provisioner
kubectl patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Download Installation Files

```bash
cd ~/charts
curl -OLs http://<nexus url>/repository/artifacts/kubeflow.tgz
tar zxvf kubeflow.tgz
```

Change Repository URL

```bash
cd ~/charts/kubeflow-*
vi <below files>
```

> **Action:** Input <nexus url> before image urls in below files

```yaml
common/istio-1-17/cluster-local-gateway/base/cluster-local-gateway.yaml
image: docker.io/istio/proxyv2:1.17.5

common/istio-1-17/profile.yaml
image: proxyv2
image: proxyv2
image: pilot

common/istio-1-17/istio-install/base/install.yaml
image: docker.io/istio/proxyv2:1.17.5
image: docker.io/istio/pilot:1.17.5

common/knative/knative-serving/base/upstream/net-istio.yaml
image: gcr.io/knative-releases/knative.dev/net-istio/cmd/controller@sha256:421aa67057240fa0c56ebf2c6e5b482a12842005805c46e067129402d1751220
image: gcr.io/knative-releases/knative.dev/net-istio/cmd/webhook@sha256:bfa1dfea77aff6dfa7959f4822d8e61c4f7933053874cd3f27352323e6ecd985

common/knative/knative-serving/base/upstream/serving-core.yaml
image: gcr.io/knative-releases/knative.dev/serving/cmd/queue@sha256:dabaecec38860ca4c972e6821d5dc825549faf50c6feb8feb4c04802f2338b8a
queue-sidecar-image: gcr.io/knative-releases/knative.dev/serving/cmd/queue@sha256:dabaecec38860ca4c972e6821d5dc825549faf50c6feb8feb4c04802f2338b8a
image: gcr.io/knative-releases/knative.dev/serving/cmd/activator@sha256:c2994c2b6c2c7f38ad1b85c71789bf1753cc8979926423c83231e62258837cb9
image: gcr.io/knative-releases/knative.dev/serving/cmd/autoscaler@sha256:8319aa662b4912e8175018bd7cc90c63838562a27515197b803bdcd5634c7007
image: gcr.io/knative-releases/knative.dev/serving/cmd/controller@sha256:98a2cc7fd62ee95e137116504e7166c32c65efef42c3d1454630780410abf943
image: gcr.io/knative-releases/knative.dev/serving/cmd/domain-mapping@sha256:f66c41ad7a73f5d4f4bdfec4294d5459c477f09f3ce52934d1a215e32316b59b
image: gcr.io/knative-releases/knative.dev/serving/cmd/domain-mapping-webhook@sha256:7368aaddf2be8d8784dc7195f5bc272ecfe49d429697f48de0ddc44f278167aa
image: gcr.io/knative-releases/knative.dev/serving/cmd/webhook@sha256:4305209ce498caf783f39c8f3e85dfa635ece6947033bf50b0b627983fd65953

common/knative/knative-serving-post-install-jobs/base/serving-post-install-jobs.yaml
image: gcr.io/knative-releases/knative.dev/pkg/apiextensions/storageversion/cmd/migrate@sha256:bc91e1fdaf3b67876ca33de1ce15b1268ed0ca8da203102b7699286fae97cf58

common/knative/knative-eventing-post-install-jobs/base/eventing-post-install.yaml
image: gcr.io/knative-releases/knative.dev/pkg/apiextensions/storageversion/cmd/migrate@sha256:56780f69e6496bb4790b0c147deb652a2b020ff81e08d58cc58a61cd649b1121

common/knative/knative-eventing/base/upstream/eventing-core.yaml
image: gcr.io/knative-releases/knative.dev/eventing/cmd/controller@sha256:92967bab4ad8f7d55ce3a77ba8868f3f2ce173c010958c28b9a690964ad6ee9b
image: gcr.io/knative-releases/knative.dev/eventing/cmd/mtping@sha256:6d35cc98baa098fc0c5b4290859e363a8350a9dadc31d1191b0b5c9796958223
image: gcr.io/knative-releases/knative.dev/eventing/cmd/webhook@sha256:ebf93652f0254ac56600bedf4a7d81611b3e1e7f6526c6998da5dd24cdc67ee1

common/knative/knative-eventing/base/upstream/in-memory-channel.yaml
image: gcr.io/knative-releases/knative.dev/eventing/cmd/in_memory/channel_controller@sha256:e004174a896811aec46520b1f2857f1973762389426bb0e0fc5d2332d5e36c7a
image: gcr.io/knative-releases/knative.dev/eventing/cmd/in_memory/channel_dispatcher@sha256:521234b4cff9d3cd32f8264cd7c830caa06f9982637b4866e983591fa1abc418

common/knative/knative-eventing/base/upstream/mt-channel-broker.yaml
image: gcr.io/knative-releases/knative.dev/eventing/cmd/broker/filter@sha256:29bd9f43359153c0ea39cf382d5f25ca43f55abbbce3d802ca37cc4d5c4a6942
image: gcr.io/knative-releases/knative.dev/eventing/cmd/broker/ingress@sha256:7f3b05f6e0abae19e9438fac44dd9938ddd2293014ef0fb8d388450c9ff63000
image: gcr.io/knative-releases/knative.dev/eventing/cmd/mtchannel_broker@sha256:4040ffc2d34e950b7969b4ba90cec29e65e506126ddb195faf3a56cb2fa653e8

common/cert-manager/cert-manager/base/cert-manager.yaml
image: "quay.io/jetstack/cert-manager-cainjector:v1.12.2"
image: "quay.io/jetstack/cert-manager-controller:v1.12.2"
image: "quay.io/jetstack/cert-manager-webhook:v1.12.2"

common/dex/base/deployment.yaml
image: ghcr.io/dexidp/dex:v2.36.0

common/oidc-client/oauth2-proxy/base/deployment.yaml
image: quay.io/oauth2-proxy/oauth2-proxy:v7.4.0

common/oidc-client/oidc-authservice/base/statefulset.yaml
image: gcr.io/arrikto/kubeflow/oidc-authservice:e236439

common/oidc-client/oidc-authservice/overlays/ibm-storage-config/statefulset.yaml
image: busybox

common/istio-cni-1-17/cluster-local-gateway/base/cluster-local-gateway.yaml
image: docker.io/istio/proxyv2:1.17.5

common/istio-cni-1-17/istio-install/base/install.yaml
image: busybox:1.28
image: docker.io/istio/install-cni:1.17.5
image: docker.io/istio/proxyv2:1.17.5
image: docker.io/istio/pilot:1.17.5

contrib/ray/kuberay-operator/base/resources.yaml
image: "kuberay/operator:v0.4.0"

contrib/ray/raycluster_example.yaml
image: rayproject/ray:2.2.0-py38-cpu
image: rayproject/ray:2.2.0-py38-cpu
image: busybox:1.28

contrib/metacontroller/base/stateful-set.yaml
image: 'docker.io/metacontrollerio/metacontroller:v2.0.4'

contrib/bentoml/bentoml-yatai-stack/bases/yatai-image-builder/resources.yaml
image: "quay.io/bentoml/yatai-image-builder:1.1.3"

contrib/bentoml/bentoml-yatai-stack/bases/yatai-deployment/resources.yaml
image: "quay.io/bentoml/yatai-deployment:1.1.4"

contrib/bentoml/deployment_from_bento.yaml
image: docker.io/bentoml/fraud_detection:o5smnagbncigycvj

contrib/kserve/kserve/kserve_kubeflow.yaml
image: kserve/kserve-controller:v0.11.0
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1

contrib/kserve/kserve/kserve.yaml
image: kserve/kserve-controller:v0.11.0
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1

contrib/kserve/kserve/kserve-runtimes.yaml
image: kserve/lgbserver:v0.11.0
image: docker.io/seldonio/mlserver:1.3.2
image: kserve/paddleserver:v0.11.0
image: kserve/pmmlserver:v0.11.0
image: kserve/sklearnserver:v0.11.0
image: tensorflow/serving:2.6.2
image: pytorch/torchserve-kfs:0.8.0
image: nvcr.io/nvidia/tritonserver:23.05-py3
image: kserve/xgbserver:v0.11.0

contrib/kserve/models-web-app/base/deployment.yaml
image: kserve/models-web-app:latest

contrib/seldon/seldon-core-operator/base/resources.yaml
image: 'docker.io/seldonio/seldon-core-operator:1.17.1'

contrib/seldon/example.yaml
image: seldonio/echo-model:1.17.1

apps/pipeline/upstream/third-party/metacontroller/base/stateful-set.yaml
image: 'docker.io/metacontrollerio/metacontroller:v2.0.4'

apps/pipeline/upstream/third-party/prometheus/prometheus-deployment.yaml
image: prom/prometheus

apps/pipeline/upstream/third-party/argo/upstream/manifests/quick-start/postgres/postgres-deployment.yaml
image: postgres:12-alpine

apps/pipeline/upstream/third-party/argo/upstream/manifests/quick-start/base/prometheus/prometheus-deployment.yaml
image: prom/prometheus

apps/pipeline/upstream/third-party/argo/upstream/manifests/quick-start/base/minio/minio-deploy.yaml
image: minio/minio

apps/pipeline/upstream/third-party/argo/upstream/manifests/quick-start/mysql/mysql-deployment.yaml
image: mysql:8

apps/pipeline/upstream/third-party/argo/upstream/manifests/quick-start/sso/dex/dex-deploy.yaml
image: quay.io/dexidp/dex:v2.23.0

apps/pipeline/upstream/third-party/argo/upstream/manifests/base/argo-server/argo-server-deployment.yaml
image: quay.io/argoproj/argocli:latest

apps/pipeline/upstream/third-party/argo/upstream/manifests/base/workflow-controller/workflow-controller-deployment.yaml
image: quay.io/argoproj/workflow-controller:latest

apps/pipeline/upstream/third-party/argo/base/workflow-controller-deployment-patch.yaml
image: gcr.io/ml-pipeline/workflow-controller:v3.3.10-license-compliance

apps/pipeline/upstream/third-party/grafana/grafana-deployment.yaml
image: grafana/grafana:5.3.4

apps/pipeline/upstream/third-party/minio/base/minio-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/pipeline/upstream/third-party/mysql/base/mysql-deployment.yaml
image: gcr.io/ml-pipeline/mysql:8.0.26

apps/pipeline/upstream/third-party/application/application-controller-deployment.yaml
image: gcr.io/ml-pipeline/application-crd-controller:1.0-beta-non-cluster-role

apps/pipeline/upstream/base/pipeline/ml-pipeline-persistenceagent-deployment.yaml
image: gcr.io/ml-pipeline/persistenceagent:dummy

apps/pipeline/upstream/base/pipeline/ml-pipeline-scheduledworkflow-deployment.yaml
image: gcr.io/ml-pipeline/scheduledworkflow:dummy

apps/pipeline/upstream/base/pipeline/metadata-writer/metadata-writer-deployment.yaml
image: gcr.io/ml-pipeline/metadata-writer:dummy

apps/pipeline/upstream/base/pipeline/ml-pipeline-ui-deployment.yaml
image: gcr.io/ml-pipeline/frontend:dummy

apps/pipeline/upstream/base/pipeline/ml-pipeline-viewer-crd-deployment.yaml
image: gcr.io/ml-pipeline/viewer-crd-controller:dummy

apps/pipeline/upstream/base/pipeline/ml-pipeline-apiserver-deployment.yaml
image: gcr.io/ml-pipeline/api-server:dummy

apps/pipeline/upstream/base/pipeline/ml-pipeline-visualization-deployment.yaml
image: gcr.io/ml-pipeline/visualization-server:dummy

apps/pipeline/upstream/base/cache-deployer/cache-deployer-deployment.yaml
gcr.io/ml-pipeline/cache-deployer:dummy

apps/pipeline/upstream/base/metadata/base/metadata-grpc-deployment.yaml
image: gcr.io/tfx-oss-public/ml_metadata_store_server:1.5.0

apps/pipeline/upstream/base/metadata/base/metadata-envoy-deployment.yaml
image: gcr.io/ml-pipeline/metadata-envoy:dummy

apps/pipeline/upstream/base/metadata/overlays/db/metadata-db-deployment.yaml
image: mysql:8.0.3

apps/pipeline/upstream/base/installs/multi-user/pipelines-profile-controller/deployment.yaml
image: python:3.7

apps/pipeline/upstream/base/installs/multi-user/pipelines-profile-controller/sync.py
visualization_server_image: gcr.io/ml-pipeline/visualization-server
frontend_image: gcr.io/ml-pipeline/frontend

apps/pipeline/upstream/base/cache/cache-deployment.yaml
image: gcr.io/ml-pipeline/cache-server:dummy

apps/pipeline/upstream/env/azure/minio-azure-gateway/minio-azure-gateway-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/pipeline/upstream/env/gcp/inverse-proxy/proxy-deployment.yaml
image: gcr.io/ml-pipeline/inverse-proxy-agent:dummy

apps/pipeline/upstream/env/gcp/minio-gcs-gateway/minio-gcs-gateway-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/pipeline/upstream/env/gcp/cloudsql-proxy/cloudsql-proxy-deployment.yaml
image: gcr.io/cloudsql-docker/gce-proxy:1.25.0

apps/profiles/upstream/manager/manager.yaml
image: docker.io/kubeflownotebookswg/profile-controller

apps/profiles/upstream/default/manager_auth_proxy_patch.yaml
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.11.0

apps/profiles/upstream/overlays/kubeflow/patches/kfam.yaml
image: docker.io/kubeflownotebookswg/kfam

apps/tensorboard/tensorboard-controller/upstream/manager/manager.yaml
image: docker.io/kubeflownotebookswg/tensorboard-controller

apps/tensorboard/tensorboard-controller/upstream/default/manager_auth_proxy_patch.yaml
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0

apps/tensorboard/tensorboards-web-app/upstream/base/deployment.yaml
image: docker.io/kubeflownotebookswg/tensorboards-web-app

apps/training-operator/upstream/base/deployment.yaml
image: kubeflow/training-operator

apps/jupyter/jupyter-web-app/upstream/base/deployment.yaml
image: docker.io/kubeflownotebookswg/jupyter-web-app

apps/jupyter/notebook-controller/upstream/manager/manager.yaml
image: docker.io/kubeflownotebookswg/notebook-controller

apps/jupyter/notebook-controller/upstream/samples/_v1beta1_notebook.yaml
image: kubeflownotebookswg/jupyter:latest

apps/jupyter/notebook-controller/upstream/samples/_v1_notebook.yaml
image: kubeflownotebookswg/jupyter:latest

apps/jupyter/notebook-controller/upstream/samples/_v1alpha1_notebook.yaml
image: kubeflownotebookswg/jupyter:latest

apps/jupyter/notebook-controller/upstream/default/manager_auth_proxy_patch.yaml
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.4.0

apps/centraldashboard/upstream/base/deployment.yaml
image: docker.io/kubeflownotebookswg/centraldashboard

apps/pvcviewer-controller/upstream/manager/manager.yaml
image: docker.io/kubeflownotebookswg/pvcviewer-controller

apps/pvcviewer-controller/upstream/default/manager_auth_proxy_patch.yaml
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1

apps/kfp-tekton/upstream/v1/third-party/metacontroller/base/stateful-set.yaml
image: 'docker.io/metacontrollerio/metacontroller:v2.0.4'

apps/kfp-tekton/upstream/v1/third-party/prometheus/prometheus-deployment.yaml
image: prom/prometheus

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/quick-start/postgres/postgres-deployment.yaml
image: postgres:12-alpine

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/quick-start/base/prometheus/prometheus-deployment.yaml
image: prom/prometheus

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/quick-start/base/minio/minio-deploy.yaml
image: minio/minio

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/quick-start/mysql/mysql-deployment.yaml
image: mysql:8

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/quick-start/sso/dex/dex-deploy.yaml
image: quay.io/dexidp/dex:v2.23.0

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/base/argo-server/argo-server-deployment.yaml
image: quay.io/argoproj/argocli:latest

apps/kfp-tekton/upstream/v1/third-party/argo/upstream/manifests/base/workflow-controller/workflow-controller-deployment.yaml
image: quay.io/argoproj/workflow-controller:latest

apps/kfp-tekton/upstream/v1/third-party/argo/base/workflow-controller-deployment-patch.yaml
image: gcr.io/ml-pipeline/workflow-controller:v3.3.8-license-compliance

apps/kfp-tekton/upstream/v1/third-party/tekton/upstream/manifests/base/tektoncd-dashboard/tekton-dashboard-release.yaml
image: gcr.io/tekton-releases/github.com/tektoncd/dashboard/cmd/dashboard:v0.35.0@sha256:057471aa317c900e30178d64d83fc9d32cf2fcd718632243f7b902403b64981b

apps/kfp-tekton/upstream/v1/third-party/tekton/upstream/manifests/base/tektoncd-install/tekton-release.yaml
image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.47.1@sha256:9336443dc0b28585a8f75bb9d56082b69fcc61b0e92e968f8cd2ac4dd1f781c5
image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.47.1@sha256:e68ab3f4efa096a4aa96fec0bc8fd91ee2d7a4bcf671ae0c90b2345cd0cb89c7
image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.47.1@sha256:0f045f5e9a9bc025ab1e66909476616a7c2d69d0b0fcf2fbbeefdc8c99d8fd5b

apps/kfp-tekton/upstream/v1/third-party/tekton-custom-task/pipeline-loops/500-webhook.yaml
image: quay.io/aipipeline/pipelineloop-webhook:nightly

apps/kfp-tekton/upstream/v1/third-party/tekton-custom-task/pipeline-loops/500-controller.yaml
image: quay.io/aipipeline/pipelineloop-controller:nightly

apps/kfp-tekton/upstream/v1/third-party/grafana/grafana-deployment.yaml
image: grafana/grafana:5.3.4

apps/kfp-tekton/upstream/v1/third-party/minio/base/minio-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/kfp-tekton/upstream/v1/third-party/kfp-csi-s3/csi-s3-deployment.yaml
image: "k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.3.0"
image: "quay.io/datashim-io/csi-s3:latest"
image: "k8s.gcr.io/sig-storage/csi-attacher:v3.3.0"
image: "k8s.gcr.io/sig-storage/csi-provisioner:v2.2.2"

apps/kfp-tekton/upstream/v1/third-party/mysql/base/mysql-deployment.yaml
image: gcr.io/ml-pipeline/mysql:8.0.26

apps/kfp-tekton/upstream/v1/third-party/application/application-controller-deployment.yaml
image: gcr.io/ml-pipeline/application-crd-controller:1.0-beta-non-cluster-role

apps/kfp-tekton/upstream/v1/base/pipeline/ml-pipeline-persistenceagent-deployment.yaml
image: gcr.io/ml-pipeline/persistenceagent:dummy

apps/kfp-tekton/upstream/v1/base/pipeline/ml-pipeline-scheduledworkflow-deployment.yaml
image: gcr.io/ml-pipeline/scheduledworkflow:dummy

apps/kfp-tekton/upstream/v1/base/pipeline/metadata-writer/metadata-writer-deployment.yaml
image: gcr.io/ml-pipeline/metadata-writer:dummy

apps/kfp-tekton/upstream/v1/base/pipeline/ml-pipeline-ui-deployment.yaml
image: gcr.io/ml-pipeline/frontend:dummy

apps/kfp-tekton/upstream/v1/base/pipeline/ml-pipeline-viewer-crd-deployment.yaml
image: gcr.io/ml-pipeline/viewer-crd-controller:dummy

apps/kfp-tekton/upstream/v1/base/pipeline/ml-pipeline-apiserver-deployment.yaml
image: gcr.io/ml-pipeline/api-server:dummy

apps/kfp-tekton/upstream/v1/base/pipeline/kfp-pipeline-config.yaml
artifact_image: "minio/mc:RELEASE.2020-11-25T23-04-07Z"
moveresults_image: "busybox:1.34.1"

apps/kfp-tekton/upstream/v1/base/pipeline/ml-pipeline-visualization-deployment.yaml
image: gcr.io/ml-pipeline/visualization-server:dummy

apps/kfp-tekton/upstream/v1/base/cache-deployer/cache-deployer-deployment.yaml
image: gcr.io/ml-pipeline/cache-deployer:dummy

apps/kfp-tekton/upstream/v1/base/metadata/base/metadata-grpc-deployment.yaml
image: gcr.io/tfx-oss-public/ml_metadata_store_server:1.5.0

apps/kfp-tekton/upstream/v1/base/metadata/base/metadata-envoy-deployment.yaml
image: gcr.io/ml-pipeline/metadata-envoy:dummy

apps/kfp-tekton/upstream/v1/base/metadata/overlays/db/metadata-db-deployment.yaml
image: mysql:8.0.3

apps/kfp-tekton/upstream/v1/base/installs/multi-user/pipelines-profile-controller/deployment.yaml
image: python:3.7

apps/kfp-tekton/upstream/v1/base/installs/multi-user/pipelines-profile-controller/sync.py
visualization_server_image: gcr.io/ml-pipeline/visualization-server
frontend_image: gcr.io/ml-pipeline/frontend

apps/kfp-tekton/upstream/v1/base/cache/cache-deployment.yaml
image: gcr.io/ml-pipeline/cache-server:dummy

apps/kfp-tekton/upstream/v1/env/azure/minio-azure-gateway/minio-azure-gateway-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/kfp-tekton/upstream/v1/env/gcp/inverse-proxy/proxy-deployment.yaml
image: gcr.io/ml-pipeline/inverse-proxy-agent:dummy

apps/kfp-tekton/upstream/v1/env/gcp/minio-gcs-gateway/minio-gcs-gateway-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/kfp-tekton/upstream/v1/env/gcp/cloudsql-proxy/cloudsql-proxy-deployment.yaml
image: gcr.io/cloudsql-docker/gce-proxy:1.25.0

apps/kfp-tekton/upstream/third-party/metacontroller/base/stateful-set.yaml
image: 'docker.io/metacontrollerio/metacontroller:v2.0.4'

apps/kfp-tekton/upstream/third-party/prometheus/prometheus-deployment.yaml
image: prom/prometheus

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/quick-start/postgres/postgres-deployment.yaml
image: postgres:12-alpine

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/quick-start/base/prometheus/prometheus-deployment.yaml
image: prom/prometheus

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/quick-start/base/minio/minio-deploy.yaml
image: minio/minio

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/quick-start/mysql/mysql-deployment.yaml
image: mysql:8

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/quick-start/sso/dex/dex-deploy.yaml
image: quay.io/dexidp/dex:v2.23.0

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/base/argo-server/argo-server-deployment.yaml
image: quay.io/argoproj/argocli:latest

apps/kfp-tekton/upstream/third-party/argo/upstream/manifests/base/workflow-controller/workflow-controller-deployment.yaml
image: quay.io/argoproj/workflow-controller:latest

apps/kfp-tekton/upstream/third-party/argo/base/workflow-controller-deployment-patch.yaml
image: gcr.io/ml-pipeline/workflow-controller:v3.3.10-license-compliance

apps/kfp-tekton/upstream/third-party/tekton/upstream/manifests/base/tektoncd-dashboard/tekton-dashboard-release.yaml
image: gcr.io/tekton-releases/github.com/tektoncd/dashboard/cmd/dashboard:v0.32.0@sha256:f11d6bdd2a1f2bb1e6366e416741ce1754a7695837305c054a0167f3f7b055ca

apps/kfp-tekton/upstream/third-party/tekton/upstream/manifests/base/tektoncd-install/tekton-release.yaml
image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.47.3@sha256:cfbca9c19a8e7fe4f68b80499c9d921a03240ae2185d6f7d536c33b1177138ca
image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.47.3@sha256:ea46db5fd1c6c1774762fee57cb49aef6a9a6ba862c85232c8a89f1ab67b43fd
image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.47.3@sha256:20fe883b019e80fecddbb97a86d6773925c7b6727cf5e8e7007c47416bd9ebf7

apps/kfp-tekton/upstream/third-party/tekton-custom-task/kfptask/500-webhook.yaml
image: tekton-kfptask-webhook:dummy

apps/kfp-tekton/upstream/third-party/tekton-custom-task/kfptask/500-controller.yaml
image: tekton-kfptask-controller:dummy

apps/kfp-tekton/upstream/third-party/tekton-custom-task/pipeline-loops/500-webhook.yaml
image: quay.io/aipipeline/pipelineloop-webhook:nightly

apps/kfp-tekton/upstream/third-party/tekton-custom-task/pipeline-loops/500-controller.yaml
image: quay.io/aipipeline/pipelineloop-controller:nightly

apps/kfp-tekton/upstream/third-party/tekton-custom-task/driver-controller/500-controller.yaml
image: kfp-v2-dev-driver-controller:dummy

apps/kfp-tekton/upstream/third-party/tekton-custom-task/exit-handler/500-webhook.yaml
image: tekton-exithandler-webhook:dummy

apps/kfp-tekton/upstream/third-party/tekton-custom-task/exit-handler/500-controller.yaml
image: tekton-exithandler-controller:dummy

apps/kfp-tekton/upstream/third-party/grafana/grafana-deployment.yaml
image: grafana/grafana:5.3.4

apps/kfp-tekton/upstream/third-party/minio/base/minio-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/kfp-tekton/upstream/third-party/kfp-csi-s3/csi-s3-deployment.yaml
image: "k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.3.0"
image: "quay.io/datashim-io/csi-s3:latest"
image: "k8s.gcr.io/sig-storage/csi-attacher:v3.3.0"
image: "k8s.gcr.io/sig-storage/csi-provisioner:v2.2.2"

apps/kfp-tekton/upstream/third-party/mysql/base/mysql-deployment.yaml
image: gcr.io/ml-pipeline/mysql:8.0.26

apps/kfp-tekton/upstream/third-party/application/application-controller-deployment.yaml
image: gcr.io/ml-pipeline/application-crd-controller:1.0-beta-non-cluster-role

apps/kfp-tekton/upstream/base/pipeline/ml-pipeline-persistenceagent-deployment.yaml
image: gcr.io/ml-pipeline/persistenceagent:dummy

apps/kfp-tekton/upstream/base/pipeline/ml-pipeline-scheduledworkflow-deployment.yaml
image: gcr.io/ml-pipeline/scheduledworkflow:dummy

apps/kfp-tekton/upstream/base/pipeline/metadata-writer/metadata-writer-deployment.yaml
image: gcr.io/ml-pipeline/metadata-writer:dummy

apps/kfp-tekton/upstream/base/pipeline/ml-pipeline-ui-deployment.yaml
image: gcr.io/ml-pipeline/frontend:dummy

apps/kfp-tekton/upstream/base/pipeline/ml-pipeline-viewer-crd-deployment.yaml
image: gcr.io/ml-pipeline/viewer-crd-controller:dummy

apps/kfp-tekton/upstream/base/pipeline/ml-pipeline-apiserver-deployment.yaml
image: gcr.io/ml-pipeline/api-server:dummy

apps/kfp-tekton/upstream/base/pipeline/ml-pipeline-visualization-deployment.yaml
image: gcr.io/ml-pipeline/visualization-server:dummy

apps/kfp-tekton/upstream/base/cache-deployer/cache-deployer-deployment.yaml
image: gcr.io/ml-pipeline/cache-deployer:dummy

apps/kfp-tekton/upstream/base/metadata/base/metadata-grpc-deployment.yaml
image: gcr.io/tfx-oss-public/ml_metadata_store_server:1.5.0

apps/kfp-tekton/upstream/base/metadata/base/metadata-envoy-deployment.yaml
image: gcr.io/ml-pipeline/metadata-envoy:dummy

apps/kfp-tekton/upstream/base/metadata/overlays/db/metadata-db-deployment.yaml
image: mysql:8.0.3

apps/kfp-tekton/upstream/base/installs/multi-user/pipelines-profile-controller/deployment.yaml
image: python:3.7

apps/kfp-tekton/upstream/base/installs/multi-user/pipelines-profile-controller/sync.py
visualization_server_image: gcr.io/ml-pipeline/visualization-server
frontend_image: gcr.io/ml-pipeline/frontend

apps/kfp-tekton/upstream/base/cache/cache-deployment.yaml
image: gcr.io/ml-pipeline/cache-server:dummy

apps/kfp-tekton/upstream/env/azure/minio-azure-gateway/minio-azure-gateway-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/kfp-tekton/upstream/env/gcp/inverse-proxy/proxy-deployment.yaml
image: gcr.io/ml-pipeline/inverse-proxy-agent:dummy

apps/kfp-tekton/upstream/env/gcp/minio-gcs-gateway/minio-gcs-gateway-deployment.yaml
image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

apps/kfp-tekton/upstream/env/gcp/cloudsql-proxy/cloudsql-proxy-deployment.yaml
image: gcr.io/cloudsql-docker/gce-proxy:1.25.0

apps/admission-webhook/upstream/base/deployment.yaml
image: docker.io/kubeflownotebookswg/poddefaults-webhook

apps/volumes-web-app/upstream/base/deployment.yaml
image: docker.io/kubeflownotebookswg/volumes-web-app
image: filebrowser/filebrowser:latest

apps/katib/upstream/components/postgres/postgres.yaml
image: postgres:14.5-alpine

apps/katib/upstream/components/db-manager/db-manager.yaml
image: docker.io/kubeflowkatib/katib-db-manager

apps/katib/upstream/components/ui/ui.yaml
image: docker.io/kubeflowkatib/katib-ui

apps/katib/upstream/components/controller/controller.yaml
image: docker.io/kubeflowkatib/katib-controller

apps/katib/upstream/components/controller/trial-templates.yaml
image: docker.io/kubeflowkatib/mxnet-mnist:v0.16.0-rc.1
image: docker.io/kubeflowkatib/enas-cnn-cifar10-cpu:v0.16.0-rc.1
image: docker.io/kubeflowkatib/pytorch-mnist-cpu:v0.16.0-rc.1
image: docker.io/kubeflowkatib/pytorch-mnist-cpu:v0.16.0-rc.1

apps/katib/upstream/components/mysql/mysql.yaml
image: mysql:8.0.29

apps/katib/upstream/installs/katib-leader-election/katib-config.yaml
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/tfevent-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperband:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-skopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-enas:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-darts:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-pbt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/earlystopping-medianstop:v0.16.0-rc.1

apps/katib/upstream/installs/katib-cert-manager/katib-config.yaml
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/tfevent-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperband:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-skopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-enas:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-darts:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-pbt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/earlystopping-medianstop:v0.16.0-rc.1

apps/katib/upstream/installs/katib-standalone-postgres/katib-config.yaml
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/tfevent-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperband:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-skopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-enas:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-darts:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-pbt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/earlystopping-medianstop:v0.16.0-rc.1

apps/katib/upstream/installs/katib-external-db/katib-config.yaml
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/tfevent-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperband:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-skopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-enas:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-darts:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-pbt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/earlystopping-medianstop:v0.16.0-rc.1

apps/katib/upstream/installs/katib-openshift/katib-config.yaml
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/tfevent-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperband:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-skopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-enas:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-darts:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-pbt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/earlystopping-medianstop:v0.16.0-rc.1

apps/katib/upstream/installs/katib-standalone/katib-config.yaml
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/file-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/tfevent-metrics-collector:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-hyperband:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-skopt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-goptuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-optuna:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-enas:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-darts:v0.16.0-rc.1
image: docker.io/kubeflowkatib/suggestion-pbt:v0.16.0-rc.1
image: docker.io/kubeflowkatib/earlystopping-medianstop:v0.16.0-rc.1

apps/kubebench/upstream/base/config-map.yaml
image: gcr.io/kubeflow-images-public/kubebench/workflow-agent:bc682c1

apps/kubebench/upstream/base/deployment.yaml
image: gcr.io/kubeflow-images-public/kubebench/kubebench-operator-v1alpha2

tests/gh-actions/kind-cluster.yaml
image: kindest/node:v1.26.6@sha256:5e5d789e90c1512c8c480844e0985bc3b4da4ba66179cc5b540fe5b785ca97b5
image: kindest/node:v1.26.6@sha256:5e5d789e90c1512c8c480844e0985bc3b4da4ba66179cc5b540fe5b785ca97b5
image: kindest/node:v1.26.6@sha256:5e5d789e90c1512c8c480844e0985bc3b4da4ba66179cc5b540fe5b785ca97b5

tests/gh-actions/kf-objects/katib_test.yaml
image: docker.io/kubeflowkatib/mxnet-mnist:latest

tests/gh-actions/kf-objects/tfjob.yaml
image: gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0

tests/gh-actions/kind-cluster-1-24.yaml
image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
image: kindest/node:v1.25.3@sha256:f52781bc0d7a19fb6c405c2af83abfeb311f130707a0e219175677e366cc45d1
image: kindest/node:v1.25.3@sha256:f52781bc0d7a19fb6c405c2af83abfeb311f130707a0e219175677e366cc45d1
image: kindest/node:v1.25.3@sha256:f52781bc0d7a19fb6c405c2af83abfeb311f130707a0e219175677e366cc45d1
```

Install Kubeflow

```bash
cd ~/charts/kubeflow-*
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
```

Configure Endpoint

```bash
kubectl patch svc -n istio-system istio-ingressgateway -p '{"spec": {"type": "NodePort"}}'
```

## 8 Grafana and Prometheus

> **Environment:** Repository Server

Fetch kube-prometheus-stack chart

```bash
cd ~/charts
helm pull nexus-helm/kube-prometheus-stack --untar
cd kube-prometheus-stack
```

Modify Options

```bash
vi values.yaml
```

Before Change

```yaml
serviceMonitorSelectorNilUsesHelmValues: true
podMonitorSelectorNilUsesHelmValues: true
```

After Change

```yaml
serviceMonitorSelectorNilUsesHelmValues: false
podMonitorSelectorNilUsesHelmValues: false
```

Install kube-prometheus-stack

```bash
cd ..
helm install -n kube-prometheus-stack kube-prometheus-stack kube-prometheus-stack --create-namespace
```

Patch NodePort

```bash
kubectl patch svc -n kube-prometheus-stack kube-prometheus-stack-grafana -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc -n kube-prometheus-stack kube-prometheus-stack-prometheus -p '{"spec": {"type": "NodePort"}}'
```

Login Password for Grafana

```yaml
admin / prom-operator
```

## 9 Images

## 10 Sample Pipeline

Code

```python

```

Kubeflow Pipeline

```python

```

## 11 Monitoring Dashboard
