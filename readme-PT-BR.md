# Stock Control API

[English](README.md) · **Português**

API REST multi-tenant para controle de estoque. Cada empresa (organização) tem seus próprios usuários, produtos e estoques. O saldo de cada produto é calculado a partir do histórico de movimentações de entrada e saída, em vez de ficar guardado num contador que é alterado a cada operação.

**Stack:** Node.js · Express · Sequelize · PostgreSQL (drivers de MySQL e SQL Server incluídos) · JWT · bcrypt

---

## O que faz

- **Cadastro autônomo de empresa**: ao criar uma organização, a API cria automaticamente o primeiro usuário `admin`, com senha gerada aleatoriamente. Essa senha aparece uma única vez, na resposta.
- **Controle de acesso por cargo**: são dois cargos, `admin` e `employee`. O admin gerencia usuários e a organização. Os dois cargos operam produtos, estoques e movimentações.
- **Isolamento por empresa**: o `organization_id` vem do JWT, não do corpo da requisição. Assim, as consultas de produtos, estoques e usuários ficam filtradas pela empresa de quem faz a chamada.
- **Histórico de movimentações**: toda alteração de estoque vira uma movimentação `in` ou `out`, ligada a um produto, a um estoque e ao usuário que a registrou. Ao consultar um estoque, a API devolve o saldo atual por produto, calculado a partir desse histórico.

## Decisões técnicas

| Decisão | Motivo | Trade-off |
| --- | --- | --- |
| Saldo calculado a partir das movimentações (`src/fns/get-product-movements.js`) | Trilha de auditoria completa: sempre se sabe quem movimentou o quê, e o saldo nunca diverge do histórico | O saldo é recalculado a cada leitura. Funciona bem em pequena escala; com muitos registros, precisaria de um total consolidado (snapshot ou view materializada) |
| ID da empresa tirado do token | O cliente não consegue acessar dados de outra empresa só trocando um campo no body | Todo método de serviço precisa lembrar de filtrar por ele (ver *Limitações conhecidas*) |
| Estrutura em camadas: `route → api → service → model` | As rotas só ligam os middlewares, os controllers tratam o HTTP e os serviços concentram as regras de negócio | Mais arquivos por entidade do que um app Express de estrutura plana |
| Senha aleatória para o admin na criação da empresa | Nunca existem credenciais padrão | A senha só aparece uma vez, e ainda não há fluxo de redefinição |
| Senhas com bcrypt (custo 12); JWT expira em 1 hora | Armazenamento com hash lento e tokens de vida curta | Sem refresh token: o cliente precisa logar de novo quando o token expira |

## Modelo de dados

```
organizations ─┬─< users
               ├─< products
               └─< inventories ─< inventoryMovements >─ products
                                         │
                                         └─ users (quem fez a movimentação)
```

## Rodando localmente

**Requisitos:** Node.js 18+ e um banco PostgreSQL.

```bash
git clone https://github.com/GabRfg/StockControl.git
cd StockControl
npm install
```

Crie um arquivo `.env` na raiz do projeto:

```env
DB_DIALECT=postgres
DB_HOST=localhost
DB_PORT=5432
DB_NAME=stock_control
DB_USER=postgres
DB_PASSWORD=sua_senha
```

Inicie a API. As tabelas são criadas automaticamente na primeira execução, via `sequelize.sync`:

```bash
npm run dev
```

O servidor sobe em `http://localhost:3000`.

## API

Todas as rotas começam com `/api/v1`. As rotas protegidas esperam o JWT no header `Authorization`.

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| POST | `/organization` | público | Cria a organização e o usuário admin |
| POST | `/login` | público | Retorna um JWT |
| GET / PUT / DELETE | `/organization/:id` | admin | Gerencia a organização |
| GET | `/user/info` | qualquer | Perfil do usuário logado |
| PUT / DELETE | `/user` | qualquer | Atualiza ou exclui a própria conta |
| GET / POST | `/user` | admin | Lista ou cria usuários da organização |
| GET / PUT / DELETE | `/user/:id` | admin | Gerencia um usuário específico |
| GET / POST | `/products` | qualquer | Lista ou cria produtos |
| GET / PUT / DELETE | `/products/:id` | qualquer | Gerencia um produto |
| GET / POST | `/inventory` | qualquer | Lista ou cria estoques |
| GET | `/inventory/:id` | qualquer | Detalhes do estoque **com o saldo atual por produto** |
| PUT / DELETE | `/inventory/:id` | qualquer | Gerencia um estoque |
| GET / POST | `/inventoryMovement/:inventoryId` | qualquer | Lista ou registra movimentações (`type`: `in` ou `out`) |
| GET / PUT / DELETE | `/inventoryMovement/:inventoryId/:id` | qualquer | Gerencia uma movimentação |

**Exemplo: registrar uma entrada de estoque**

```http
POST /api/v1/inventoryMovement/1
Authorization: <token>
Content-Type: application/json

{ "type": "in", "amount": 50, "productId": 3 }
```

## Estrutura do projeto

```
src/
├── route/        # Routers do Express e verificação de cargo por endpoint
├── api/          # Controllers: leem a requisição, chamam o serviço e montam a resposta
├── service/      # Regras de negócio e validações
├── model/        # Models e associações do Sequelize
├── middleware/   # Autenticação JWT e autorização por cargo
├── fns/          # Funções auxiliares (cálculo de saldo, gerador de senha)
└── database.js   # Conexão do Sequelize a partir das variáveis de ambiente
```

## Limitações conhecidas / roadmap

- [ ] As rotas de movimentação e `/organization/:id` ainda não verificam se o recurso pertence à empresa de quem faz a chamada. A checagem de empresa deve ser aplicada na camada de serviço
- [ ] O segredo do JWT está fixo no código. Mover para variável de ambiente
- [ ] `amount` aceita qualquer valor, e movimentações `out` podem deixar o saldo negativo. Falta validar
- [ ] Todos os erros retornam HTTP 500. Mapear erros de validação para 400 e de recurso não encontrado para 404
- [ ] Testes automatizados: o Jest está instalado, mas ainda não há testes
- [ ] Trocar o `sync()` por migrations

## Licença

MIT
