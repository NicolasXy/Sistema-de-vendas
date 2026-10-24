# Sistema de Vendas com Comissões, Metas e Performance

API REST em **Node.js + Express + MySQL** (`mysql2/promise`, `async/await`) para gerenciar vendedores, vendas, comissões e metas de uma rede de lojas com unidades nas regiões Norte, Nordeste, Sudeste e Sul.

## Tecnologias

- Node.js 18+
- Express 4
- mysql2 (`mysql2/promise`) com consultas parametrizadas (`execute` + `?`) contra SQL Injection
- dotenv (credenciais fora do código)

## Estrutura

```
sistema-vendas/
├── sql/db_vendas.sql        # criação do banco, tabelas, FKs, constraints e dados de teste
├── src/
│   ├── db.js                # pool de conexão (lê o .env)
│   ├── erros.js             # ErroApp (erro com código HTTP) e validarId
│   ├── comissao.js          # calcularComissao() – única implementação da regra
│   ├── vendedores.js        # cadastrar/listar/consultar/atualizar/desativar
│   ├── vendas.js            # registrar/listar/consultar por vendedor e região/excluir
│   ├── relatorios.js        # gerarRelatorioPerformance()
│   ├── app.js               # rotas Express e middleware de erros
│   └── server.js            # inicialização (testa a conexão e sobe o servidor)
├── tests/
│   ├── comissao.test.js     # testes unitários (sem banco)
│   └── api.test.js          # testes de integração (com banco)
├── requests.http            # requisições prontas (REST Client / Postman / Insomnia)
├── .env.example
└── package.json
```

## Instalação e execução

1. **Pré-requisitos:** Node.js 18+ e MySQL 8.0.16+ (necessário para as constraints `CHECK`).

2. **Instale as dependências**
   ```bash
   npm install
   ```

3. **Crie o banco e carregue os dados de teste**
   ```bash
   mysql -u root -p < sql/db_vendas.sql
   ```
   O script recria o banco `db_vendas` do zero (`DROP DATABASE IF EXISTS`), cria as tabelas `vendedores`, `regioes` e `vendas` e insere 4 regiões, 5 vendedores e 18 vendas (16 em out/2026 e 2 em set/2026).

4. **Configure as variáveis de ambiente**
   ```bash
   cp .env.example .env
   ```
   Edite o `.env` com os dados do seu MySQL:
   ```
   DB_HOST=localhost
   DB_PORT=3306
   DB_USER=root
   DB_PASSWORD=sua_senha_aqui
   DB_NAME=db_vendas
   PORT=3000
   ```
   > O `.env` está no `.gitignore`; nenhuma senha fica no código-fonte.

5. **Inicie a API**
   ```bash
   npm start
   ```
   A API sobe em `http://localhost:3000`. Se o MySQL estiver inacessível, o servidor informa o erro e encerra.

## Modelagem

| Tabela | Chave | Observações |
| --- | --- | --- |
| `vendedores` | PK `id_vendedor` | `ativo` (BOOLEAN); `CHECK` meta ≥ 0 e % base entre 0 e 100 |
| `regioes` | PK `id_regiao` | `nome_regiao` único e restrito a Norte/Nordeste/Sudeste/Sul; bônus regional |
| `vendas` | PK `id_venda` | FK → `vendedores` e FK → `regioes` (`ON DELETE RESTRICT`); `tipo_cliente ENUM('PF','PJ')`; `CHECK` valor > 0 |

Como as FKs usam `RESTRICT` e o vendedor é apenas **desativado** (nunca apagado), o histórico de vendas é sempre preservado.

## Regra de comissão

```
comissão = valor_venda × (percentual_base + bônus_cliente + bônus_região) / 100
```

Os percentuais são **somados**, sem composição.

| Cliente | Bônus | | Região | Bônus |
| --- | --- | --- | --- | --- |
| PF | 2% | | Norte | 1% |
| PJ | 3% | | Nordeste | 1% |
| | | | Sudeste | 0% |
| | | | Sul | 0,5% |

Exemplo: R$ 2.000,00, PJ, Norte, base 5% → 5 + 3 + 1 = 9% → **R$ 180,00**.

A lógica fica apenas em `calcularComissao()` (`src/comissao.js`). O bônus de cliente é uma constante do código; o bônus da região vem da tabela `regioes`.

## Endpoints

| Método | Endpoint | Operação | Respostas |
| --- | --- | --- | --- |
| POST | `/vendedores` | Cadastrar vendedor | 201, 400 |
| GET | `/vendedores` | Listar (filtro opcional `?ativo=1`) | 200 |
| GET | `/vendedores/:id` | Consultar vendedor | 200, 400, 404 |
| PUT | `/vendedores/:id` | Atualizar `meta_mensal` e/ou `percentual_base` | 200, 400, 404 |
| PATCH | `/vendedores/:id/desativar` | Desativar vendedor | 200, 404 |
| POST | `/vendas` | Registrar venda | 201, 400, 404, 409, 422 |
| GET | `/vendas` | Listar vendas | 200 |
| GET | `/vendas/vendedor/:id` | Vendas por vendedor | 200, 404 |
| GET | `/vendas/regiao/:id` | Vendas por região | 200, 404 |
| DELETE | `/vendas/:id` | Excluir venda | 200, 404 |
| GET | `/relatorios/performance` | Relatório gerencial | 200, 400 |
| GET | `/saude` | Verifica a conexão com o banco | 200, 503 |

Erros retornam JSON `{ "erro": "mensagem" }`. Código **409** = código de venda duplicado; **422** = vendedor inativo; **503** = banco indisponível; **500** = erro interno.

### Exemplos de corpo

`POST /vendedores`
```json
{ "nome": "Fernanda Costa", "meta_mensal": 18000, "percentual_base": 5 }
```

`POST /vendas` (`id_venda` e `data_venda` são opcionais: sem eles, o código é gerado automaticamente e a data é a atual)
```json
{ "id_venda": 2001, "id_vendedor": 1, "id_regiao": 1, "data_venda": "2026-10-23 10:00:00", "valor_venda": 2000, "tipo_cliente": "PJ" }
```
Regiões: `1` Norte, `2` Nordeste, `3` Sudeste, `4` Sul.

`PUT /vendedores/1`
```json
{ "meta_mensal": 22000, "percentual_base": 5.5 }
```

### Relatório de performance

```
GET /relatorios/performance?inicio=2026-10-01&fim=2026-10-31
```

- `inicio` e `fim` (YYYY-MM-DD, ambos inclusivos) são obrigatórios.
- `mes` (YYYY-MM) é opcional e define o mês de referência das metas; por padrão é o mês de `inicio`.

Indicadores retornados: total de vendas, vendas por região (R$), vendas por tipo de cliente (R$), melhor vendedor (maior valor vendido), maior comissão acumulada, metas atingidas no mês, comissão média geral e comissão média por região. Todos são calculados com SQL (`JOIN`, `GROUP BY`, `SUM()`, `AVG()` e `COUNT()`).

Com os dados de teste, o relatório de outubro/2026 indica que **Ana Souza (R$ 20.800 / meta 20.000)** e **Carla Mendes (R$ 33.800 / meta 30.000)** atingiram a meta; Bruno, Diego e Elisa não.

## Funções assíncronas

| Função | Arquivo |
| --- | --- |
| `cadastrarVendedor()`, `listarVendedores()`, `atualizarVendedor()`, `desativarVendedor()` | `src/vendedores.js` |
| `registrarVenda()`, `listarVendas()`, `consultarVendasPorVendedor()`, `excluirVenda()` | `src/vendas.js` |
| `calcularComissao()` | `src/comissao.js` |
| `gerarRelatorioPerformance()` | `src/relatorios.js` |

Todas usam `async/await`, `try/catch` e consultas parametrizadas. Também existem `consultarVendedor()` e `consultarVendasPorRegiao()` para os endpoints `GET /vendedores/:id` e `GET /vendas/regiao/:id`.

## Testes

**Unitários** (regra de comissão, não precisam de banco):
```bash
npm test
```

**Integração** (precisam do MySQL com o `db_vendas.sql` carregado e o `.env` configurado):
```bash
npm run test:api
```
Cobrem: cadastro de vendedor, venda com comissão correta (R$ 180,00), código duplicado, vendedor inexistente/inativo, região inexistente, valor inválido, atualização de meta, consultas por vendedor e região, relatório com atingimento de meta, exclusão e tentativa de SQL Injection. Os dados criados são removidos ao final.

**Teste manual (Postman / Insomnia / Thunder Client):** o arquivo `requests.http` traz todas as requisições prontas. Com a extensão *REST Client* do VS Code basta clicar em "Send Request". Para Postman/Insomnia, importe ou copie as requisições.

Para testar o **tratamento de erro de conexão**, pare o MySQL com a API rodando e chame `GET /saude`: a resposta é `503` com a mensagem de banco indisponível.
