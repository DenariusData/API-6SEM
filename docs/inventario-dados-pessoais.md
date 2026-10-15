# Inventário de Dados Pessoais 

> Registro das operações de tratamento de dados pessoais do sistema AKAVISION.

## Origem dos dados de identificação

Os dados de identificação do usuário (**e-mail, senha, nome, cargo e gestor**) são os fornecidos/validados pelo **Sistema da Akavison**. O sistema usa esses dados para autenticar e para definir o papel e o fluxo de aprovação do usuário.

## Tabela do inventário

| # | Dado pessoal | Titular | Finalidade | Quem acessa |
|---|---|---|---|---|
| 1 | Nome | Usuário | Identificar o usuário no sistema, nos logs e no fluxo de validação | Administrador, o próprio usuário |
| 2 | E-mail | Usuário | Login, comunicação e confirmação de e-mail | Administrador, o próprio usuário |
| 3 | Senha (hash) | Usuário | Autenticar o acesso ao sistema | Somente o serviço de autenticação |
| 4 | Cargo | Usuário | Definir papel/permissões e contexto de aprovação | Administrador, serviço de autorização |
| 5 | Gestor | Usuário | Definir quem aprova/valida os documentos do usuário | Administrador, fluxo de validação |
| 6 | Último login, tentativas de login, bloqueado até, ativo | Usuário | Segurança: proteção contra força bruta e controle de conta | Administrador, serviço de autenticação |
| 7 | E-mail pendente e token de confirmação (AKA-57, Sprint 2) | Usuário | Confirmar a troca de e-mail | Serviço de autenticação |
| 8 | ID do usuário (PerfilOperacional) | Usuário | Vincular o perfil operacional ao usuário | Serviços internos |
| 9 | Log de acesso: usuário, ação, documento, data e IP de origem | Usuário | Auditoria, rastreabilidade e segurança | Administrador/auditoria |
| 10 | Texto da pergunta (RegistroPergunta) | Usuário | Histórico de uso, melhoria e auditoria das respostas da IA | Administrador; o próprio usuário |
| 11 | Aceite do Termo de Uso: usuário, versão, data | Usuário | Comprovar o aceite dos termos | Administrador/jurídico |
| 12 | Fluxo de validação: `validado_por` e comentário de devolução (texto livre) | Usuário / validador | Registrar quem validou/devolveu documentos e por quê | Usuário envolvido, gestor, administrador |
| 13 | Conteúdo dos documentos (pode citar pessoas: autores, aprovadores) | Terceiros citados nos documentos | Gestão documental e busca com IA (RAG) | Usuários com permissão ao documento |
| 14 | Marca d'água (nome e e-mail) | Usuário | Rastrear vazamento de documentos | Quem visualiza o documento |
