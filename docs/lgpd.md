# LGPD no AkaVision

> RNF03 — Anonimização de dados conforme a LGPD · RNF04 — Registro de logs de acesso e manipulação · RNF05 — Registros de descaracterização da base de dados

Este documento reúne **o que o sistema faz hoje** para proteger dados pessoais, com a evidência de cada item (task do Jira e PR). O que ainda não foi feito está listado na [Seção 8](#8-limitações-e-pendências), sem maquiagem.

Documentos relacionados:

- [Inventário de dados pessoais](inventario-dados-pessoais.md) — o que é guardado, para quê, base legal e retenção (art. 37)
- [Documentação da API](api.md) — rotas de dados pessoais, exportação e auditoria
- [Termo de Uso do Portal](Termo_de_Uso_do_Portal.pdf)

## Sumário

1. [Princípios aplicados](#1-princípios-aplicados)
2. [Separação da base de identificação](#2-separação-da-base-de-identificação)
3. [Pseudonimização e minimização (RNF03)](#3-pseudonimização-e-minimização-rnf03)
4. [Direitos do titular](#4-direitos-do-titular)
5. [Controle de acesso](#5-controle-de-acesso)
6. [Logs de acesso e manipulação (RNF04)](#6-logs-de-acesso-e-manipulação-rnf04)
7. [Registro de descaracterização da base (RNF05)](#7-registro-de-descaracterização-da-base-rnf05)
8. [Limitações e pendências](#8-limitações-e-pendências)

---

## 1. Princípios aplicados

| Princípio (LGPD, art. 6º) | Como aparece no sistema |
|---|---|
| **Finalidade e necessidade** (minimização) | Cada dado pessoal guardado tem finalidade registrada no inventário. Logs, listas e telas mostram só o necessário: o log guarda o uuid, as listas mostram o nome, nunca o e-mail |
| **Segurança** | Senha e tokens guardados só como hash; documento visto só como imagem com marca d'água; log que ninguém altera nem apaga |
| **Prevenção** | O filtro de permissão é aplicado **antes** da IA: o modelo nunca recebe trecho de documento que o usuário não pode ver |
| **Transparência e livre acesso** | O titular vê e corrige os próprios dados (*Meus dados*) e baixa uma cópia deles (JSON ou PDF) |
| **Responsabilização** | Toda ação relevante fica registrada no log de auditoria, consultável pelo administrador |

Além disso, **a IA roda localmente** (Ollama, decisão D1): nenhum documento nem pergunta sai para uma API externa.

## 2. Separação da base de identificação

Os dados de identificação ficam num **banco separado** (`credenciais_db`), longe do banco de documentos (`default`):

| Banco | Guarda |
|---|---|
| `credenciais_db` | Usuário: nome, e-mail, hash da senha, papel, dados de login e de confirmação de e-mail |
| `default` | Documentos, trechos e embeddings, permissões, histórico, logs. Pessoas aparecem **só pelo uuid** |

O vínculo entre os dois é só o uuid do usuário (`PerfilOperacional`). Quem tem acesso ao banco de documentos, ou a um backup dele, **não consegue identificar ninguém** sem também ter o banco de credenciais.

*Evidência:* AKA-11 (PR #8 do back), modelagem em [`modelagem-dados.md`](https://github.com/DenariusData/API-6SEM-BACKEND/blob/develop/docs/modelagem-banco/modelagem-dados.md), Seção 1.8.

## 3. Pseudonimização e minimização (RNF03)

O sistema aplica **pseudonimização**: o dado que identifica a pessoa (nome, e-mail) é trocado por um identificador (uuid) em tudo o que não precisa dele, e a ligação entre os dois fica guardada à parte, no banco de credenciais.

| Medida | Onde | Evidência |
|---|---|---|
| Logs guardam só o **uuid** do usuário, nunca nome, e-mail, telefone ou senha | `log_auditoria`, `log_acesso` | AKA-31, AKA-80 (PR #45). Teste automático confere colunas e valores |
| Log de alteração guarda só **os nomes dos campos** alterados, não os valores antigos nem novos | `log_auditoria.campos_alterados` | AKA-31 (PR #25) |
| **IP removido** do log de acesso | coluna `ip_origem` apagada | AKA-80 (PR #45, migration `0010_log_imutavel`) |
| Nome exibido **na hora**, buscado no banco de credenciais; usuário excluído aparece como "Usuário removido" | tela de Auditoria | AKA-81 (PR #56) |
| Listas e detalhes de documentos mostram pessoas **só pelo nome**, sem e-mail | `GET /api/documents/...` | S2-35 |
| Lista de responsáveis devolve **só id e nome** | `GET /api/responsaveis/` | S2-32 |
| Senha guardada só como **hash** (PBKDF2, padrão do Django) | `usuario.senha_hash` | AKA-11 |
| Token de confirmação de e-mail guardado só como **hash SHA-256**, de uso único, válido por 24 horas | `usuario.token_confirmacao_hash` | AKA-57 (PR #32) |
| Marca d'água com nome e e-mail montada **na hora** de gerar a imagem, **sem ser gravada** | visualizador | AKA-48 (PR #27) |
| O PDF original **nunca sai do servidor**: o navegador recebe só imagens das páginas | visualizador | AKA-48 (PR #27) |

**Sobre o termo "anonimização".** Pela LGPD (art. 5º, III e XI), dado anonimizado é o que não pode mais ser ligado à pessoa por nenhum meio razoável. O que o sistema faz é **pseudonimização**: a ligação existe, mas fica isolada no banco de credenciais. A anonimização propriamente dita (por exemplo, ao excluir uma conta) está na [Seção 8](#8-limitações-e-pendências).

## 4. Direitos do titular

| Direito (art. 18) | Como o usuário exerce | Evidência |
|---|---|---|
| Confirmação e acesso | Tela *Meus dados* (`GET /api/auth/me/`) | AKA-56 |
| Correção | Edita nome e e-mail em *Meus dados*. A troca de e-mail só vale depois de confirmada pelo link enviado ao novo endereço | AKA-56, AKA-57 (PR #32) |
| Portabilidade | Botão *Baixar meus dados*, em **JSON ou PDF**, com dados da conta, envios, histórico de validação e acessos a documentos | AKA-75 (PR #43) |
| — limite de abuso | Até 1 exportação por minuto e 5 por dia | AKA-76 (PR #54) |
| — sem vazar documentos | A exportação **nunca** leva o conteúdo dos documentos, nem senha, hash ou token | AKA-77 (PR #53) |
| Eliminação | **Não implementado** — ver [Seção 8](#8-limitações-e-pendências) | — |

A identidade vem sempre do token: **não há como pedir os dados de outra pessoa**.

## 5. Controle de acesso

- **Permissão = usuário + área + nível de sigilo máximo** naquela área (decisão D9). Pertencer a uma área não libera todos os documentos dela. *Evidência:* AKA-26 (PR #48).
- **Filtro antes da IA:** a busca só recupera trechos de documentos disponíveis que o usuário pode ver; o modelo nunca recebe trecho de fora da permissão. Validado com dois usuários fazendo a mesma pergunta. *Evidência:* AKA-26 (PR #48), AKA-66 (PR #55, relatório em `docs/validacoes/` do back).
- **Existência mínima (D11):** quem não tem permissão recebe só "Acesso não autorizado", sem título, código, área ou trecho. *Evidência:* AKA-27 (PR #49).
- **Quem envia não aprova o próprio documento (D12)**, e só o administrador valida.

## 6. Logs de acesso e manipulação (RNF04)

| Log | O que registra | Onde |
|---|---|---|
| `log_auditoria` | Criação, alteração e exclusão de documentos, fluxo de validação (envio, aprovação, devolução), login e login falho, exportação de dados pessoais | Banco `default` |
| `log_acesso` | Visualização de documento | Banco `default` |

Cada linha guarda **quem** (uuid), **o quê** (ação), **sobre o quê** (entidade e id do registro) e **quando** (data/hora). Nenhuma guarda dado pessoal.

- **Só de inclusão:** um trigger no banco recusa `UPDATE` e `DELETE` nas duas tabelas, venha do sistema, do painel administrativo ou de SQL direto. Excluir um documento ou um usuário **não** apaga o log. *Evidência:* AKA-80 (PR #45), com testes automáticos para cada caso.
- **Consulta:** o administrador consulta e filtra o log (usuário, período, ação) e exporta o resultado em CSV. *Evidência:* AKA-81 (PR #56 do back e #44 do front).

## 7. Registro de descaracterização da base (RNF05)

Registro dos procedimentos que **retiraram ou impediram** dados pessoais na base, em ordem de data. Cada linha tem a evidência (PR e, quando houve, migration) que permite conferir o que foi feito.

| Data | Base / tabela | Dado | Procedimento | Motivo | Evidência |
|---|---|---|---|---|---|
| 22/09/2026 | `credenciais_db` | Identificação do usuário | Dados de identificação separados num banco próprio; o banco de documentos referencia a pessoa só pelo uuid | Isolar o dado que identifica a pessoa (pseudonimização) | AKA-11, PR #8 (back) |
| 22/09/2026 | `credenciais_db.usuario` | Senha | Guardada só como hash (PBKDF2); a senha em texto nunca é gravada | Segurança da credencial | AKA-11, PR #8 (back) |
| 21/10/2026 | visualizador | Nome e e-mail na marca d'água; PDF original | Marca d'água gerada na hora, sem gravar; o PDF nunca é enviado ao navegador | Rastrear vazamento sem guardar dado novo | AKA-48, PR #27 (back) |
| 25/10/2026 | `default.log_auditoria` | Valores alterados | O log de alteração guarda só os **nomes** dos campos, nunca os valores | Não copiar dado pessoal para o log | AKA-31, PR #25 (back) |
| 25/10/2026 | `credenciais_db.usuario` | Token de confirmação de e-mail | Guardado só como hash SHA-256, de uso único, com validade de 24 h | Não guardar credencial reaproveitável | AKA-57, PR #32 (back) |
| 11/11/2026 | exportação de dados pessoais | Hash da senha, tokens | Excluídos do arquivo exportado, que só explica que eles existem | O titular recebe seus dados, não credenciais | AKA-75, PR #43 (back) |
| 13/11/2026 | `default.log_acesso` | **IP de origem** | Coluna `ip_origem` **removida** (e com ela os IPs já gravados); a visualização deixou de gravar e a exportação deixou de mostrar o IP | IP é dado pessoal e o RF21 pede auditoria sem dado pessoal (decisão de 11/11) | AKA-80, PR #45 (back), migration `0010_log_imutavel` |
| 13/11/2026 | `default.log_acesso`, `default.log_auditoria` | Ligação com o usuário | O log deixou de ser apagado em cascata com o usuário ou o documento; fica só o uuid | Manter a trilha de auditoria sem dado pessoal | AKA-80, PR #45 (back) |
| 18/11/2026 | exportação de dados pessoais | Conteúdo de documentos | Confirmado por teste que a exportação não leva nenhum documento | Não vazar documento de terceiros pela portabilidade | AKA-77, PR #53 (back) |

Os usuários de demonstração (`seed_usuarios_demo`) são **fictícios**: nenhum dado pessoal real é usado em desenvolvimento, testes ou apresentação.

## 8. Limitações e pendências

O que o projeto **não** entrega hoje, para que fique claro na avaliação:

| Pendência | Situação |
|---|---|
| **Eliminação / anonimização da conta** (art. 18, VI) | Não há rota para excluir ou anonimizar um usuário. Como o log guarda só o uuid e não apaga em cascata, a exclusão ficaria preparada: bastaria apagar o usuário no banco de credenciais para o log passar a mostrar "Usuário removido" |
| **Registro do aceite do Termo de Uso** | O termo existe (tela no front e [PDF](Termo_de_Uso_do_Portal.pdf)), mas o aceite (usuário, versão, data) ainda não é gravado no back |
| **Prazos de retenção** | Vários itens do inventário estão "a definir"; não há rotina automática de descarte |
| **Texto das perguntas à IA** | `registro_pergunta` guarda a pergunta como foi digitada, e ela pode conter dado pessoal; não há anonimização desse texto |
| **Conteúdo dos documentos** | Documentos técnicos podem citar pessoas (autores, aprovadores); o conteúdo não é anonimizado, só protegido por permissão |
| **Notificação de incidente** (art. 48) | Não há rotina de comunicação de vazamento implementada |
