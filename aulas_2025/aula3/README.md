# Laboratório Atualizado: Monitoramento com Prometheus e Grafana no Kubernetes

## Objetivo do Laboratório

Este laboratório prático apresenta um sistema completo de monitoramento para clusters Kubernetes, integrando diversas ferramentas essenciais para observabilidade e elasticidade de aplicações em ambiente de produção:

- **Prometheus** - Sistema de monitoramento que coleta e armazena métricas como séries temporais
- **Grafana** - Plataforma de visualização que transforma os dados do Prometheus em dashboards interativos
- **Metrics Server** - Coletor de métricas que alimenta tanto o monitoramento quanto o escalonamento automático
- **HPA (Horizontal Pod Autoscaler)** - Demonstra como escalar automaticamente aplicações baseado em métricas de utilização
- **Kind** - Cria um cluster Kubernetes em containers locais para fins de laboratório
- **Helm** - Gerenciador de pacotes que simplifica a instalação de aplicações complexas como o Prometheus

Ao final deste laboratório, você será capaz de:
1. Configurar um sistema de monitoramento completo em Kubernetes
2. Visualizar métricas de cluster, nodes e aplicações em dashboards interativos
3. Implementar escalonamento automático baseado em métricas
4. Executar testes de carga e observar o comportamento do sistema em tempo real

Este guia demonstra como configurar e utilizar o Prometheus e Grafana para monitoramento em um cluster Kubernetes. Ele abrange:
- Criação de um cluster Kind
- Instalação do Metrics Server
- Configuração do Prometheus e Grafana
- Exemplos de consultas PromQL
- Teste de carga com uma aplicação httpbin e HPA (Horizontal Pod Autoscaler)

✅ **Data da Atualização: 21 de Maio de 2025**

---

## 1. Configuração do Ambiente

### 1.1. Instalar o Helm
```bash
# Instalar o Helm (execute na instância EC2 da AWS Academy)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash

# Verificar a instalação
sudo helm version
```

### 1.2. Criar Cluster Kind
Primeiro, crie um arquivo chamado `kind-config.yaml` com o seguinte conteúdo exato:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: aulathree
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 31090
        hostPort: 9090
        protocol: TCP
      - containerPort: 32000
        hostPort: 3000
        protocol: TCP 
```

Depois, crie o cluster (certifique-se de estar no diretório onde salvou o arquivo):
```bash
# Crie um cluster no KIND
sudo kind create cluster --config kind-config.yaml

# Verifique se o cluster foi criado corretamente
sudo kubectl get nodes
```

### 1.3. Instalar o Metrics Server
```bash
# Aplique o manifesto do Metrics Server
sudo kubectl apply -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/metricserverfull.yaml

# Aguarde o Metrics Server subir completamente (pode levar alguns segundos)
sudo kubectl get pods -n kube-system | grep metrics-server
# Verifique se aparece "1/1 Running" na saída

# Espere mais uns 30 segundos e verifique se o Metrics Server está funcionando corretamente
sudo kubectl top nodes
# Se mostrar estatísticas de CPU e memória, está funcionando
```

---

## 2. Instalação do Prometheus e Grafana

### 2.1. Adicionar Repositórios do Helm
```bash
# Verifique se o cluster está acessível
sudo kubectl cluster-info
# Deve mostrar o endereço do servidor API Kubernetes

# Adicione os repositórios do Helm para Prometheus e Grafana
sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Verifique se o repositório foi adicionado corretamente
sudo helm repo list
# Deve aparecer "prometheus-community" na lista

# Atualize os repositórios
sudo helm repo update

# Verifique se o repositório está atualizado
sudo helm search repo prometheus-community
# Deve mostrar uma lista de charts disponíveis
```

### 2.2. Instalar o Prometheus Operator
```bash
# Verifique se o cluster está acessível novamente
sudo kubectl cluster-info

# Instale o Prometheus Operator no seu cluster
sudo helm install prometheus prometheus-community/kube-prometheus-stack \
  --set prometheus.prometheusSpec.maximumStartupDurationSeconds=60

# Verifique se a instalação foi iniciada
sudo helm list
# Deve mostrar "prometheus" na lista

# Espere todos os componentes subirem antes de acessar o Grafana (pode levar 2-3 minutos)
sudo kubectl --namespace default get pods -l "release=prometheus"

# Verifique se todos os pods estão em estado Running
# Aguarde até que todos os pods estejam prontos (READY 1/1 ou 2/2 dependendo do pod)
# Se algum pod mostrar "0/1" ou "Pending", espere mais um pouco e execute o comando novamente
```

---

## 3. Acesso e Uso do Prometheus

### 3.1. Acessar o Prometheus
```bash
# Verifique se o serviço do Prometheus está disponível
sudo kubectl get svc prometheus-kube-prometheus-prometheus
# Inicialmente, o serviço é do tipo ClusterIP, precisamos alterá-lo para NodePort

sudo kubectl patch svc prometheus-kube-prometheus-prometheus \
  -p '{
    "spec": {
      "type": "NodePort",
      "ports": [
        {
          "name": "http-web",
          "port": 9090,
          "targetPort": 9090,
          "nodePort": 31090,
          "protocol": "TCP"
        }
      ]
    }
  }'
# Use o número da porta retornado acima para acessar o Prometheus
# No seu navegador, acesse: http://<IP_PUBLICO_DA_EC2>:<PORTA>
# Substitua <IP_PUBLICO_DA_EC2> pelo endereço IP público da sua instância EC2
# Substitua <PORTA> pelo número retornado no comando anterior
```

### 3.2. Exemplos de Consultas PromQL

Depois de acessar a interface do Prometheus no navegador, siga estes passos para executar consultas:

1. Na parte superior da interface, localize a barra de consulta longa
2. Cole uma das consultas abaixo na barra
3. Clique no botão azul "Execute" à direita da barra
4. Veja o resultado em formato tabela ou clique na aba "Graph" para visualizar em gráfico
5. Para gráficos, ajuste o intervalo de tempo no canto superior direito (1h, 6h, etc.)

**Consultas básicas para testar (em ordem crescente de complexidade):**

```
# Verifica quais targets estão sendo monitorados pelo Prometheus (deve ser o primeiro teste)
up

# Conta quantos pods estão rodando no cluster
count(kube_pod_info)

# Lista todos os pods em execução com detalhes
kube_pod_info

# Mostra o status de todos os pods (Running, Pending, etc.)
kube_pod_status_phase

# Conta pods por namespace
count(kube_pod_info) by (namespace)

# Uso de CPU por pod (últimos 5 minutos)
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)

# Uso de memória por pod em bytes
sum(container_memory_usage_bytes) by (pod)

# Carga de CPU dos nodes (em porcentagem)
(1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100
```

**Observações importantes:**
- Algumas métricas podem não estar disponíveis imediatamente após a instalação
- Para descobrir as métricas disponíveis, clique no botão "Insert metric at cursor" na barra de consulta
- Para visualizar mudanças durante o teste de carga, use a aba "Graph" e defina um intervalo de tempo curto

---

## 4. Configuração e Uso do Grafana

### 4.1. Obter a Senha do Grafana
```bash
sudo kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 4.2. Acessar o Grafana
```bash
# Verifique se o serviço do Grafana está disponível
sudo kubectl get svc prometheus-grafana
# Inicialmente, o serviço é do tipo ClusterIP, precisamos alterá-lo para NodePort
# Altere o serviço para NodePort para acessar externamente

sudo kubectl patch svc prometheus-grafana \
  -p '{
    "spec": {
      "type": "NodePort",
      "ports": [
        {
          "name": "http-web-test",
          "port": 3000,
          "targetPort": 3000,
          "nodePort": 32000,
          "protocol": "TCP"
        }
      ]
    }
  }'

**Por que usar `kubectl patch`?**
- O Helm chart do Prometheus/Grafana cria serviços como ClusterIP por padrão
- Não há uma opção simples no Helm para forçar NodePort durante a instalação
- O `kubectl patch` permite modificar serviços existentes sem recriar recursos
- É essencial no ambiente AWS Academy para acessar via IP público da EC2

**Acesse via navegador:**
```
http://<IP_PUBLICO_DA_EC2>:3000
```

**Login:** admin  
**Senha:** Use o comando da seção 4.1 para obter a senha
```
### 4.4. Dashboards do Grafana

#### 4.4.1. Visualizar Dashboards Pré-instalados (Recomendado para iniciantes)
O Prometheus Operator já instala automaticamente um conjunto completo de dashboards para monitoramento do Kubernetes:

1. No menu lateral esquerdo, clique em "Dashboards" (ícone de quatro quadrados).
2. Clique em "Browse" (ou em "Browsable" na barra lateral).
3. Procure por dashboards que começam com "Kubernetes / ":
   - **Kubernetes / Compute Resources / Cluster**: Visão geral do uso de recursos do cluster
   - **Kubernetes / Compute Resources / Namespace**: Uso de recursos por namespace
   - **Kubernetes / Compute Resources / Node**: Uso de recursos por nó
   - **Kubernetes / Compute Resources / Pod**: Uso de recursos por pod
   - **Kubernetes / Compute Resources / Workload**: Uso de recursos por workload
   - **Kubernetes / Networking**: Métricas de rede por cluster, pods, namespaces, etc.
4. Clique em qualquer um desses dashboards para explorar as métricas já disponíveis.

**Dica**: Se os dashboards padrão não aparecerem, aguarde alguns minutos para que o Prometheus colete dados suficientes. Certifique-se de que todos os pods do Prometheus e Grafana estão em estado "Running" (execute `sudo kubectl get pods`).

#### 4.4.2. Importar Dashboards Adicionais (Opcional)
Se quiser explorar dashboards alternativos criados pela comunidade:



1. No menu lateral esquerdo, clique em "Dashboards" (ícone de quatro quadrados).
2. Clique em "+ Import" no topo da página.
3. Na tela de importação, você verá várias opções:
   - Na seção "Import via grafana.com", digite um dos seguintes IDs:
     - **ID 22128** HPA
     - **ID 315:** Kubernetes cluster monitoring (por CoreOS).
     - **ID 12740:** Kubernetes Monitoring (por Bitnami).
   - Clique no botão "Load".
4. Na próxima tela, você verá configurações do dashboard:
   - Na seção "Prometheus", selecione sua fonte de dados Prometheus no menu dropdown.
   - **IMPORTANTE:** Esta seleção da fonte de dados é obrigatória para o dashboard funcionar.
   - Se houver outros campos de seleção, mantenha os valores padrão.
5. Clique no botão "Import" na parte inferior da tela.
6. O dashboard será importado e mostrado automaticamente.

### 4.5. Analisando os Dashboards do Kubernetes

Para extrair o máximo dos dashboards:

1. Na parte superior direita de qualquer dashboard, ajuste o intervalo de tempo para "Last 15 minutes" para ver dados recentes.

2. Principais painéis para verificar em "Kubernetes / Compute Resources / Cluster":
   - **CPU Usage**: Mostra o uso de CPU por namespace (deve mostrar o namespace padrão/default)
   - **Memory Usage**: Mostra o uso de memória por namespace
   - **CPU Quota**: Mostra o uso vs. limites de CPU (útil para identificar pods throttled)
   - **Memory Quota**: Mostra o uso vs. limites de memória (útil para identificar pods com risco de OOM)

3. Em "Kubernetes / Compute Resources / Pod", você pode:
   - Filtrar por namespace (canto superior esquerdo)
   - Ver métricas detalhadas de CPU e memória por pod
   - Identificar pods com alta utilização de recursos

4. Para análise de rede, use os dashboards "Kubernetes / Networking".

**Solução de Problemas:**
- **Dashboard vazio ou sem dados?** Verifique se os pods do Prometheus e coletor de métricas estão rodando: `sudo kubectl get pods -A | grep -E 'prometheus|metrics'`
- **Métricas parciais?** Algumas métricas levam mais tempo para aparecer. Aguarde pelo menos 5 minutos após a instalação.
- **Erro na fonte de dados?** Verifique a configuração da fonte de dados em Configuration > Data Sources.

---

## 5. Teste de Carga com httpbin e HPA

### 5.1. Implantar httpbin e Configurar HPA
```bash
# Implantar o httpbin
sudo kubectl apply -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/deploymentexample.yaml

# Verifique se o deployment foi criado
sudo kubectl get deployments

# Suba o service NodePort:
sudo kubectl apply -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/servicehttpbin-nodeport.yaml

# Verifique se o serviço está disponível
sudo kubectl get svc httpbin-service

# Suba o pod de stress test
sudo kubectl apply -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/pod.yaml

# Aguarde o pod de stress test estar pronto
sudo kubectl get pods | grep ab-stress

# Suba o HPA
sudo kubectl apply -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/hpa-full.yaml

# Verifique se o HPA foi criado
sudo kubectl get hpa
```

### 5.2. Executar Teste de Carga
```bash
# Verifique se o pod de stress test está pronto antes de executar o teste
sudo kubectl get pods | grep ab-stress

# Obter a porta do serviço httpbin
sudo kubectl get svc httpbin-service -o jsonpath='{.spec.ports[0].nodePort}'
# Use o número da porta retornado acima para o teste de carga
# Exemplo: http://<IP_PUBLICO_DA_EC2>:<PORTA>

# Comando stress test
sudo kubectl exec -it ab-stress -- ab -n 10000 -c 100 http://svc/get

# Durante o teste, monitore o HPA em outro terminal
sudo kubectl get hpa -w

# Observe os dashboards no Grafana para monitorar o comportamento do sistema durante o teste
```

---

## 6. Limpeza dos Recursos
```bash
# Para limpar os recursos:
sudo helm delete prometheus
sudo kubectl delete -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/servicehttpbin-nodeport.yaml
sudo kubectl delete -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/deploymentexample.yaml
sudo kubectl delete -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/pod.yaml
sudo kubectl delete -f https://raw.githubusercontent.com/able2cloud/continuous_monitoring_log_analytics/main/aulas_2025/aula3/hpa-full.yaml

sudo kind delete cluster --name aulathree
``` 
