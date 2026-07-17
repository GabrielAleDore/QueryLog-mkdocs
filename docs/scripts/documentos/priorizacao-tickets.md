# Priorização de Tickets

## Critério solicitação de prioridade

| Prioridade | Nome      | Impacto                                                                       | Urgencia                                   |
| ---------- | --------- | ----------------------------------------------------------------------------- | ------------------------------------------ |
| P1         | Critica   | Afeta toda a operação do cliente. serviço essencial                           | Necessita ação imediata                    |
| P2         | Alta      | Impacta equipes ou setores inteiros. Há impacto operacional significativo     | Precisa ser resolvido com urgencia         |
| P3         | Media     | Afeta função não essencial. Existem alternativas                              | Pode ser resolvido dentro de alguns dias   |
| P4         | Baixa     | Solicitação de melhoraria, ajuste, duvida, sem impacto operacional ou pratico | Pode ser agendado sem urgencia             |
| P5         | Planejada | Mudança agendada, tarefa prevista, sem impacto atual                          | Sem urgência, faz parte de um planejamento |


## Regras para definição

>Se o ticket não impacta a operação nem tem urgência alta, não pode ser classificado como P1 ou P2. toda prioridade deve ser justificada com base nos critérios definidos.

## Critérios

### Impacto

- **Crítico**: Interrupção total de serviço essencial.
- **Alto**: Equipe ou setor prejudicado, atraso significativo.
- **Médio**: Usuário afetado, mas com solução alternativa.
- **Baixo**: Sem impacto direto no trabalho.
- **Nenhum**: Solicitação futura ou planejamento.

### Urgência

- **Alta**: Sem solução imediata, há prejuízo ou quebra de SLA.
- **Média**: Precisa ser resolvido nos próximos dias.    
- **Baixa**: Pode ser feito conforme disponibilidade.
- **Nenhuma**: Tarefa agendada, sem necessidade imediata.



## Matriz de Eisenhower

|                    | Urgente                                                                                                                                                  | Não Urgente                                                                                                                                               |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Importante**     | **FAZER AGORA**  <br>Prioridade: **P1 ou P2**  <br>🔺Impacto alto/crítico  <br>🔺Urgência alta  <br>🔧 Ex: Sistema parado, prejuízo iminente             | **AGENDAR / PLANEJAR**  <br>Prioridade: **P3**  <br>🟡Impacto médio/alto  <br>🟢Urgência média/baixa  <br>📅 Ex: Requisição importante mas sem emergência |
| **Não Importante** | ELEGAR OU RESOLVER EM SEQUÊNCIA**  <br>Prioridade: **P4**  <br>🟢Impacto baixo  <br>🟡Urgência média/baixa  <br>🛠️ Ex: Problema Simples, dúvida simples | **ADIAR / IGNORAR / AGENDAR NO FUTURO**  <br>Prioridade: **P5**  <br>⚪ Impacto e urgência baixos  <br>🗓️ Ex: Melhorias futuras, mudanças agendadas       |
