---
name: madeiramadeira-fulfill-order
description: >-
  Read new MadeiraMadeira marketplace orders and advance them through the order lifecycle — received,
  invoiced (NF-e), shipped, delivered — using the seller Marketplace API.
api: madeiramadeira:marketplace
generated: '2026-08-25'
method: generated
source: >-
  Grounded in openapi/madeiramadeira-marketplace-openapi.yml (derived from MadeiraMadeira's own
  published Postman collection) and the published "Fluxo de Pedido" documentation. Every operationId
  below was verified against that spec.
operations:
  - novoConsultaListaDePedidosNovos
  - consultaPedidoPorId
  - listaDePedidosPorStatus
  - listaTodosOsPedidosEntreDatas
  - recebidoNotificaQueVariosPedidosForamRecebidos
  - nfEmitidaNotificaQueVariosPedidosTiveramNotaFiscalEmitida
  - enviadoNotificaQueVariosPedidosForamEnviados
  - entregueNotificaQueVariosPedidosForamEntregues
  - atualizarCodigoDeRastreio
  - atualizarDataDeEntrega
---

# Fulfill a MadeiraMadeira marketplace order

Base URL `https://marketplace.madeiramadeira.com.br`. Header `TOKENMM: <seller token>`.

## READ THIS BEFORE ANY WRITE

**Order status on this API is a one-way ratchet.** There is no un-receive, un-invoice, un-ship or
un-deliver operation, and MadeiraMadeira's own documentation is explicit that `CANCELADO` is never
accepted inbound — the marketplace only ever pushes it out to the seller. The published order-flow
diagram labels correction paths as "possible through contact and intermediation of MadeiraMadeira",
which is a human process, not an API call.

There is also no idempotency key. Do not blind-retry a timed-out status transition — re-read the
order with `consultaPedidoPorId` and check `status` first.

Treat every transition in step 3 as irreversible and confirm with the operator before firing one.

## Step 1 — find work

    GET /v1/pedido/new/limit={limit}&offset={offset}          (novoConsultaListaDePedidosNovos)
    GET /v1/pedido/{status}/limit={limit}&offset={offset}     (listaDePedidosPorStatus)
    GET /v1/pedido/from={from}&to={to}&limit={limit}&offset={offset}  (listaTodosOsPedidosEntreDatas)

Remember `limit`/`offset` are path segments, not query parameters.

Better than polling: register the `PEDIDO_NOVO` and `PEDIDO_APROVADO` callbacks — see
`madeiramadeira-manage-callbacks`.

## Step 2 — read the order

    GET /v1/pedido/id/{id}    (consultaPedidoPorId)

The response wraps in `{meta:{count}, data:{...}}` and carries `skus[]` (each with `sku`, `skuseller`,
`quantidade`, `valor_unitario`, `frete`, `id_pedido_item`), `comprador`, `dados_entrega`,
`pagamento[]`, `faturamento[]` (NF-e records) and `envio[]` (shipments).

Note two shape traps: identifiers come back as JSON **strings** (`"id_pedido": "6051652"`), and money
appears as both numbers (`"total": 341.91` on an item) and strings (`"subtotal": "341.91"`) in the
same object. Dates appear as `YYYY-MM-DD HH:MM:SS`, `YYYY-MM-DD` and `DD/MM/YYYY` with no timezone.

## Step 3 — advance the status

Statuses: `1` NOVO (placed, payment not authorised), `3` APROVADO (payment authorised), `2`
PROCESSADO (captured by the store), `6` NF EMITIDA / FATURADO, `7` ENVIADO, `8` ENTREGUE, `4`
CANCELADO (inbound-only from MadeiraMadeira).

Only act on an order once it is APROVADO. Each call takes multiple orders:

    PUT /v1/pedido/received     (recebidoNotificaQueVariosPedidosForamRecebidos)
    PUT /v1/pedido/invoiced     (nfEmitidaNotificaQueVariosPedidosTiveramNotaFiscalEmitida)
    PUT /v1/pedido/shipped      (enviadoNotificaQueVariosPedidosForamEnviados)
    PUT /v1/pedido/delivered    (entregueNotificaQueVariosPedidosForamEntregues)

`invoiced` is not a generic flag — it asserts a Brazilian **Nota Fiscal Eletronica** was issued. The
order's `faturamento[]` block carries the 44-digit NF-e `chave_acesso` and a SEFAZ portal URL. Do not
call it until the NF-e actually exists.

No per-item result array is documented for these bulk calls, so you cannot tell which orders in a
batch succeeded. Keep batches small enough to re-verify individually.

## Step 4 — tracking and delivery detail

    PUT /v1/mensageria/orders/shipping    (atualizarCodigoDeRastreio)
    PUT /v1/mensageria/orders/delivery    (atualizarDataDeEntrega)

These sit on the Mensageria surface and need a **bearer JWT**, not the TOKENMM header. Obtain it with
`POST /v1/mensageria/auth/generate-token` (send MMTOKEN; returns `access_token`, `expires_in` 3600).

## Errors

See `errors/madeiramadeira-problem-types.yml`. `404` on the shipping surface means "we do not serve
that region", not "not found".
