# Testes unitários

Guia para rodar, entender e escrever os testes unitários do **AkaVision** (backend e frontend).

> Testes de integração ficam em outro guia: [testes-integracao.md](testes-integracao.md).

## O que é um teste unitário aqui

Um teste unitário verifica **uma unidade de código isolada** (uma função, um decorator, um componente Vue) sem depender de nada externo:

- sem banco de dados
- sem rede (backend, Ollama, internet)
- sem ordem entre testes: cada um prepara o que precisa

Tudo que é externo é trocado por um **mock** (um objeto falso que responde o que o teste mandar). Por isso a suíte inteira roda em menos de 1 segundo e não precisa de Docker.

## Ferramentas

| Repositório | Ferramenta | Para que serve |
|---|---|---|
| Backend | [pytest](https://docs.pytest.org) 9 | Executor de testes |
| Backend | [pytest-django](https://pytest-django.readthedocs.io) 4 | Carrega as settings do Django no pytest |
| Backend | [pytest-cov](https://pytest-cov.readthedocs.io) 7 | Relatório de cobertura |
| Backend | `unittest.mock` (biblioteca padrão) | Mocks de Ollama, `requests`, funções |
| Frontend | [Vitest](https://vitest.dev) 5 | Executor de testes (mesma config do Vite) |
| Frontend | [Vue Test Utils](https://test-utils.vuejs.org) 2 | Monta componentes Vue e simula cliques/digitação |
| Frontend | [happy-dom](https://github.com/capricorn86/happy-dom) | DOM simulado (sem abrir navegador) |

## Estrutura

```text
API-6SEM-BACKEND/
├── requirements-dev.txt        # dependências de teste (inclui requirements.txt)
├── pyproject.toml              # [tool.pytest.ini_options] e [tool.coverage.run]
└── tests/
    └── unit/
        ├── test_jwt_auth.py          # core_api/utils/jwt_auth.py
        ├── test_db_router.py         # credenciais/db_router.py
        ├── test_cors_middleware.py   # core_api/utils/cors_middleware.py
        └── test_pergunta_service.py  # core_api/services/rag/pergunta.py

API-6SEM-FRONTEND/
├── vite.config.js              # bloco "test" do Vitest
└── tests/
    └── unit/
        ├── LoginView.spec.js
        └── TermsView.spec.js
```

Regra: o nome do arquivo de teste aponta para o arquivo testado (`jwt_auth.py` → `test_jwt_auth.py`, `LoginView.vue` → `LoginView.spec.js`).

## Como rodar

### Backend

Pré-requisito: Python 3.12.

```bash
cd API-6SEM-BACKEND
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
pytest tests/unit
```

Saída esperada:

```text
tests/unit/test_cors_middleware.py ..                    [  6%]
tests/unit/test_db_router.py ..........                  [ 41%]
tests/unit/test_jwt_auth.py ........                     [ 68%]
tests/unit/test_pergunta_service.py .........            [100%]
29 passed in 0.40s
```

Comandos úteis:

```bash
pytest tests/unit -v                                   # nome de cada teste
pytest tests/unit/test_jwt_auth.py                     # um arquivo
pytest tests/unit -k expirado                          # só testes com "expirado" no nome
pytest tests/unit --cov --cov-report=term-missing      # cobertura com linhas não testadas
pytest tests/unit --cov --cov-report=html              # relatório em htmlcov/index.html
```

### Frontend

Pré-requisito: Node 22 ou mais novo (exigido pelo Vitest 5).

```bash
cd API-6SEM-FRONTEND
npm install
npm run test:unit
```

Saída esperada:

```text
 Test Files  2 passed (2)
      Tests  8 passed (8)
```

Comandos úteis:

```bash
npx vitest tests/unit                        # modo watch: reroda ao salvar
npx vitest run tests/unit/LoginView.spec.js  # um arquivo
npm run test:unit -- --reporter=verbose      # nome de cada teste
npx vitest run tests/unit -t "senha"         # só testes com "senha" no nome
npm run test:coverage                        # cobertura em coverage/index.html
```

## Anatomia de um teste

Todo teste segue o padrão **AAA** (Arrange, Act, Assert), separado por linhas em branco:

1. **Arrange**: prepara os dados e os mocks
2. **Act**: executa a unidade testada
3. **Assert**: confere o resultado

### Backend: decorator de autenticação

```python
from django.test import RequestFactory

from core_api.utils.jwt_auth import generate_token, require_role


def view(request):
    return JsonResponse({"email": request.auth_user["email"]})


def test_require_role_bloqueia_papel_sem_permissao():
    protegida = require_role("ADMINISTRADOR")(view)
    token = generate_token({"id": "1", "email": "eng@akaer.com.br", "role_code": "ENGENHARIA"})
    requisicao = RequestFactory().get("/", headers={"Authorization": f"Bearer {token}"})

    response = protegida(requisicao)

    assert response.status_code == 403
```

O `RequestFactory` do Django cria uma requisição em memória: não sobe servidor e não acessa banco.

### Backend: vários casos com `parametrize`

Em vez de copiar o mesmo teste, o `@pytest.mark.parametrize` roda a função uma vez por linha:

```python
@pytest.mark.parametrize("p1, p2, esperado", [(7, 7, "7"), (42, 43, "42-43")])
def test_formatar_paginas(p1, p2, esperado):
    assert formatar_paginas(p1, p2) == esperado
```

### Backend: mock de dependência externa

`responder_pergunta` chama o banco vetorial e o Ollama. No teste unitário as duas funções são trocadas com `patch.object`, então dá para testar só a regra de negócio (filtro de similaridade, montagem das fontes):

```python
from unittest.mock import patch

from core_api.services.rag import pergunta


def test_sem_trecho_relevante_nao_chama_a_ia():
    trecho_irrelevante = {..., "similaridade": 0.1}

    with (
        patch.object(pergunta, "buscar_trechos", return_value=[trecho_irrelevante]),
        patch.object(pergunta, "perguntar_ia") as ia,
    ):
        resultado = pergunta.responder_pergunta("Como marcar a carga?")

    ia.assert_not_called()
    assert resultado["encontrou"] is False
```

Também dá para simular falhas:

```python
def test_ollama_fora_do_ar_vira_erro_ia():
    with patch.object(pergunta.requests, "post", side_effect=requests.ConnectionError):
        with pytest.raises(ErroIA):
            pergunta._ollama("/api/chat", {})
```

### Frontend: evento de componente

```js
import { describe, expect, it } from 'vitest';
import { mount } from '@vue/test-utils';
import TermsView from '@/components/TermsView.vue';

describe('TermsView', () => {
  it('emite accept ao aceitar os termos', async () => {
    const wrapper = mount(TermsView);

    await wrapper.find('.btn-accept').trigger('click');

    expect(wrapper.emitted('accept')).toHaveLength(1);
  });
});
```

### Frontend: mock do `fetch`

O `LoginView` chama `fetch`. No unitário ele vira um `vi.fn()`, então o teste controla a resposta e confere o que foi enviado:

```js
beforeEach(() => {
  fetchMock = vi.fn();
  vi.stubGlobal('fetch', fetchMock);
});

afterEach(() => {
  vi.unstubAllGlobals();
});

it('mostra a mensagem do servidor quando o login falha', async () => {
  fetchMock.mockResolvedValue({ ok: false, json: () => Promise.resolve({ detail: 'Usuário bloqueado.' }) });
  const wrapper = mount(LoginView);

  await wrapper.find('#engineerId').setValue('eng@akaer.com.br');
  await wrapper.find('#password').setValue('errada');
  await wrapper.find('form').trigger('submit');
  await flushPromises();

  expect(wrapper.find('.alert-error').text()).toBe('Usuário bloqueado.');
});
```

`flushPromises()` espera as promessas pendentes (o `fetch` falso e o `response.json()`) terminarem antes do `expect`.

## Passo a passo para escrever um teste novo

1. Escolha uma função ou componente com regra de negócio (validação, cálculo, decisão).
2. Crie o arquivo em `tests/unit/` com o nome do arquivo testado (`test_<arquivo>.py` ou `<Componente>.spec.js`).
3. Liste os casos: caminho feliz, entrada inválida, erro da dependência externa.
4. Troque toda dependência externa por mock (`patch.object` no Python, `vi.fn`/`vi.stubGlobal` no JS).
5. Escreva no padrão AAA, com o nome do teste dizendo o comportamento (`test_token_expirado_e_recusado`).
6. Rode só o arquivo novo, depois a suíte inteira.
7. Faça o teste falhar de propósito uma vez (mude o valor esperado) para ter certeza de que ele testa algo.

Se o teste precisar de banco ou de outro serviço rodando, ele não é unitário: vai para `tests/integration/`.

## CI

Os dois repositórios rodam os unitários no GitHub Actions em todo PR e push para `main` e `develop` (`.github/workflows/ci.yml`, job `unit-tests`). O job roda depois do `lint`; se algum teste falhar, o PR fica vermelho e não pode ser mesclado.

```text
lint ──> unit-tests ──> integration-tests
```

## Problemas comuns

| Sintoma | Causa | Solução |
|---|---|---|
| `RuntimeError: Database access not allowed` | O teste tocou no banco | Use mock ou mova o teste para `tests/integration/` |
| `ModuleNotFoundError: No module named 'core_api'` | pytest rodado fora da raiz do backend | Rode `pytest` na pasta `API-6SEM-BACKEND` |
| `Unsupported engine` / erro de sintaxe ao rodar o Vitest | Node antigo | Instale Node 22+ |
| Teste do Vue passa sem esperar a resposta | Faltou `await flushPromises()` | Aguarde as promessas antes do `expect` |
