# Checklist de DoR e DoD

Acordo entre o time e a Product Owner sobre quando uma tarefa **pode começar** (DoR) e quando ela está **pronta** (DoD). Vale para as três Sprints; a documentação de cada Sprint está em [`sprints/`](sprints/).

## DoR — Definition of Ready

Antes de uma tarefa entrar na Sprint e ser atribuída a alguém:

**Sobre a história e a tarefa**

- [ ] Título claro e descrição bem definida (no Jira)
- [ ] User story no formato "Como [papel], quero [ação], para [benefício]", com **critérios de aceite** escritos e testáveis
- [ ] Decisões de produto (regras de negócio) aplicáveis identificadas e citadas (ex.: D9, D11)
- [ ] Requisito funcional ou não funcional ligado à tarefa (label `RFxx` / `RNFxx` no Jira)
- [ ] Estimada pelo time, **em horas**, com duração prevista **menor que 8 horas** por tarefa
- [ ] Sem dependência bloqueadora conhecida
- [ ] Entendimento validado com o time na Planning

**Sobre os artefatos**

- [ ] Wireframe anexado, quando a tarefa tem tela
- [ ] Modelo de dados disponível, quando a tarefa mexe no banco ([modelagem](https://github.com/DenariusData/API-6SEM-BACKEND/blob/develop/docs/modelagem-banco/modelagem-dados.md))
- [ ] Estratégia de teste definida (unitário, integração ou validação manual)

## DoD — Definition of Done

Para a tarefa ser movida para **Concluído**:

**Código e processo**

- [ ] Branch criada a partir de `develop` no padrão `<tipo>/AKA-<número>-descricao` ([estratégia de branch](estrategia-branch.md))
- [ ] Commits no padrão `<tipo>: AKA-<número> <descrição>`
- [ ] Pull Request aberto para `develop`, com o template preenchido (descrição, tarefa, como testar, checklist)
- [ ] **Pelo menos 1 aprovação** de outra pessoa (quem abre o PR não aprova o próprio)
- [ ] CI verde: lint, testes unitários, testes de integração e build
- [ ] Análise do SonarQube Cloud sem reprovação no Quality Gate
- [ ] Sem conflitos com `develop`; merge por **Squash and merge** e branch apagada

**Qualidade e aceite**

- [ ] Critérios de aceite da tarefa conferidos (pelo revisor e validados pela PO)
- [ ] Testes automatizados cobrindo os critérios de aceite, quando a tarefa tem código
- [ ] Decisões de produto aplicáveis verificadas
- [ ] Sem bug crítico conhecido aberto
- [ ] Documentação atualizada no Git, quando aplicável (API, modelagem, LGPD, manuais)
