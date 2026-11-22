# Documentação da API — AkaVision

> RNF01 — Documentação da API (Application Programming Interface)

API REST do backend ([API-6SEM-BACKEND](https://github.com/DenariusData/API-6SEM-BACKEND)), em Django. Todas as rotas começam com `/api/`.

- **URL local:** `http://localhost:8000/api/`
- **Formato:** JSON (`Content-Type: application/json`), exceto upload (`multipart/form-data`), CSV, PDF e PNG, indicados em cada rota
- **Código-fonte das rotas:** `core_api/urls.py`; regras de negócio em `core_api/services/`

## Sumário

1. [Autenticação](#1-autenticação)
2. [Erros](#2-erros)
3. [Paginação](#3-paginação)
4. [Papéis e permissões](#4-papéis-e-permissões)
5. [Rotas](#5-rotas)
   - [Autenticação e conta](#51-autenticação-e-conta)
   - [Dados pessoais (LGPD)](#52-dados-pessoais-lgpd)
   - [Gestão de usuários](#53-gestão-de-usuários)
   - [Opções do formulário de upload](#54-opções-do-formulário-de-upload)
   - [Documentos: envio e consulta](#55-documentos-envio-e-consulta)
   - [Documentos: validação](#56-documentos-validação)
   - [Documentos: visualização segura](#57-documentos-visualização-segura)
   - [Pergunta à IA](#58-pergunta-à-ia)
   - [Auditoria](#59-auditoria)
6. [Resumo das rotas](#6-resumo-das-rotas)

---

## 1. Autenticação

A API usa **JWT** (HS256). O token é obtido em `POST /api/auth/login/` e enviado em todas as outras rotas no cabeçalho:

```
Authorization: Bearer <token>
```

Conteúdo do token:

| Campo | Significado |
|---|---|
| `sub` | uuid do usuário (é também o id do perfil operacional) |
| `email` | e-mail do usuário |
| `role` | papel: `ENGENHARIA`, `QUALIDADE` ou `ADMINISTRADOR` |
| `iat` / `exp` | emissão e expiração. Validade padrão: **60 minutos** (`JWT_EXP_MINUTES`) |

A identidade de quem chama vem **sempre do token**, nunca de parâmetro de URL ou do corpo. Não existe rota que receba o id de outro usuário para ler ou alterar os dados dele.

**Bloqueio de conta:** depois de **5 tentativas** de login erradas, a conta fica bloqueada por **15 minutos** (`423`).

## 2. Erros

Toda resposta de erro em JSON segue o mesmo formato:

```json
{ "detail": "Mensagem para o usuário.", "field": "nome_do_campo" }
```

`field` só aparece quando o erro é de um campo ou parâmetro específico.

| Status | Quando |
|---|---|
| `400` | Corpo ou parâmetro inválido (`field` indica qual) |
| `401` | Sem token, token inválido ou expirado; login errado |
| `403` | Papel sem permissão, ou documento que o usuário não pode ver |
| `404` | Recurso inexistente |
| `405` | Método HTTP não permitido na rota |
| `409` | Conflito: duplicado, e-mail em uso, documento já decidido |
| `415` | Documento que não é PDF (visualização) |
| `423` | Conta bloqueada por excesso de tentativas de login |
| `429` | Limite de pedidos atingido. Vem com o cabeçalho `Retry-After` (segundos) |
| `500` | Falha ao gravar arquivo ou no banco |
| `503` | Serviço de IA indisponível |

**Documento sem permissão (existência mínima, decisão D11).** Quem não pode ver um documento recebe `403` com um corpo fixo, igual para qualquer documento e qualquer motivo, sem título, código, área ou conteúdo:

```json
{ "detail": "Acesso não autorizado.", "codigo": "ACESSO_NAO_AUTORIZADO", "pode_solicitar_acesso": true }
```

Documento que não existe continua `404`.

## 3. Paginação

As listas aceitam `?page=` (começa em 1) e devolvem **20 itens por página**:

```json
{
  "count": 45,
  "page": 1,
  "page_size": 20,
  "total_pages": 3,
  "results": [ ... ]
}
```

Página além da última devolve `results` vazio com os totais certos. Página que não é número inteiro ou menor que 1 devolve `400` com `"field": "page"`.

## 4. Papéis e permissões

| Papel | Pode |
|---|---|
| `ENGENHARIA` | Enviar documentos, ver os próprios envios, consultar e visualizar documentos que a permissão (área + nível de sigilo) libera, perguntar à IA, gerenciar os próprios dados |
| `QUALIDADE` | O mesmo que Engenharia, com as permissões de área e sigilo do seu perfil |
| `ADMINISTRADOR` | Tudo acima, mais: fila de validação, aprovar e devolver documentos (nunca os próprios), gestão de usuários e consulta do log de auditoria |

O acesso a um documento combina **usuário + área + nível de sigilo máximo naquela área** (decisão D9). Pertencer a uma área não libera todos os documentos dela. A mesma regra vale na busca da IA: o modelo nunca recebe trecho de documento fora da permissão do usuário.

---

## 5. Rotas

Legenda da coluna **Acesso:** 🌐 público · 🔑 qualquer usuário logado · 🛡️ só `ADMINISTRADOR`.

### 5.1 Autenticação e conta

#### `POST /api/auth/login/` 🌐

Autentica e devolve o token.

**Corpo**

```json
{ "email": "engenharia@akaer.com.br", "password": "eng123" }
```

**200**

```json
{
  "message": "Autenticação efetuada com sucesso!",
  "token": "<jwt>",
  "user": {
    "id": "b5bdf636-...",
    "name": "Carlos Eduardo",
    "email": "engenharia@akaer.com.br",
    "role": "Engenharia",
    "role_code": "ENGENHARIA",
    "email_pendente": null,
    "pode_validar": false,
    "matricula": "AK-90822",
    "cargo": "Engenheiro Aeroespacial Senior",
    "avatar": "/avatars/default.png",
    "allowed_menus": ["Início", "Documentos", "Meus documentos"],
    "ativo": true,
    "ultimo_login_em": "2026-11-20T00:20:40-03:00"
  }
}
```

**Erros:** `401` e-mail ou senha errados (mensagem única, que não revela se o e-mail existe) · `423` conta bloqueada.

#### `GET /api/auth/me/` 🔑

Perfil do usuário logado (mesmo formato de `user` do login, incluindo `email_pendente`).

**Erros:** `401` · `404` usuário não encontrado.

### 5.2 Dados pessoais (LGPD)

#### `PATCH /api/auth/me/` 🔑

Altera os próprios dados pessoais (tela *Meus dados*, AKA-56). Os dois campos são opcionais; qualquer outra chave do corpo (papel, ativo, id) é ignorada.

```json
{ "nome": "Carlos Eduardo Silva", "email": "novo@akaer.com.br" }
```

Trocar o e-mail **não** muda o e-mail na hora: grava `email_pendente` e envia um link de confirmação. O e-mail só muda em `POST /api/auth/confirmar-email/`.

**200** perfil atualizado · **Erros:** `400` campo inválido ou nada para alterar · `404` · `409` e-mail em uso · `429` mais de 1 link de confirmação por minuto.

#### `POST /api/auth/confirmar-email/` 🌐

Confirma a troca de e-mail com o token do link. Não exige JWT, porque quem clica no link pode não estar logado naquele navegador; o token do link é a credencial (256 bits, uso único, válido por 24 horas).

```json
{ "token": "<token do link>" }
```

**200** `{ "message": "E-mail confirmado." }` · **Erros:** `400` token inválido ou expirado · `409` e-mail ocupado nesse meio tempo.

#### `POST /api/auth/reenviar-confirmacao/` 🔑

Gera um novo link de confirmação (o anterior deixa de valer) e reenvia.

**200** `{ "message": "Link de confirmação reenviado." }` · **Erros:** `400` não há e-mail pendente · `429` mais de 1 pedido por minuto.

#### `GET /api/auth/me/exportar/?formato=json|pdf` 🔑

Baixa a cópia dos dados pessoais do próprio usuário (portabilidade, LGPD art. 18 — AKA-75). Traz dados da conta, envios, histórico de validação e acessos a documentos; **nunca** traz o conteúdo dos documentos nem senha, token ou hash.

**200** arquivo para download (`Content-Disposition: attachment`) · **Erros:** `400` formato inválido · `404` · `429` mais de **1 exportação por minuto** ou mais de **5 no dia** (`Retry-After` até a meia-noite de Brasília — AKA-76).

### 5.3 Gestão de usuários

#### `GET /api/auth/users/` 🛡️

Lista os usuários cadastrados.

**200** `{ "users": [ { ...mesmo formato de user... } ], "count": 6 }`

#### `POST /api/auth/users/` 🛡️

Cadastra um usuário.

```json
{
  "name": "Ana Souza",
  "email": "ana@akaer.com.br",
  "password": "senha-inicial",
  "role": "QUALIDADE",
  "matricula": "AK-77401",
  "cargo": "Inspetora de Qualidade"
}
```

`name` e `email` são obrigatórios. `role` padrão: `ENGENHARIA`.

**201** `{ "message": "Usuário cadastrado com sucesso!", "user": { ... } }` · **Erros:** `400` corpo inválido ou campo obrigatório ausente · `403` não é administrador.

### 5.4 Opções do formulário de upload

Todas 🔑, sem parâmetros (exceto `responsaveis`), e respondem `{ "results": [ ... ] }`.

| Rota | Devolve |
|---|---|
| `GET /api/tipos-documento/` | Tipos de documento (os 9 tipos da decisão D20) |
| `GET /api/areas/` | Áreas, com as categorias e subcategorias de cada uma |
| `GET /api/niveis-sigilo/` | Níveis de sigilo: Público, Interno, Restrito, Confidencial |
| `GET /api/idiomas/` | Idiomas aceitos |
| `GET /api/responsaveis/?area=<id>` | Responsáveis da área, **só id e nome** (sem e-mail, por LGPD). `400` se `area` for inválida |

### 5.5 Documentos: envio e consulta

#### `POST /api/documents/upload/` 🔑

Envia um documento (`multipart/form-data`). O documento nasce **aguardando validação** e só aparece na busca depois de aprovado. Quem enviou vem do token, nunca do formulário.

| # | Campo | Obrigatório | Observação |
|---|---|---|---|
| 1 | `titulo` | sim | até 300 caracteres |
| 2 | `identificador_codigo` | sim | identificador + revisão não podem repetir |
| 3 | `tipo_documento` | sim | id de `/api/tipos-documento/` |
| 4 | `numero_revisao` | sim | |
| 5 | `data_emissao` | sim | `AAAA-MM-DD` |
| 6 | `data_vigencia` | não | não pode ser anterior à emissão |
| 7 | `area_principal` | sim | id de `/api/areas/` |
| | `areas_secundarias` | não | pode repetir o campo |
| 8 | `categoria`, `subcategoria` | sim | ids de `/api/areas/` |
| 9 | `status` | sim | `VIGENTE`, `SUBSTITUIDO`, `CANCELADO`, `EM_REVISAO` |
| 10 | `nivel_sigilo` | sim | id de `/api/niveis-sigilo/` (fica como nível **sugerido**; o validador decide o final) |
| 11 | `origem_fonte` | sim | até 200 caracteres |
| 12 | `responsavel` | sim | id de `/api/responsaveis/` |
| 13 | `idioma` | sim | código de `/api/idiomas/` |
| 14 | `referencia_normativa` | não | texto livre, até 1000 caracteres |
| 15 | `data_validade` | não | não pode ser anterior à emissão |
| 16 | `arquivo` | sim | **só PDF**, até **10 MB** (o conteúdo é conferido, não só a extensão) |

O PDF é gravado com nome gerado numa pasta **sem URL pública**. Documento, áreas e histórico são gravados na mesma transação; se o banco falhar, o arquivo é apagado.

**201** `{ "message": "Documento enviado para validação.", "id": "<uuid>", "status_validacao": "AGUARDANDO_VALIDACAO" }` · **Erros:** `400` primeiro campo inválido (`field`) · `403` sem perfil operacional · `409` identificador + revisão já cadastrados · `500` falha ao gravar.

#### `PUT /api/documents/<id>/reenviar/` 🔑

Reenvia um documento **devolvido**, só pelo próprio autor (`multipart/form-data`, mesmos campos do upload; o arquivo é opcional).

**200** `{ "message": "Documento reenviado para validação.", "id": "<uuid>", "status_validacao": "AGUARDANDO_VALIDACAO" }` · **Erros:** `400` campo inválido · `403` não é o autor · `404` · `409` documento não está devolvido, ou identificador + revisão duplicados · `500`.

#### `GET /api/documents/meus/?status=&page=` 🔑

Envios do próprio usuário, mais recentes primeiro, com `contagens` por status (sem o filtro) para as abas da tela.

| Parâmetro | Valores |
|---|---|
| `status` | `AGUARDANDO_VALIDACAO`, `DEVOLVIDO`, `DISPONIVEL` (opcional) |
| `page` | ver [Paginação](#3-paginação) |

Cada item: `id`, `titulo`, `identificador_codigo`, `tipo_documento`, `numero_revisao`, `nivel_sigilo`, `nivel_sigilo_sugerido`, `status_validacao`, `status_processamento`, `autor`, `criado_em`.

#### `GET /api/documents/pendentes/?page=` 🛡️

Fila de validação: documentos aguardando validação, **sem os enviados pelo próprio administrador** (quem envia não aprova o próprio documento — D12). Mesmo formato de item da rota acima.

#### `GET /api/documents/<id>/` 🔑

Metadados e histórico do documento. Esta rota **não grava** no log de acesso: quem registra a visualização é o `/visualizar/`, uma vez por abertura (correção `fix: AKA-81 Log each document view once`).

**200**

```json
{
  "id": "7c68be48-...",
  "titulo": "Inspeção e calibragem dos pneus do trem de pouso principal",
  "identificador_codigo": "PROC-AKV-310",
  "tipo_documento": "Instrução",
  "numero_revisao": "A",
  "data_emissao": "2026-10-27", "data_vigencia": null, "data_validade": null,
  "status": "VIGENTE",
  "subcategoria": "...", "areas": ["Engenharia"], "idioma": "Português",
  "origem_fonte": "...",
  "nivel_sigilo": "Interno", "nivel_sigilo_sugerido": "Interno",
  "status_validacao": "DISPONIVEL", "status_processamento": "CONCLUIDO",
  "erro_processamento": null, "comentario_devolucao": null,
  "autor": "Carlos Eduardo", "responsavel": "Carlos Eduardo",
  "validado_por": "Ricardo Mendes", "validado_em": "2026-10-27T12:41:56-03:00",
  "criado_em": "...", "atualizado_em": "...",
  "historico": [ { "acao": "ENVIADO", "usuario": "Carlos Eduardo", "comentario": null, "nivel_sigilo": null, "criado_em": "..." } ]
}
```

Pessoas aparecem **só pelo nome**, nunca pelo e-mail. **Erros:** `403` sem permissão (corpo de [existência mínima](#2-erros)) · `404`.

### 5.6 Documentos: validação

#### `POST /api/documents/<id>/aprovar/` 🛡️

Aprova o documento, que passa a `DISPONIVEL` e entra na busca.

```json
{ "nivel_sigilo": "Confidencial", "confirmar_confidencial": true }
```

Os dois campos são opcionais. `nivel_sigilo` (pelo **nome** do nível) troca o nível final; se omitido, vale o atual. O nível sugerido no upload continua guardado. Aprovar como **Confidencial** exige `"confirmar_confidencial": true`.

#### `POST /api/documents/<id>/devolver/` 🛡️

Devolve o documento ao autor para correção.

```json
{ "comentario": "Falta a página de assinaturas." }
```

`comentario` é obrigatório.

**Resposta das duas rotas (200)**

```json
{
  "id": "<uuid>",
  "status_validacao": "DISPONIVEL",
  "nivel_sigilo": "Interno",
  "nivel_sigilo_sugerido": "Interno",
  "validado_por": "<uuid>",
  "validado_em": "2026-11-20T16:48:00-03:00",
  "comentario_devolucao": null
}
```

**Erros:** `400` dados inválidos (ex.: devolver sem comentário, Confidencial sem confirmação) · `403` não é administrador, ou é o autor do documento · `404` · `409` documento já aprovado ou devolvido.

### 5.7 Documentos: visualização segura

O PDF original **nunca sai do servidor**: o navegador recebe só imagens das páginas, com marca d'água.

#### `GET /api/documents/<id>/visualizar/` 🔑

Abre o documento e **registra a visualização no log de acesso** (usuário, documento, data/hora; sem IP — AKA-80).

**200** `{ "total_paginas": 12, "titulo": "..." }`

#### `GET /api/documents/<id>/paginas/<n>/` 🔑

Página `n` como **PNG com marca d'água** (nome e e-mail de quem está vendo, data/hora e código do documento). O nome e o e-mail vêm do token, nunca do front, e a marca é montada na hora, sem ser gravada.

**Erros das duas rotas:** `403` sem permissão (existência mínima) ou sem perfil operacional · `404` documento inexistente ou página fora do intervalo · `415` documento que não é PDF.

As respostas de visualização vêm com cabeçalhos que impedem cache no navegador.

### 5.8 Pergunta à IA

#### `POST /api/perguntas/` 🔑

Faz uma pergunta ao acervo. A busca considera **só os documentos disponíveis que o usuário tem permissão para ver**, e a IA roda localmente (Ollama), sem chamada externa.

```json
{ "pergunta": "Qual a pressão de calibragem dos pneus do trem principal?", "idioma": "pt-BR" }
```

`pergunta`: obrigatória, até 1000 caracteres. `idioma`: `pt-BR` (padrão) ou `en`.

**200**

```json
{
  "pergunta": "...",
  "resposta": "...",
  "encontrou": true,
  "idioma": "pt-BR",
  "fontes": [
    {
      "numero": 1, "documento_id": "<uuid>", "trecho_id": 42,
      "documento": "PROC-AKV-310", "revisao": "A",
      "pagina": "2", "pagina_inicio": 2, "pagina_fim": 2,
      "secao": "3.1", "titulo": "...", "similaridade": 0.812, "citada": true
    }
  ],
  "trechos_extraidos": [
    { "numero": 1, "citado": true, "documento_id": "<uuid>", "documento": "PROC-AKV-310", "revisao": "A", "pagina": "2", "secao": "3.1", "texto": "trecho literal do documento" }
  ],
  "interpretacao_ia": "...",
  "afirmacoes": [ { "texto": "...", "fontes": [1] } ],
  "modelos": { "embedding": "bge-m3", "llm": "llama3.2" },
  "tempos_ms": { "busca": 120, "ia": 3400, "total": 3530 }
}
```

- `trechos_extraidos` traz o texto **literal** do documento; `interpretacao_ia` é o texto gerado pela IA (decisão D3: os dois aparecem separados na tela, com o aviso de que a interpretação não é determinação oficial).
- Cada fonte traz documento, revisão, página e seção (D2).
- Sem base nos documentos: `encontrou: false` e as listas vazias.

**Erros:** `400` pergunta vazia, longa demais ou idioma inválido · `503` serviço de IA indisponível.

### 5.9 Auditoria

#### `GET /api/auditoria/logs/?usuario=&de=&ate=&acao=&page=` 🛡️

Consulta o log de auditoria (AKA-81, RF29): junta o log de auditoria (criação, alteração, validação, login, dados pessoais) e o log de acesso (visualização de documento), do mais recente para o mais antigo. Só leitura.

| Parâmetro | Formato | Observação |
|---|---|---|
| `usuario` | uuid | opcional |
| `de`, `ate` | `AAAA-MM-DD` | opcionais, inclusivos, no **horário de Brasília** |
| `acao` | código | opcional, uma das opções devolvidas em `acoes` |
| `page` | número | ver [Paginação](#3-paginação) |

**200**: paginação + `acoes` (opções do filtro):

```json
{
  "count": 2, "page": 1, "page_size": 20, "total_pages": 1,
  "results": [
    {
      "id": "acesso-17",
      "criado_em": "2026-11-20T03:27:05+00:00",
      "usuario_id": "b5bdf636-...",
      "usuario_nome": "Carlos Eduardo",
      "acao": "VISUALIZACAO", "acao_rotulo": "Visualização",
      "entidade": "DOCUMENTO", "entidade_rotulo": "Documento",
      "registro_id": "7c68be48-..."
    }
  ],
  "acoes": [ { "valor": "LOGIN", "rotulo": "Login" } ]
}
```

O log guarda **só o uuid**; o nome é buscado no banco de credenciais na hora. Usuário excluído aparece como `Usuário removido`; linha sem usuário (login com e-mail inexistente, alteração feita pelo sistema) aparece como `Não identificado`.

**Erros:** `400` usuário que não é uuid, data inválida, data final antes da inicial, ação desconhecida ou página inválida (`field`) · `403` não é administrador.

#### `GET /api/auditoria/logs/exportar/?usuario=&de=&ate=&acao=` 🛡️

Mesmos filtros, **sem paginação**: devolve o resultado inteiro em **CSV** (`text/csv`, UTF-8 com BOM, separado por `;`, data/hora em Brasília), pronto para o Excel em português. Células que começam com `=`, `+`, `-` ou `@` recebem um `'` na frente, para não virarem fórmula (CSV injection).

Colunas: `Data/hora (Brasília)`, `Usuário`, `ID do usuário`, `Ação`, `Entidade`, `Registro`.

---

## 6. Resumo das rotas

| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| POST | `/api/auth/login/` | 🌐 | Login (devolve o JWT) |
| GET | `/api/auth/me/` | 🔑 | Perfil do usuário logado |
| PATCH | `/api/auth/me/` | 🔑 | Alterar nome e e-mail |
| POST | `/api/auth/confirmar-email/` | 🌐 | Confirmar troca de e-mail (token do link) |
| POST | `/api/auth/reenviar-confirmacao/` | 🔑 | Reenviar link de confirmação |
| GET | `/api/auth/me/exportar/` | 🔑 | Exportar os próprios dados (JSON ou PDF) |
| GET | `/api/auth/users/` | 🛡️ | Listar usuários |
| POST | `/api/auth/users/` | 🛡️ | Cadastrar usuário |
| GET | `/api/tipos-documento/` | 🔑 | Opções: tipos de documento |
| GET | `/api/areas/` | 🔑 | Opções: áreas, categorias e subcategorias |
| GET | `/api/niveis-sigilo/` | 🔑 | Opções: níveis de sigilo |
| GET | `/api/idiomas/` | 🔑 | Opções: idiomas |
| GET | `/api/responsaveis/` | 🔑 | Opções: responsáveis da área |
| POST | `/api/documents/upload/` | 🔑 | Enviar documento |
| PUT | `/api/documents/<id>/reenviar/` | 🔑 | Reenviar documento devolvido |
| GET | `/api/documents/meus/` | 🔑 | Meus envios |
| GET | `/api/documents/pendentes/` | 🛡️ | Fila de validação |
| GET | `/api/documents/<id>/` | 🔑 | Detalhe do documento |
| POST | `/api/documents/<id>/aprovar/` | 🛡️ | Aprovar documento |
| POST | `/api/documents/<id>/devolver/` | 🛡️ | Devolver documento |
| GET | `/api/documents/<id>/visualizar/` | 🔑 | Abrir documento (registra visualização) |
| GET | `/api/documents/<id>/paginas/<n>/` | 🔑 | Página como PNG com marca d'água |
| POST | `/api/perguntas/` | 🔑 | Pergunta à IA |
| GET | `/api/auditoria/logs/` | 🛡️ | Consultar o log de auditoria |
| GET | `/api/auditoria/logs/exportar/` | 🛡️ | Exportar o log em CSV |

Além delas, `/admin/` é o painel administrativo do Django (os logs aparecem lá só para leitura).
