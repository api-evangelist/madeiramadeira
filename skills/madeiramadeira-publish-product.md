---
name: madeiramadeira-publish-product
description: >-
  Submit a product to the MadeiraMadeira marketplace catalog and follow it through the approval
  pipeline until it is published, using the seller Marketplace API.
api: madeiramadeira:marketplace
generated: '2026-08-25'
method: generated
source: >-
  Grounded in openapi/madeiramadeira-marketplace-openapi.yml (derived from MadeiraMadeira's own
  published Postman collection). Every operationId below was verified against that spec.
operations:
  - consultaArvoreDeCategoriasPorNome
  - enviaUmOuMaisProdutos
  - buscaTodosOsProdutosPendentes
  - buscaUmProdutoAguardandoAprovacaoPeloSeuSku
  - buscaUmProdutoComDivergenciaDeInformacaoDeMatchPeloSeuSku
  - buscaUmProdutoReprovadoPeloSeuSku
  - buscaUmProdutoQueEstaEmPublicacaoPeloSeuSku
  - buscaUmProdutoPorSku
  - apagaUmProdutoPendente
---

# Publish a product on the MadeiraMadeira marketplace

Base URL `https://marketplace.madeiramadeira.com.br` (sandbox: `https://marketplace-sandbox.madeiramadeira.com.br`).
Every request carries `Content-Type: application/json` and `TOKENMM: <seller token>`.

## Before you start

Read `conventions/madeiramadeira-conventions.yml`. Two things will break you if you skip it:

- **`limit` and `offset` are PATH segments, not query parameters.** The URL is literally
  `/v1/produto/limit=100&offset=0`. There is no `?`.
- **No write on this API is idempotent.** If a call times out, re-read state before retrying.

## Step 1 — resolve the category

A product cannot be submitted without a valid `id_categoria` from MadeiraMadeira's own tree.

    GET /v1/categoria/limit={limit}&offset={offset}&nome={nome}    (consultaArvoreDeCategoriasPorNome)

Search by name to narrow it, or pull the whole tree with `consultaArvoreDeCategorias`. Do not guess
an id — a wrong category is the most common cause of a later rejection.

## Step 2 — submit the product

    POST /v1/produto     (enviaUmOuMaisProdutos)

Accepts an array, so batch. Required fields and their published limits:

| field | type | note |
|---|---|---|
| `id_categoria` | integer | from step 1 |
| `nome` | String(100) | |
| `descricao` | String(10000) | accented characters count as 2 |
| `sku` | String(30) | letters, numbers and underscore only |
| `ean` | String(14) | GS1 EAN — a global natural key |
| `marca` | String(255) | |
| `preco_de` / `preco_por` | Float(9,2) | `preco_por` has a floor of R$ 30.00 |
| `altura`, `largura`, `profundidade` | Float(9,5) | packaging, centimetres |
| `peso` | Float(10,5) | kilograms |
| `estoque` | integer | |
| `imagens` | array | at least one URL |

`atributos` is an optional array of `{nome, valor}` pairs carrying category-specific attributes.

**Handle HTTP 422 explicitly.** It means the EAN or SKU is already queued for processing. This is the
API's only replay guard and it is a natural-key check, not idempotency — it returns an error rather
than the original result. On 422, stop and poll step 3 instead of resubmitting.

## Step 3 — follow the approval pipeline

The product moves through states, each with its own read path. Poll rather than assume:

    GET /v1/produto/situacao/pendentes/limit=100&offset=0   (buscaTodosOsProdutosPendentes)
    GET /v1/produto/aguardando/{sku}                        (buscaUmProdutoAguardandoAprovacaoPeloSeuSku)
    GET /v1/produto/divergente/{sku}                        (buscaUmProdutoComDivergenciaDeInformacaoDeMatchPeloSeuSku)
    GET /v1/produto/reprovado/{sku}                         (buscaUmProdutoReprovadoPeloSeuSku)
    GET /v1/produto/empublicacao/{sku}                      (buscaUmProdutoQueEstaEmPublicacaoPeloSeuSku)
    GET /v1/produto/{sku}                                   (buscaUmProdutoPorSku — once published)

`divergente` means the submitted data conflicts with an existing catalog match on the same EAN — fix
the data and resubmit. `reprovado` means rejected outright.

Do not poll the `aenriquecer` or `enriquecendo` paths. MadeiraMadeira has marked those stages
deprecated and states they will be removed; they are flagged `deprecated: true` in the OpenAPI.

Register the `PRODUTO_APROVADO` callback instead of polling if you can receive webhooks — see
`madeiramadeira-manage-callbacks`.

## Step 4 — withdraw, if you must

    DELETE /v1/produto/{sku}    (apagaUmProdutoPendente)

**This only reaches a product that is still PENDING.** There is no documented delete, delist or
unpublish operation for a product that has already been approved and published. Deactivating it with
`PUT /v1/produto/status` (value `0` = Inativo) is the nearest available action, and it is a new write,
not a reversal. Confirm with the operator before publishing anything you may need to retract.

## Errors

See `errors/madeiramadeira-problem-types.yml`. There is no machine-readable error code — only the
HTTP status and a Portuguese `errors.detail` string. `400` malformed JSON, `401` missing token,
`403` revoked token, `405` wrong method, `422` duplicate EAN/SKU, `500` server error.
