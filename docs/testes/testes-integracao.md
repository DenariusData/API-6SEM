# Testes de integração

Guia para rodar, entender e escrever os testes de integração do **AkaVision** (backend e frontend).

> Testes unitários ficam em outro guia: [testes-unitarios.md](testes-unitarios.md).

## O que é um teste de integração aqui

Um teste de integração verifica **várias partes funcionando juntas**, do jeito que o sistema roda de verdade:

- **Backend**: requisição HTTP → URL → view → decorator de JWT → serviço → ORM → **PostgreSQL real** (com migrations, triggers e pgvector) → resposta JSON.
- **Frontend**: tela Vue → `fetch` real → **requisição HTTP de verdade** → resposta da API → estado da tela e `localStorage`.

## Unitário x integração

| | Unitário (`tests/unit`) | Integração (`tests/integration`) |
|---|---|---|
| Testa | uma função ou componente | um fluxo completo entre camadas |
| Banco | não usa | PostgreSQL real (backend) |
| Rede | `fetch`/`requests` trocados por mock | HTTP de verdade (MSW responde no frontend) |
| Ollama (IA) | mock | mock (não depende de modelo no CI) |
| Precisa de Docker | não | sim, no backend |
| Tempo | < 1 s | alguns segundos |
| Quando falha | a regra de negócio quebrou | as partes não conversam mais (rota, schema, contrato da API) |

## Ferramentas

| Repositório | Ferramenta | Para que serve |
|---|---|---|
| Backend | pytest + pytest-django | Cria os bancos de teste, aplica migrations e isola cada teste |
| Backend | `django.test.Client` | Faz requisições HTTP para as views sem subir servidor |
| Backend | PostgreSQL 17 + pgvector (Docker) | Mesmos bancos do `docker-compose.yml` |
| Frontend | Vitest (projeto `integration`) | Executor de testes |
| Frontend | [MSW](https://mswjs.io) 3 (Mock Service Worker) | Intercepta as requisições HTTP e responde como o backend |
| Frontend | [jsdom](https://github.com/jsdom/jsdom) | DOM simulado que usa o `fetch` nativo do Node (o MSW intercepta esse `fetch`) |

## Estrutura

```text
API-6SEM-BACKEND/
├── docker-compose.yml           # postgres (5432, pgvector) e credenciais-postgres (5433)
└── tests/
    └── integration/
        ├── test_auth.py         # login, perfil, gestão de usuários
        ├── test_upload.py       # upload de documento
        ├── test_pergunta.py     # endpoint de perguntas + registro no banco
        ├── test_documento.py    # constraints e triggers do banco de negócio
        └── test_validacao.py    # estados de validação, histórico e migrations

API-6SEM-FRONTEND/
├── vite.config.js               # projetos "unit" (happy-dom) e "integration" (jsdom + MSW)
└── tests/
    └── integration/
        ├── server.js            # servidor MSW
        ├── setup.js             # liga/desliga o MSW e limpa o estado entre testes
        ├── login.spec.js        # App.vue: login → termos → painel
        └── pergunta.spec.js     # DashboardView: pergunta à IA com fontes
```

## Como rodar

### Backend

Pré-requisitos: Python 3.12 e Docker.

```bash
cd API-6SEM-BACKEND
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt

docker compose up -d               # sobe os dois bancos
```

O banco de negócio precisa ser o PostgreSQL (as migrations usam triggers e a extensão `vector`). Coloque no `.env` da raiz do backend:

```bash
DEFAULT_DB_ENGINE=django.db.backends.postgresql
DEFAULT_DB_NAME=rag
DEFAULT_DB_USER=rag
DEFAULT_DB_PASSWORD=rag
DEFAULT_DB_HOST=localhost
DEFAULT_DB_PORT=5432
```

O banco de credenciais já usa os valores padrão do `docker-compose.yml` (porta 5433). Então:

```bash
pytest tests/integration
```

Saída esperada:

```text
67 passed in 10.98s
```

O pytest-django cria `test_rag` e `test_credenciais` ao lado dos bancos de desenvolvimento, aplica todas as migrations e apaga os dois no final. Seus dados de desenvolvimento não são tocados.

Comandos úteis:

```bash
pytest tests/integration -v                         # nome de cada teste
pytest tests/integration/test_auth.py               # um arquivo
pytest tests/integration --reuse-db                 # reaproveita os bancos de teste (mais rápido)
pytest tests/integration --create-db                # recria os bancos depois de mudar uma migration
pytest tests --cov --cov-report=term-missing        # unitários + integração com cobertura
docker compose down                                 # desliga os bancos
```

### Frontend

Pré-requisito: Node 22 ou mais novo. O backend **não** precisa estar rodando: o MSW faz o papel dele.

```bash
cd API-6SEM-FRONTEND
npm install
npm run test:integration
```

Saída esperada:

```text
 Test Files  2 passed (2)
      Tests  5 passed (5)
```

Comandos úteis:

```bash
npx vitest --project integration                       # modo watch
npm run test:integration -- --reporter=verbose         # nome de cada teste
npx vitest run --project integration -t "login"        # só testes com "login" no nome
```

## Como funciona no backend

### Dois bancos no mesmo teste

Trecho simplificado de `tests/integration/test_auth.py`. O projeto separa dados pessoais (`credenciais_db`) do banco de negócio (`default`). Por padrão o Django só libera o `default` dentro de um teste, então o teste declara os dois:

```python
from django.test import Client, TestCase


class AuthTestCase(TestCase):
    databases = {"default", "credenciais_db"}

    @classmethod
    def setUpTestData(cls):
        AuthService.create_user(
            {"email": "admin@akaer.com.br", "password": "admin123", "role": Papel.ADMINISTRADOR}
        )

    def test_user_management_list_with_admin_token(self):
        login = self.client.post(
            "/api/auth/login/",
            data=json.dumps({"email": "admin@akaer.com.br", "password": "admin123"}),
            content_type="application/json",
        )
        token = login.json()["token"]

        response = self.client.get(
            "/api/auth/users/", HTTP_AUTHORIZATION=f"Bearer {token}"
        )

        self.assertEqual(response.status_code, 200)
```

Esse único teste passa por: rota, view de login, hash de senha, JWT, decorator `require_role`, roteador de banco e os dois PostgreSQL.

### `TestCase` x `TransactionTestCase`

| Classe | Como isola | Use quando |
|---|---|---|
| `TestCase` | Cada teste roda dentro de uma transação desfeita no final (rápido) | Testes de endpoint e de regra comum |
| `TransactionTestCase` | Limpa as tabelas depois de cada teste (mais lento) | Testar triggers, constraints que só disparam no `COMMIT` ou migrations (`test_documento.py`, `test_validacao.py`) |

### IA continua mockada

Trecho de `tests/integration/test_pergunta.py`. O Ollama não roda no CI. O teste de integração mocka **só** a IA e deixa o resto real (view, JWT, gravação do `RegistroPergunta` no banco):

```python
@patch(PERGUNTAR_IA, return_value="A arquitetura deve ... [1]")
@patch(BUSCAR, return_value=TRECHOS)
def test_resposta_com_fontes(self, mock_buscar, mock_ia):
    response = self._perguntar("Quais os requisitos da arquitetura?")

    self.assertEqual(response.status_code, 200)
    dados = response.json()
    self.assertTrue(dados["encontrou"])
    self.assertEqual(dados["fontes"][0]["documento"], "CxP 70000")

    registro = RegistroPergunta.objects.get()
    self.assertEqual(registro.quantidade_fontes, 2)
```

## Como funciona no frontend

### MSW: um backend falso na camada de rede

O componente chama `fetch('http://localhost:8000/api/...')` normalmente. O MSW intercepta a requisição antes de ela sair da máquina e devolve a resposta definida no teste. Assim o teste confere URL, método, headers e corpo de verdade, sem precisar do Django rodando.

`tests/integration/server.js`:

```js
import { setupServer } from 'msw/node';

export const API = 'http://localhost:8000/api';

export const server = setupServer();
```

`tests/integration/setup.js` (carregado só no projeto `integration`):

```js
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => {
  server.resetHandlers();
  localStorage.clear();
});
afterAll(() => server.close());
```

`onUnhandledRequest: 'error'` faz o teste falhar se o componente chamar uma URL que o teste não esperava.

### Exemplo: fluxo completo de login

```js
import { http, HttpResponse } from 'msw';
import App from '@/App.vue';
import { API, server } from './server';

it('faz login, aceita os termos e chega ao painel do usuário', async () => {
  let corpoRecebido;
  server.use(
    http.post(`${API}/auth/login/`, async ({ request }) => {
      corpoRecebido = await request.json();
      return HttpResponse.json({ token: 'jwt-de-teste', user: USUARIO });
    }),
  );
  const wrapper = mount(App);

  await wrapper.find('#engineerId').setValue('Engenharia@Akaer.com.br');
  await wrapper.find('#password').setValue('eng123');
  await wrapper.find('form').trigger('submit');
  await vi.waitFor(() => expect(wrapper.find('.btn-accept').exists()).toBe(true));
  await wrapper.find('.btn-accept').trigger('click');

  expect(corpoRecebido).toEqual({ email: 'engenharia@akaer.com.br', password: 'eng123' });
  expect(localStorage.getItem('akaer_token')).toBe('jwt-de-teste');
  expect(wrapper.text()).toContain('Bem-vindo de volta, Carlos Eduardo');
});
```

Como a requisição é HTTP de verdade, a resposta chega depois de alguns ciclos do event loop. Por isso o teste usa `vi.waitFor(...)`, que tenta o `expect` de novo até passar (ou estourar o tempo limite).

### Exemplo: simular erro da API

```js
it('avisa quando o serviço de IA está fora do ar', async () => {
  server.use(http.post(`${API}/perguntas/`, () => HttpResponse.json({}, { status: 503 })));

  const wrapper = await perguntar('Como marcar a carga?');

  expect(wrapper.text()).toContain('O serviço de IA está indisponível no momento');
});
```

### Por que jsdom e não happy-dom

O happy-dom tem um `fetch` próprio que o MSW não intercepta (a requisição sairia para `localhost:8000` de verdade e daria `ECONNREFUSED`). O jsdom usa o `fetch` nativo do Node, que o MSW intercepta. Por isso o `vite.config.js` tem dois projetos:

```js
projects: [
  { extends: true, test: { name: 'unit', include: ['tests/unit/**/*.spec.js'], environment: 'happy-dom' } },
  {
    extends: true,
    test: {
      name: 'integration',
      include: ['tests/integration/**/*.spec.js'],
      environment: 'jsdom',
      setupFiles: ['tests/integration/setup.js'],
    },
  },
],
```

## CI

No GitHub Actions (`.github/workflows/ci.yml`) o job `integration-tests` roda depois do `unit-tests`:

```text
lint ──> unit-tests ──> integration-tests
```

No backend, o job sobe os mesmos dois bancos do `docker-compose.yml` como **services** do GitHub Actions:

```yaml
integration-tests:
  needs: unit-tests
  services:
    postgres:
      image: pgvector/pgvector:0.8.6-pg17
      env: { POSTGRES_USER: rag, POSTGRES_PASSWORD: rag, POSTGRES_DB: rag }
      ports: ["5432:5432"]
    credenciais-postgres:
      image: postgres:17
      env: { POSTGRES_USER: credenciais, POSTGRES_PASSWORD: credenciais, POSTGRES_DB: credenciais }
      ports: ["5433:5432"]
  env:
    DEFAULT_DB_ENGINE: django.db.backends.postgresql
    DEFAULT_DB_NAME: rag
    # ...
  steps:
    - run: pip install -r requirements-dev.txt
    - run: pytest tests/integration --cov --cov-report=term
```

## Passo a passo para escrever um teste novo

1. Escolha um fluxo que atravessa camadas (ex.: "admin cadastra usuário e ele consegue logar").
2. Crie o arquivo em `tests/integration/`.
3. **Backend**: herde de `TestCase` (ou `TransactionTestCase` se envolver trigger/migration), declare `databases` se usar os dois bancos e crie os dados no `setUpTestData`/`setUp`. Chame a API pelo `self.client`.
4. **Frontend**: registre as respostas da API com `server.use(http.<metodo>(url, handler))`, monte a tela e interaja como o usuário. Use `vi.waitFor` para esperar a resposta.
5. Mocke só o que não dá para rodar no CI (Ollama). Banco, rotas e componentes ficam reais.
6. Confira o resultado visível (status/JSON, texto na tela) **e** o efeito colateral (linha no banco, token no `localStorage`).

## Problemas comuns

| Sintoma | Causa | Solução |
|---|---|---|
| `connection refused` na porta 5432/5433 | Bancos não estão rodando | `docker compose up -d` |
| `type "vector" does not exist` ou erro de trigger | Banco de negócio está em SQLite | Configure `DEFAULT_DB_*` no `.env` para o PostgreSQL |
| `database "test_rag" already exists` | Execução anterior interrompida | Rode com `--create-db` |
| `Database access not allowed ... 'credenciais_db'` | Faltou declarar o segundo banco | Adicione `databases = {"default", "credenciais_db"}` na classe |
| `ECONNREFUSED ::1:8000` no frontend | Teste rodando no happy-dom | Coloque o arquivo em `tests/integration/` (projeto com jsdom) |
| `[MSW] Error: intercepted a request without a matching request handler` | O componente chamou uma URL sem handler | Adicione o `server.use(...)` para essa URL no teste |
