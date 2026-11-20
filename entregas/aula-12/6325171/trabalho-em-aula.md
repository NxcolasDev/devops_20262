# Trabalho em Aula — Aula 12: Rollback Strategies e AIOps

**Aluno:** Nicolas de Jesus Silva  
**RA:** 6325171  
**Data:** 19/11/2026

## Parte 1 — Análise de Cenário de Incidente

1. **Severidade: P1.** A latência cresceu de 150 ms para 1200 ms, os pedidos criados caíram de 42 para 2 por intervalo e a API passou a rejeitar pedidos por timeout e overflow da fila. Há perda direta de pedidos e impacto amplo em produção, portanto a resposta deve ser imediata.

2. **Causa raiz provável:** falha ou degradação do serviço externo `payment-gateway`, possivelmente ligada à nova integração V2. Os erros começam por timeout nesse gateway, seguido de abertura do circuit breaker e rejeição da fila.

3. **Evidências:**
   - A sequência de logs mostra timeouts consecutivos no `payment-gateway`, encerrando com `Circuit breaker OPEN`.
   - A latência dos pedidos acompanha o aumento dos timeouts e chega a 1200 ms.
   - `OrdersCreated` cai de 42 para 2 enquanto `Error5xx` sobe para 30, indicando que a degradação externa está impedindo a conclusão de pedidos.

4. **Ação imediata:** primeiro reduzir o tráfego para a integração problemática e ativar o fallback seguro por feature flag ou kill switch. Em paralelo, verificar logs, métricas do gateway e tamanho da fila. Também abrir P1, congelar novos deploys e preservar evidências.

5. **Rollback necessário? Sim, mas como segunda decisão.** O rollback é necessário se o problema começou após a implantação da integração V2 ou se a desativação da feature não recuperar o fluxo. Desligar `FF_PAYMENT_GATEWAY_V2` é mais rápido e pode tratar a causa sem derrubar toda a aplicação.

6. **Feature flag:** com `FF_PAYMENT_GATEWAY_V2=true`, eu mudaria a flag para `false` para retornar imediatamente ao gateway estável ou para o fluxo legado. Isso funciona sem novo deploy e sem interromper os containers saudáveis. Deve haver um timeout e circuit breaker no fallback para evitar nova sobrecarga.

7. **Prevenção:**
   - Canary com monitoramento específico da taxa de pedidos concluídos e dos erros do gateway.
   - Thresholds de rollback automáticos para erro 5xx, latência e perda de pedidos.
   - Circuit breaker, fila limitada, retry com backoff e timeout curto.
   - Health check que valide o caminho completo do pagamento, não apenas resposta HTTP.
   - Runbook testado mensalmente e alarmes compostos que correlacionem latência, 5xx e pedidos.

## Parte 2 — Runbook

# Runbook: Degradacao do Gateway de Pagamento

## Trigger

- Alarme em ResponseTime acima da banda dinâmica por mais de 5 minutos.
- Error5xx acima de 5% ou queda superior a 30% em OrdersCreated.
- Logs repetidos de timeout ou circuit breaker aberto no payment-gateway.

## Severidade: P1

Impacto em pedidos pagos ou criados em produção; resposta imediata e comunicação a cada 15 minutos.

## Diagnóstico (< 2 min)

1. Confirmar a degradação:
   `aws cloudwatch get-metric-data --metric-data-queries file://metricas.json --start-time "$(date -u -d '15 minutes ago' +%FT%TZ)" --end-time "$(date -u +%FT%TZ)" --scan-by TimestampDescending`
2. Verificar os logs da API:
   `aws logs filter-log-events --log-group-name /technova/api --start-time $(($(date +%s)-900))000 --filter-pattern "payment-gateway OR timeout OR circuit breaker"`
3. Verificar estado da flag e dos targets:
   `curl -s https://<ALB-DNS>/flags`
   `aws elbv2 describe-target-health --target-group-arn <TG-ARN>`
4. Confirmar se há deploy recente:
   `aws ecr describe-images --repository-name technova-api --query 'imageDetails[*].[imageTags,imagePushedAt]' --output table`

Como confirmar a causa raiz: timeouts concentrados no payment-gateway, aumento simultâneo de latência e erro, queda de pedidos e gateway marcado como indisponível. Confirmar também se o timeout começou logo após ativação da integração V2.

## Ação Imediata

1. Abrir incidente P1 e chamar responsável por pagamentos e on-call.
2. Congelar deploys e execuções automáticas de promoção.
3. Desativar `FF_PAYMENT_GATEWAY_V2` e habilitar o gateway legado ou modo seguro de fallback.
4. Reduzir o tráfego da versão problemática para 0% com o Canary ou voltar o Listener Blue/Green para a versão estável.
5. Limitar ou pausar a fila antes que ela faça overflow; não aceitar novos retries indefinidamente.
6. Validar se as métricas voltam ao baseline em até 5 minutos.

## Ação via Feature Flag

1. Alterar `FF_PAYMENT_GATEWAY_V2=false`.
2. Reiniciar instâncias apenas se o mecanismo de flags exigir reload.
3. Confirmar no endpoint `/flags` que a chave mudou e que o legado está ativo.
4. Executar transação de teste no endpoint de pedidos.
5. Monitorar 5xx, latência e criação de pedidos por pelo menos 10 minutos.

## Verificação

- [ ] ResponseTime voltou para 100-250 ms ou para a banda normal.
- [ ] Error5xx caiu abaixo de 2%.
- [ ] OrdersCreated voltou ao volume esperado.
- [ ] Nenhum target permanece unhealthy.
- [ ] Timeout e circuit breaker não estão sendo registrados novamente.
- [ ] Fila voltou abaixo de 50 itens e não há overflow.
- [ ] Transação de pedido de teste foi concluída.

## Escalação

- Se não houver recuperação em 5 minutos: escalar para o dono do payment-gateway e responsável pela infraestrutura.
- Se houver possível perda ou duplicação de pedidos: escalar P1 para DBA e responsável por dados.
- Se o rollback não resolver: congelar tráfego na rota de pagamentos até contenção, preservando logs, traces e evidências do incidente.
