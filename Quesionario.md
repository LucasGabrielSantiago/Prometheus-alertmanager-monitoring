1. Qual é a função do Windows Exporter na arquitetura implementada?
Coleta informações do sistema operacional, como utilização da CPU, e disponibiliza essas métricas para o Prometheus

2. Qual é a responsabilidade do Prometheus no processo de monitoramento?
coletar, armazenar e consultar as métricas da infraestrutura

3. O que significa o valor 1 retornado pela consulta up?
Serviço monitorado está funcionando

4. Qual é a finalidade da função rate() utilizada na consulta PromQL?
Calcula a taxa média de variação de uma métrica durante um determinado período, permitindo analisar a utilização da CPU

5. Qual é a diferença entre os estados PENDING e FIRING?
PENDING: a condição foi identificada, mas ainda não completou o período configurado
FIRING: a condição permaneceu verdadeira pelo tempo exigido e o alerta foi disparado.

6. Qual é a função do parâmetro for: 1m na regra de alerta?
Exige que a condição permaneça verdadeira durante um minuto

7. Qual é a diferença entre as responsabilidades do Prometheus e do Alertmanager?
O Prometheus coleta as métricas e identifica condições de alerta
Alertmanager recebe e gerencia esses alertas.

8. Por que é importante monitorar a utilização da CPU em uma infraestrutura?
Evita problemas de desempenho pela sobrecarga de uso elevado da CPU ajudando antecipar possíveis falhas e manter os serviços funcionando corretamente
