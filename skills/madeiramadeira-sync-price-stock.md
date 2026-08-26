---
name: madeiramadeira-sync-price-stock
description: >-
  Keep price, stock, status, shipping tables and expedition lead times in sync on the MadeiraMadeira
  marketplace, using the bulk endpoints of the seller Marketplace API.
api: madeiramadeira:marketplace
generated: '2026-08-25'
method: generated
source: >-
  Grounded in openapi/madeiramadeira-marketplace-openapi.yml (derived from MadeiraMadeira's own
  published Postman collection). Every operationId below was verified against that spec.
operations:
  - consultarPrecoDeSku
  - atualizaPrecoDeUmOuVariosSkus
  - consultaEstoqueComLimitEOffset
  - atualizaEstoqueDeUmOuVariosSkus
  - consultaStatusComLimitEOffset
  - atualizaOStatusDeUmOuVariosSkus
  - atualizacaoEmMassaDePrecoEstoqueStatusETabelaDeFreteDimensoesDeFrete
  - atualizaPrecoEstoqueEFreteDeProdutosPendentes
  - informaTabelaDeFretePadraoParaTodosOsSkus
  - informarTabelaDeFretePorSku
  - consultaTabelaDeFreteDoSellerComLimitEOffset
  - alterarPrazoDeExpedicaoDeUmOuMaisProdutos
  - atualizacaoDoPrazoDeExpedicaoDefault
---

# Sync price, stock and availability on MadeiraMadeira

Base URL `https://marketplace.madeiramadeira.com.br`. Header `TOKENMM: <seller token>`.

## Safety rules for this skill

Every operation here is a **last-write-wins overwrite with no rollback**. There is no versioning, the
API does not return the prior value, and there is no idempotency key. Concretely:

- **Read before you write.** Use the query operations below to capture current state, so you can
  restore it yourself if a sync goes wrong. The API will not do it for you.
- **Never blind-retry.** A timed-out `PUT` may or may not have landed. Re-read, then decide.
- A price error here is a real commercial event on a live storefront. Confirm large or unexpected
  deltas with the operator before firing.

`limit`/`offset` are PATH segments, not query parameters.

## Read current state

    GET /v1/produto/preco/{sku}                              (consultarPrecoDeSku)
    GET /v1/produto/estoque/limit={limit}&offset={offset}    (consultaEstoqueComLimitEOffset)
    GET /v1/produto/status/limit={limit}&offset={offset}     (consultaStatusComLimitEOffset)
    GET /v1/frete/limit={limit}&offset={offset}              (consultaTabelaDeFreteDoSellerComLimitEOffset)

## Targeted updates

    PUT /v1/produto/preco     (atualizaPrecoDeUmOuVariosSkus)
    PUT /v1/produto/estoque   (atualizaEstoqueDeUmOuVariosSkus)
    PUT /v1/produto/status    (atualizaOStatusDeUmOuVariosSkus)

Status values are `1` Ativo and `0` Inativo. Setting `0` is the closest thing this API has to
unpublishing a live product — there is no delist operation — but note it is a new write, not a
reversal, and it does not remove the product.

`preco_por` has a published floor of R$ 30.00.

## Bulk update

    PUT /v1/produto/bulk    (atualizacaoEmMassaDePrecoEstoqueStatusETabelaDeFreteDimensoesDeFrete)

One call covers price, stock, status, shipping table and shipping dimensions. This is the efficient
path for a full catalog sync. No per-item result contract is documented, so a partial failure is not
visible in the response — verify a sample with the read operations afterwards.

For items still in the pending queue, use the pending-specific variant:

    PUT /v1/produto/pendente/bulk    (atualizaPrecoEstoqueEFreteDeProdutosPendentes)

## Shipping tables

    PUT /v1/produto/frete        (informaTabelaDeFretePadraoParaTodosOsSkus)
    PUT /v1/produto/frete/sku    (informarTabelaDeFretePorSku)

A seller may keep several named tables and choose per SKU which applies. MadeiraMadeira requires a
table named **"Contingencia"** to exist before a seller's store can be released.

Alternatively the seller can quote shipping on its own platform via the `FRETE` callback — see
`madeiramadeira-manage-callbacks`, and note MadeiraMadeira imposes 1500 ms maximum response time and
85% minimum availability on that endpoint.

## Expedition lead time

    PUT /v1/produto/prazo-expedicao        (alterarPrazoDeExpedicaoDeUmOuMaisProdutos)
    PUT /v1/seller/prazo_default/{dias}    (atualizacaoDoPrazoDeExpedicaoDefault)

The second sets the seller-wide default in days and affects the whole catalog. Treat it as a
high-blast-radius write.

## Rate limits

No numeric limit, no header and no `429` are published. MadeiraMadeira states only that abusive usage
may cause requests to be "temporarily blocked for a short period", and that per-request record caps
exist without stating them. Pace bulk syncs conservatively and back off on any unexpected failure —
you have no runtime signal to work from. See `rate-limits/madeiramadeira-rate-limits.yml`.
