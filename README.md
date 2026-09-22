# 🧪 API Test Automation | Postman + Newman

Projeto de automação de testes de API desenvolvido do ponto de vista de **QA**, com o objetivo de demonstrar boas práticas de teste de API, organização de coleção, chaining de requisições, execução via linha de comando (CI/CD) e geração de relatórios.

A API testada é a **[ReqRes](https://reqres.in)**, uma API pública gratuita amplamente usada para prática de testes, que simula um CRUD de usuários e fluxos de autenticação.

---

## 🎯 Objetivo

- Praticar e demonstrar competências de automação de testes de API como QA.
- Servir de peça de portfólio, mostrando organização, clareza e boas práticas.
- Ser reaproveitado como base de estudo (adicionar novos cenários, negativos, de performance, etc).

---

## 🛠️ Tecnologias

| Ferramenta | Uso |
|---|---|
| [Postman](https://www.postman.com/) | Criação e organização da coleção de testes |
| [Newman](https://github.com/postmanlabs/newman) | Execução da coleção via CLI |
| [newman-reporter-htmlextra](https://github.com/DannyDainton/newman-reporter-htmlextra) | Geração de relatório HTML visual |
| Node.js | Runtime necessário para rodar o Newman |
| GitHub Actions | Pipeline de CI para rodar os testes automaticamente |

---

## 📁 Estrutura do projeto

```
api-test-automation-postman/
├── collections/
│   └── API_QA_Tests.postman_collection.json   # Coleção com todos os requests e testes
├── environments/
│   └── dev.postman_environment.json           # Variáveis de ambiente (baseUrl, tokens, ids)
├── reports/                                    # Relatórios gerados após execução (gitignored)
├── .github/
│   └── workflows/
│       └── api-tests.yml                       # Pipeline de CI (GitHub Actions)
├── package.json
├── .gitignore
└── README.md
```

---

## 🗂️ Organização da coleção (por ponto da API)

A coleção é dividida em pastas nomeadas pelo **ponto da API que está sendo testado**, não apenas por "CRUD genérico". Isso facilita achar rapidamente o cenário que você precisa:

```
📁 🔐 Autenticação › Login          → POST /login
📁 📝 Autenticação › Cadastro       → POST /register
📁 🔍 Usuários › Consultas          → GET /users, GET /users/{id}
📁 ➕ Usuários › Criação            → POST /users
📁 ✏️ Usuários › Atualização        → PUT/PATCH /users/{id}
📁 🗑️ Usuários › Remoção            → DELETE /users/{id}
📁 🔄 Fluxo Ponta a Ponta (E2E)     → Login → Criar → Atualizar → Remover, encadeados
```

Cada pasta tem uma **descrição** (visível no Postman ao clicar nela ou na aba "Documentation") explicando exatamente qual endpoint e qual regra de negócio está sendo validada ali.

## ♻️ Reutilização de código (QAHelpers)

Em vez de repetir a mesma lógica de `pm.test()` em cada request, todas as validações comuns ficam centralizadas em **uma única biblioteca de funções**, chamada `QAHelpers`, definida uma única vez na variável de coleção `helpersLib` (aba **Variables** da coleção no Postman).

Cada request carrega essa biblioteca com uma linha e reutiliza as funções prontas:

```javascript
// Início do Test script de QUALQUER request da coleção
eval(pm.collectionVariables.get('helpersLib'));

var jsonData = pm.response.json();

QAHelpers.validarStatusCode(200);
QAHelpers.validarCamposObrigatorios(jsonData, ['id', 'email'], 'Usuário');
QAHelpers.salvarVariavel('userId', jsonData.id);
```

Funções disponíveis na biblioteca:

| Função | O que faz |
|---|---|
| `validarStatusCode(codigo)` | Confere o status HTTP da resposta |
| `validarCamposObrigatorios(objeto, campos, nome)` | Confere se um objeto JSON tem os campos esperados |
| `validarValorCampo(json, caminho, valor, descricao)` | Confere o valor exato de um campo (suporta `data.id`) |
| `validarCorpoVazio()` | Confere resposta vazia (usado no 404) |
| `validarMensagemErro(mensagem)` | Confere a mensagem de erro retornada pela API |
| `salvarVariavel(nome, valor)` | Salva valor em variável de ambiente + loga no console |
| `validarTempoResposta(maxMs)` | Confere tempo de resposta (usada globalmente, ver abaixo) |

**Por que `eval()` e não só chamar a função direto?** O Postman executa cada request em um sandbox JavaScript isolado — não existe `import`/`require` de arquivos entre requests. O `eval()` a partir de uma variável de coleção é o padrão usado pela comunidade Postman/QA para simular "importar um módulo compartilhado", garantindo que o mesmo código funcione igual no Postman App e no Newman/CI.

## 🌍 Validação global automática

O teste de tempo de resposta (`< 2000ms`) **não é escrito em nenhum request individual**. Ele está configurado uma única vez no evento **Test** do nível da **coleção**, e o Postman o executa automaticamente depois de cada request, em qualquer pasta. Se amanhã você quiser mudar o limite para 1500ms, muda em um único lugar.

## ✅ Cenários cobertos

### 🔐 Autenticação › Login
- Login com sucesso — valida token retornado e salva em variável
- Login sem senha — valida erro 400 e mensagem exata

### 📝 Autenticação › Cadastro
- Cadastro com sucesso — valida id e token retornados
- Cadastro sem senha — valida erro 400 e mensagem exata

### 🔍 Usuários › Consultas
- Listar usuários (paginação) — valida schema e campos obrigatórios de cada item da lista
- Buscar usuário existente — valida dados e faz *chaining* (salva `id` em variável)
- Buscar usuário inexistente — valida tratamento de erro 404 e corpo vazio

### ➕ Usuários › Criação
- Criar usuário (`POST`) — valida status 201, campos e valor persistido

### ✏️ Usuários › Atualização
- Atualizar usuário (`PUT`) — atualização completa
- Atualizar usuário (`PATCH`) — atualização parcial

### 🗑️ Usuários › Remoção
- Remover usuário (`DELETE`) — valida status 204

### 🔄 Fluxo Ponta a Ponta (E2E)
- Encadeia Login → Criar usuário → Atualizar → Remover, reaproveitando os mesmos endpoints testados isoladamente acima, agora simulando a jornada real de um usuário na API.

### 🌍 Validações globais (aplicadas a todos os requests automaticamente)
- Tempo de resposta abaixo de 2000ms (configurado uma única vez no nível da coleção)

> Todos os testes usam `pm.test()` com asserções do Chai (via `pm.expect`), seguindo o padrão de testes do Postman, agora encapsuladas nas funções do `QAHelpers` para evitar duplicação.

---

## ▶️ Como executar

### Opção 1 — Via interface do Postman
1. Abra o Postman.
2. Importe o arquivo `collections/API_QA_Tests.postman_collection.json`.
3. Importe o arquivo `environments/dev.postman_environment.json`.
4. Selecione o environment **dev** no canto superior direito.
5. Rode a coleção inteira usando o **Collection Runner** ou execute requisições individualmente.

### Opção 2 — Via linha de comando (Newman)

Pré-requisitos: [Node.js](https://nodejs.org/) instalado.

```bash
# Instalar dependências
npm install

# Rodar todos os testes e gerar relatório HTML em reports/report.html
npm test

# Rodar e gerar relatório no formato JUnit (útil para integrações de CI)
npm run test:junit
```

Após a execução, o relatório HTML estará disponível em `reports/report.html`.

---

## 🔄 Integração Contínua (CI)

O projeto já vem com um workflow do **GitHub Actions** (`.github/workflows/api-tests.yml`) que:

1. Faz checkout do código;
2. Configura o Node.js;
3. Instala as dependências;
4. Executa a coleção via Newman;
5. Publica o relatório de testes como artefato do workflow.

O pipeline roda automaticamente em todo `push`/`pull request` para a branch `main`, e também pode ser disparado manualmente.

---

## 🧠 Decisões técnicas e boas práticas aplicadas

- **Variáveis de ambiente** em vez de valores fixos (`baseUrl`, `userId`, `authToken`) — facilita trocar de API/ambiente sem alterar a coleção.
- **Chaining de requisições**: o `id` obtido no `GET` é reaproveitado nos requests de `PUT`, `PATCH` e `DELETE`, simulando um fluxo real de teste.
- **Testes positivos e negativos**: cada fluxo de CRUD/autenticação possui pelo menos um cenário de erro esperado.
- **Script de pré-requisição na coleção**: adiciona o header exigido pela ReqRes de forma centralizada, evitando repetição em cada request.
- **Separação de responsabilidades**: coleção, environment e pipeline de CI em arquivos separados e versionáveis.

---

## 👩‍💻 Sobre

Projeto criado para fins de estudo e portfólio, demonstrando uma abordagem de QA para automação de testes de API usando Postman/Newman.

Sinta-se à vontade para clonar, adaptar e usar como base de treino.
