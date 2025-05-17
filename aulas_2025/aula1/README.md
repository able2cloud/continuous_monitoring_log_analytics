# Laboratório Atualizado: Kubernetes Local com Kind (e Pré-requisitos Docker)

Este guia mostra como configurar um ambiente Kubernetes local usando **Kind**. Ele cobre:
- Instalação dos pré-requisitos essenciais: Docker e Docker Compose
- Instalação do Kind, Kubectl e Helm
- Implantação de uma aplicação Nginx simples em um namespace

✅ **Data da Atualização: 7 de Abril de 2025**

---

## 1. Instalação dos Pré-requisitos Essenciais

### 1.1. Docker e Docker Compose

**Windows**  
Recomendado: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Baixe e instale normalmente. Ative o WSL 2 (recomendado) ou Hyper-V se solicitado.
- Reinicie e verifique:

```powershell
docker --version
docker compose version
```

Alternativa via [Chocolatey](https://chocolatey.org/install):
```powershell
choco install docker-desktop
```

**macOS**  
Recomendado: [Docker Desktop para Mac](https://www.docker.com/products/docker-desktop/)
- Baixe o .dmg, mova para Aplicativos, inicie o Docker e aceite permissões.

Verifique:
```bash
docker --version
docker compose version
```

Alternativa via Homebrew:
```bash
brew install --cask docker
```

**Linux (Ubuntu/Debian)**

Remova versões antigas (se houver):
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```
Atualize pacotes e instale pré-requisitos:
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```
Adicione a chave e repositório oficial:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Instale o Docker Engine e Compose Plugin:
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Pós-instalação:
```bash
sudo usermod -aG docker $USER
# Reinicie ou execute:
newgrp docker
```
Verificação:
```bash
docker --version
docker compose version
sudo systemctl status docker
docker run hello-world
```

Referência oficial: https://docs.docker.com/engine/install/ubuntu/

---

### 1.2. Gerenciador de Pacotes (opcional)

**Windows**: Chocolatey  
https://chocolatey.org/install

**macOS**: Homebrew  
https://brew.sh/

---

## 2. Instalação das Ferramentas Kubernetes

### 2.1. Kind (Kubernetes in Docker)

**Windows**:
```powershell
choco install kind
```
**macOS**:
```bash
brew install kind
```
**Linux**:
```bash
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64"
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
Verifique:
```bash
kind --version
```

### 2.2. Kubectl

**Windows**:
```powershell
choco install kubernetes-cli
```
**macOS**:
```bash
brew install kubectl
```
**Linux**:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
Verificação:
```bash
kubectl version --client
```

### 2.3. Helm (Gerenciador de Pacotes Kubernetes)

**Windows**:
```powershell
choco install kubernetes-helm
```
**macOS**:
```bash
brew install helm
```
**Linux (via script oficial)**:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
Verificação:
```bash
helm version
```

---

## 3. Criação e Interação com o Cluster Kind

### 3.1. Criar o Cluster
```bash
kind create cluster --config kind-config.yaml
```

### 3.2. Verificar o Cluster
```bash
kubectl cluster-info --context kind-aulaone
kubectl get nodes
kind get clusters
```

### 3.3. Criar um Namespace
```bash
kubectl create namespace aula1
kubectl get namespaces
```

---

## 4. Implantar uma Aplicação Nginx de Exemplo

### 4.1. Criar o Manifesto (nginx-app.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: aula1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: aula1
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### 4.2. Aplicar o Manifesto
```bash
kubectl apply -f nginx-app.yaml
```

### 4.3. Verificar a Aplicação
```bash
kubectl get deployment -n aula1
kubectl get pods -n aula1 -o wide
kubectl get service -n aula1
```

### 4.4. Acessar o Nginx via Port Forward

Abra o navegador em: http://<IP_PUBLICO_DA_EC2>:30080  
Para encerrar: `Ctrl + C`

---

## 5. Limpeza (Opcional)
```bash
kind delete cluster --name aulaone
```

---

Com isso, você tem um ambiente Kubernetes local funcional, com Docker, Kind, Kubectl e Helm configurados, e uma aplicação rodando no namespace `aula1`.

Referências oficiais estão linkadas ao longo do guia para aprofundamento.
