# Inventário de Dados Pessoais (ROPA — art. 37 da LGPD)

> Registro das operações de tratamento de dados pessoais do sistema AkaVision.

## Origem dos dados de identificação

Os dados de identificação do usuário (**e-mail, senha, nome, cargo e gestor**) são os fornecidos/validados pelo **Sistema da AkaVision**. O sistema usa esses dados para autenticar e para definir o papel e o fluxo de aprovação do usuário.

## Tabela do inventário

| # | Dado pessoal | Titular | Finalidade | Base legal (art. 7º) | Quem acessa | Retenção e descarte | Compartilhamento |
|---|---|---|---|---|---|---|---|
| 1 | Nome | Usuário | Identificar o usuário no sistema, nos logs e no fluxo de validação | Execução de contrato (art. 7º, V) | Administrador, o próprio usuário | Enquanto o vínculo estiver ativo; depois, a definir. Descarte: exclusão ou anonimização | Não compartilhado |
| 2 | E-mail | Usuário | Login, comunicação e confirmação de e-mail | Execução de contrato (art. 7º, V) | Administrador, o próprio usuário | Enquanto o vínculo estiver ativo; depois, a definir. Descarte: exclusão ou anonimização | Serviço de e-mail (envio do link de confirmação) |
| 3 | Senha (hash) | Usuário | Autenticar o acesso ao sistema | Execução de contrato (art. 7º, V) | Somente o serviço de autenticação | Enquanto a conta existir. Descarte: exclusão do hash junto com a conta | Não compartilhado |
| 4 | Cargo | Usuário | Definir papel/permissões e contexto de aprovação | Execução de contrato (art. 7º, V) | Administrador, serviço de autorização | Enquanto o vínculo estiver ativo; depois, a definir. Descarte: exclusão junto com a conta (se só vem da API, não é armazenado) | Não compartilhado |
| 5 | Gestor | Usuário | Definir quem aprova/valida os documentos do usuário | Execução de contrato (art. 7º, V) | Administrador, fluxo de validação | Enquanto o vínculo estiver ativo; depois, a definir. Descarte: exclusão junto com a conta (se só vem da API, não é armazenado) | Não compartilhado |
| 6 | Último login, tentativas de login, bloqueado até, ativo | Usuário | Segurança: proteção contra força bruta e controle de conta | Legítimo interesse (art. 7º, IX) | Administrador, serviço de autenticação | Enquanto a conta existir. Descarte: exclusão junto com a conta | Não compartilhado |
| 7 | E-mail pendente e token de confirmação (AKA-57, Sprint 2) | Usuário | Confirmar a troca de e-mail | Execução de contrato (art. 7º, V) | Serviço de autenticação | Até confirmar ou expirar o token (validade a definir). Descarte: limpeza do campo | Serviço de e-mail (envio do link com o token) |
| 8 | ID do usuário (PerfilOperacional) | Usuário | Vincular o perfil operacional ao usuário | Execução de contrato (art. 7º, V) | Serviços internos | Enquanto a conta existir. Descarte: exclusão em cascata com o usuário | Não compartilhado |
| 9 | Log de acesso: usuário, ação, documento, data e IP de origem | Usuário | Auditoria, rastreabilidade e segurança | Legítimo interesse (art. 7º, IX) | Administrador/auditoria | A definir. Descarte: exclusão periódica por rotina agendada | Não compartilhado |
| 10 | Texto da pergunta (RegistroPergunta). **Obs.:** a pergunta pode conter dados pessoais digitados pelo usuário | Usuário | Histórico de uso, melhoria e auditoria das respostas da IA | Legítimo interesse (art. 7º, IX) | Administrador; o próprio usuário | A definir. Descarte: exclusão periódica | Modelo de IA local, sem chamada externa |
| 11 | Aceite do Termo de Uso: usuário, versão, data | Usuário | Comprovar o aceite dos termos | Cumprimento de obrigação legal e exercício regular de direitos (art. 7º, II e VI) | Administrador/jurídico | Enquanto durar o vínculo; depois, a definir (prazo prescricional). Descarte: exclusão ou anonimização | Não compartilhado |
| 12 | Fluxo de validação: `validado_por` e comentário de devolução (texto livre) | Usuário / validador | Registrar quem validou/devolveu documentos e por quê | Execução de contrato (art. 7º, V) | Usuário envolvido, gestor, administrador | Enquanto o documento existir. Descarte: exclusão junto com o documento | Não compartilhado |
| 13 | Conteúdo dos documentos (pode citar pessoas: autores, aprovadores) | Terceiros citados nos documentos | Gestão documental e busca com IA (RAG) | Legítimo interesse (art. 7º, IX) | Usuários com permissão ao documento | Enquanto o documento estiver ativo. Descarte: exclusão do arquivo, dos trechos e dos embeddings | Modelo de IA local (embeddings e respostas), sem chamada externa |
| 14 | Marca d'água (nome e e-mail) | Usuário | Rastrear vazamento de documentos | Legítimo interesse (art. 7º, IX) | Quem visualiza o documento | Não há retenção (gerada na hora, não é gravada). Descarte: não se aplica | Não compartilhado |
