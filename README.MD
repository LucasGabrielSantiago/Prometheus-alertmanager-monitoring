# Laboratório — Monitoramento de Infraestrutura com Prometheus e Alertmanager

## Sobre o projeto

Este laboratório tem como objetivo implementar uma solução de monitoramento de infraestrutura utilizando **Prometheus, Alertmanager, Windows Exporter e Docker**.

A atividade consiste em monitorar a utilização da CPU de uma máquina Windows, coletar métricas do sistema operacional, realizar consultas utilizando PromQL e configurar alertas automáticos para identificar situações de alta utilização do processador.

O ambiente é executado localmente no Windows, utilizando Docker Desktop para gerenciar os serviços de monitoramento.

## Objetivos

- Compreender a arquitetura de monitoramento do Prometheus.
- Instalar e configurar o Windows Exporter.
- Integrar o Windows Exporter ao Prometheus.
- Coletar e consultar métricas de infraestrutura.
- Utilizar PromQL para analisar a utilização da CPU.
- Criar regras de alerta para identificar alta utilização do processador.
- Integrar o Prometheus ao Alertmanager.
- Simular sobrecarga de CPU e acompanhar o ciclo de vida dos alertas.

## Tecnologias utilizadas

| Tecnologia | Finalidade |
|---|---|
| Prometheus | Coleta e armazenamento de métricas |
| Alertmanager | Gerenciamento de alertas |
| Windows Exporter | Exportação de métricas do Windows |
| Docker Desktop | Execução dos serviços em containers |
| Docker Compose | Orquestração dos containers |
| PowerShell | Execução de comandos no Windows |
| PromQL | Consulta e análise de métricas |

## Arquitetura do laboratório

O funcionamento do ambiente segue o seguinte fluxo:

1. O Windows Exporter coleta e disponibiliza métricas do sistema operacional.
2. O Prometheus consulta o endpoint do Windows Exporter.
3. As métricas são armazenadas e disponibilizadas para consultas PromQL.
4. O Prometheus avalia as regras de alerta configuradas.
5. Quando uma condição de alerta é identificada, o Prometheus encaminha o alerta ao Alertmanager.
6. O Alertmanager recebe e gerencia os alertas.

## Pré-requisitos

Antes de iniciar, é necessário possuir:

- Windows 10 ou Windows 11.
- Docker Desktop instalado e funcionando.
- PowerShell.
- Conexão com a internet.
- Permissão para instalar aplicações no Windows.

## Estrutura do projeto

```text
prometheus-lab/
│
├── docker-compose.yml
│
├── prometheus/
│   ├── prometheus.yml
│   └── alerts.yml
│
└── alertmanager/
    └── alertmanager.yml
```

## Etapas de implementação

### 1. Instalação do Windows Exporter

O Windows Exporter é responsável por disponibilizar métricas do sistema operacional para o Prometheus.

Download oficial:

https://github.com/prometheus-community/windows_exporter/releases

Após instalar, verifique o serviço pelo PowerShell:

```powershell
Get-Service windows_exporter
```

Teste o endpoint de métricas:

```powershell
Invoke-WebRequest http://localhost:9182/metrics -UseBasicParsing
```

**Resultado esperado:** resposta HTTP 200.

### 2. Preparação do ambiente

Crie as pastas necessárias:

```powershell
mkdir C:\prometheus-lab
cd C:\prometheus-lab

mkdir prometheus
mkdir alertmanager
```

### 3. Configuração do Prometheus

Crie o arquivo `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

rule_files:
  - /etc/prometheus/alerts.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "windows"
    static_configs:
      - targets: ["host.docker.internal:9182"]
```

Essa configuração define o intervalo de coleta, os arquivos de regras e os endpoints monitorados.

### 4. Configuração do Alertmanager

Crie o arquivo `alertmanager/alertmanager.yml`:

```yaml
route:
  receiver: "default"

receivers:
  - name: "default"
```

Essa configuração estabelece um receptor padrão para os alertas, sem integração com serviços externos de notificação.

### 5. Configuração do Docker Compose

Crie o arquivo `docker-compose.yml` na raiz do projeto:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
    depends_on:
      - alertmanager
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    restart: unless-stopped

volumes:
  prometheus-data:
```

Inicie os serviços:

```powershell
docker compose up -d
```

Verifique os containers:

```powershell
docker compose ps
```

### 6. Acessando as interfaces

| Serviço | Endereço |
|---|---|
| Prometheus | http://localhost:9090 |
| Prometheus Targets | http://localhost:9090/targets |
| Prometheus Alerts | http://localhost:9090/alerts |
| Prometheus Rules | http://localhost:9090/rules |
| Alertmanager | http://localhost:9093 |

Os dois targets devem apresentar o estado `UP`.

### 7. Consultas utilizando PromQL

Algumas consultas utilizadas no laboratório:

**Verificar o estado dos targets:**

```promql
up
```

**Consultar somente o Windows:**

```promql
up{job="windows"}
```

**Identificar métricas relacionadas à CPU:**

```promql
count by (__name__) ({job="windows", __name__=~".*cpu.*"})
```

**Calcular a utilização percentual da CPU:**

```promql
100 * (1 - avg by (instance) (
  rate(windows_cpu_time_seconds_total{mode="idle"}[5m])
))
```

A última consulta calcula a porcentagem de utilização da CPU considerando o tempo em que o processador não permaneceu ocioso.

### 8. Criação da regra de alerta

Crie o arquivo `prometheus/alerts.yml`:

```yaml
groups:
  - name: windows-cpu-alerts
    rules:
      - alert: HighCPUUsage
        expr: |
          100 * (1 - avg by (instance) (
            rate(windows_cpu_time_seconds_total{mode="idle"}[5m])
          )) > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Utilizacao elevada da CPU"
          description: "A CPU do Windows esta acima de 80% por mais de 1 minuto."
```

A regra verifica se a utilização da CPU ultrapassa 80% durante pelo menos um minuto.

Reinicie o Prometheus para carregar a configuração:

```powershell
docker compose restart prometheus
```

Verifique a regra:

http://localhost:9090/rules

### 9. Simulação de sobrecarga da CPU

Para testar o funcionamento do monitoramento, execute no PowerShell:

```powershell
$jobs = 1..8 | ForEach-Object {
    Start-Job -ScriptBlock {
        while ($true) {
            [Math]::Sqrt((Get-Random -Minimum 1 -Maximum 1000000)) |
            Out-Null
        }
    }
}
```

O comando inicia oito tarefas em segundo plano realizando cálculos continuamente, aumentando a utilização do processador.

Acompanhe o comportamento do alerta:

http://localhost:9090/alerts

Os alertas podem apresentar três estados:

- **INACTIVE:** a condição de alerta não foi satisfeita.
- **PENDING:** a condição foi identificada, mas ainda não completou o período configurado.
- **FIRING:** a condição permaneceu verdadeira pelo tempo exigido e o alerta foi disparado.

Depois, acesse o Alertmanager:

http://localhost:9093

Verifique se o alerta `HighCPUUsage` foi recebido.

### 10. Encerramento do laboratório

Após finalizar a simulação, interrompa os processos de carga:

```powershell
Get-Job | Stop-Job
Get-Job | Remove-Job
```

Aguarde a normalização da utilização da CPU e acompanhe a resolução do alerta.

Para encerrar os containers:

```powershell
docker compose down
```

## Evidências do laboratório

As seguintes evidências devem ser registradas durante a execução:

- [ ] Windows Exporter funcionando com resposta HTTP 200.
- [ ] Prometheus Targets demonstrando os dois alvos em estado UP.
- [ ] Consulta PromQL apresentando a utilização percentual da CPU.
- [ ] Arquivo `alerts.yml` utilizado na atividade.
- [ ] Regra `HighCPUUsage` carregada no Prometheus.
- [ ] Alerta em estado FIRING.
- [ ] Alertmanager demonstrando o recebimento do alerta.
- [ ] Respostas das oito questões de fixação.


## Resultados esperados

Ao concluir o laboratório, espera-se ter um ambiente funcional de monitoramento capaz de:

- Coletar métricas de uma máquina Windows.
- Monitorar a utilização da CPU em tempo real.
- Consultar métricas utilizando PromQL.
- Identificar condições de alta utilização do processador.
- Gerar alertas automaticamente.
- Encaminhar alertas ao Alertmanager.
- Acompanhar os estados e a resolução dos alertas.

## Considerações finais

O laboratório proporcionou uma experiência prática com ferramentas utilizadas em ambientes de infraestrutura e DevOps.

A integração entre Prometheus, Alertmanager e Windows Exporter demonstra como é possível acompanhar recursos computacionais, identificar problemas de desempenho e implementar mecanismos de alerta.

O conhecimento adquirido pode ser aplicado em ambientes de desenvolvimento, servidores e infraestruturas que necessitam de monitoramento contínuo.

---

**Projeto desenvolvido para fins educacionais — Laboratório de Monitoramento de Infraestrutura.**