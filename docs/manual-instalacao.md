# Manual de Instalação — AkaVision

Passo a passo para rodar o AkaVision na própria máquina (backend, frontend, bancos e IA local). Os comandos estão para **Windows (PowerShell)**; no Linux ou Mac, as diferenças aparecem em comentário.

Tempo estimado: **30 a 40 minutos** na primeira vez (a maior parte é download dos modelos de IA).

## Sumário

1. [Pré-requisitos](#1-pré-requisitos)
2. [Clonar o projeto](#2-clonar-o-projeto)
3. [Backend](#3-backend)
4. [IA local (Ollama)](#4-ia-local-ollama)
5. [Frontend](#5-frontend)
6. [Primeiro acesso](#6-primeiro-acesso)
7. [Rodar os testes](#7-rodar-os-testes)
8. [Parar tudo](#8-parar-tudo)
9. [Problemas comuns](#9-problemas-comuns)

---

## 1. Pré-requisitos

| Ferramenta | Versão | Para quê |
|---|---|---|
| [Git](https://git-scm.com/) | qualquer recente | Clonar o projeto |
| [Python](https://www.python.org/downloads/) | **3.12** (3.10 ou mais recente funciona) | Backend. No Windows, marque **"Add python.exe to PATH"** na instalação |
| [Node.js](https://nodejs.org/) | **22** | Frontend |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | qualquer recente | Os dois bancos PostgreSQL. Precisa da virtualização ativada na BIOS e do WSL 2 |
| [Ollama](https://ollama.com/download) | qualquer recente | Modelos de IA locais (embeddings e respostas) |

**Recomendação:** não coloque o projeto dentro de uma pasta sincronizada pelo OneDrive, Google Drive ou Dropbox. A sincronização trava arquivos da pasta `.git` e atrapalha comandos como `git checkout` e `git am`. Prefira, por exemplo, `C:\dev\`.

## 2. Clonar o projeto

O repositório principal tem o backend e o frontend como submódulos:

```powershell
git clone --recurse-submodules https://github.com/DenariusData/API-6SEM.git
cd API-6SEM
```

## 3. Backend

```powershell
cd API-6SEM-BACKEND
git checkout develop
```

### 3.1 Ambiente virtual e dependências

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1          # Linux/Mac: source venv/bin/activate
pip install -r requirements-dev.txt  # inclui o requirements.txt e as ferramentas de teste
```

> Se o PowerShell bloquear o `Activate.ps1`, rode antes: `Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned`

### 3.2 Variáveis de ambiente

Crie o `.env` a partir do exemplo:

```powershell
copy .env.example .env               # Linux/Mac: cp .env.example .env
```

Abra o `.env` e preencha, no mínimo, estas linhas (os valores dos bancos já batem com o `docker-compose.yml`):

```ini
DEBUG=True
SECRET_KEY=troque-por-uma-chave-qualquer-local
JWT_SECRET=troque-por-outra-chave-qualquer-local
CORS_ORIGINS=http://localhost:5173
FRONTEND_URL=http://localhost:5173

# Banco de documentos (pgvector)
DEFAULT_DB_ENGINE=django.db.backends.postgresql
DEFAULT_DB_NAME=rag
DEFAULT_DB_USER=rag
DEFAULT_DB_PASSWORD=rag
DEFAULT_DB_HOST=localhost
DEFAULT_DB_PORT=5434
DATABASE_URL=postgresql://rag:rag@localhost:5434/rag

# Banco de credenciais (separado, por LGPD)
CREDENCIAIS_DB_NAME=credenciais
CREDENCIAIS_DB_USER=credenciais
CREDENCIAIS_DB_PASSWORD=credenciais
CREDENCIAIS_DB_HOST=localhost
CREDENCIAIS_DB_PORT=5433

# IA local
OLLAMA_URL=http://localhost:11434
RAG_MODELO_LLM=llama3.2
RAG_MODELO_EMBEDDING=bge-m3
```

> Sem as variáveis `DEFAULT_DB_*`, o backend cai num SQLite local, e a busca com IA não funciona (ela precisa do pgvector).

Envio de e-mail (confirmação de troca de e-mail) é opcional: sem as variáveis `EMAIL_*`, o link aparece no terminal do backend.

### 3.3 Bancos de dados

Com o **Docker Desktop aberto** (espere aparecer "Engine running"):

```powershell
docker compose up -d
docker compose ps
```

Os dois containers, `rag-postgres` (porta 5434) e `credenciais-postgres` (porta 5433), precisam estar `running`.

### 3.4 Tabelas e dados iniciais

```powershell
python manage.py migrate
python manage.py migrate --database credenciais_db
python manage.py seed_taxonomia
python manage.py seed_usuarios_demo
```

- `migrate`: cria as tabelas de cada banco (são dois comandos porque são dois bancos).
- `seed_taxonomia`: carrega áreas, categorias, subcategorias, tipos de documento, níveis de sigilo e idiomas.
- `seed_usuarios_demo`: cria os três usuários de demonstração (ver [Primeiro acesso](#6-primeiro-acesso)) e as permissões de área de cada um. São **dados fictícios**.

Os comandos podem ser rodados de novo sem problema: não duplicam nada.

### 3.5 Subir o backend

```powershell
python manage.py runserver
```

O backend fica em `http://localhost:8000`. Deixe esse terminal aberto.

## 4. IA local (Ollama)

Com o Ollama instalado e aberto, baixe os dois modelos (uma vez só; são alguns GB):

```powershell
ollama pull bge-m3
ollama pull llama3.2
```

- `bge-m3`: gera os embeddings (vetores) dos trechos dos documentos e das perguntas.
- `llama3.2`: escreve as respostas.

Quando um documento é **aprovado**, o backend o processa sozinho para a IA (extrai o texto, divide em trechos e gera os embeddings). Para carregar também a base pública de normas do EverySpec, siga o [passo a passo do pipeline RAG](https://github.com/DenariusData/API-6SEM-BACKEND/blob/develop/docs/pipeline-rag.md). É opcional.

## 5. Frontend

Em **outro terminal**, a partir da pasta `API-6SEM`:

```powershell
cd API-6SEM-FRONTEND
git checkout develop
npm install
copy .env.example .env               # Linux/Mac: cp .env.example .env
```

No `.env` do front, deixe:

```ini
VITE_API_BASE_URL=http://localhost:8000
```

Suba o front:

```powershell
npm run dev
```

Abra o endereço que aparecer no terminal (normalmente `http://localhost:5173`).

## 6. Primeiro acesso

Usuários criados pelo `seed_usuarios_demo`:

| Perfil | E-mail | Senha | Vê |
|---|---|---|---|
| Administrador | `admin@akaer.com.br` | `admin123` | Tudo, mais Validação, Gestão de Usuários e Auditoria |
| Engenharia | `engenharia@akaer.com.br` | `eng123` | Documentos das áreas e níveis de sigilo do perfil de Engenharia |
| Qualidade | `qualidade@akaer.com.br` | `qual123` | Documentos das áreas e níveis de sigilo do perfil de Qualidade |

Como usar cada tela: [Manual de Usuário](manual-usuario.md).

## 7. Rodar os testes

**Backend** (com o venv ativo e os bancos no ar):

```powershell
pytest tests              # suíte completa (integração + unitários), cerca de 5 a 7 minutos
pytest tests/unit         # só os unitários, rápido
```

Os testes criam um banco próprio e não mexem nos seus dados.

**Frontend:**

```powershell
npm run test:unit
npm run test:integration
npm run lint
npm run build
```

**Hooks de pre-commit** (validam nome da branch, mensagem de commit e lint antes de cada commit), uma vez por clone, em cada repositório:

```powershell
pip install pre-commit
pre-commit install --hook-type pre-commit --hook-type commit-msg
```

## 8. Parar tudo

1. No terminal do backend: **Ctrl + C**, depois `docker compose stop` (os dados continuam guardados) e `deactivate`.
2. No terminal do frontend: **Ctrl + C** (se perguntar "Deseja finalizar o arquivo em lotes?", responda **S**).

Para voltar depois: abra o Docker Desktop, ative o venv, rode `docker compose up -d` e `python manage.py runserver` no back, e `npm run dev` no front.

## 9. Problemas comuns

| Sintoma | Causa e solução |
|---|---|
| `ModuleNotFoundError: No module named 'django'` ou `pytest` não reconhecido | O venv não está ativo. Rode `.\venv\Scripts\Activate.ps1` |
| `failed to connect to the docker API ... dockerDesktopLinuxEngine` | O Docker Desktop está fechado. Abra e espere "Engine running" |
| Login sempre dá "e-mail ou senha inválidos" com os usuários de demonstração | Faltou `python manage.py migrate --database credenciais_db` ou `python manage.py seed_usuarios_demo` |
| Pergunta à IA responde "serviço de IA indisponível" (503) | O Ollama está fechado ou os modelos não foram baixados (ver [Seção 4](#4-ia-local-ollama)) |
| O front abre, mas nada carrega (erro de CORS no console) | `CORS_ORIGINS` no `.env` do back precisa ter o endereço do front, por exemplo `http://localhost:5173` |
| `Deletion of directory ... failed. Should I try again?` no Git | Pasta sincronizada (OneDrive) ou VS Code segurando arquivos. Pause a sincronização e feche o editor |
| `&&` dá erro no PowerShell | O PowerShell 5 não aceita `&&`. Rode um comando por linha ou separe com `;` |
