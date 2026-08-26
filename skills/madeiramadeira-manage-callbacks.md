---
name: madeiramadeira-manage-callbacks
description: >-
  Register, inspect and remove MadeiraMadeira marketplace webhook callback URLs so a seller receives
  order and product events instead of polling, and can serve live shipping quotes.
api: madeiramadeira:marketplace
generated: '2026-08-25'
method: generated
source: >-
  Grounded in openapi/madeiramadeira-marketplace-openapi.yml and the "Callbacks" and "Integracao de
  Frete" sections of MadeiraMadeira's published documentation. See
  asyncapi/madeiramadeira-marketplace-webhooks.yml.
operations:
  - consultaUrlsDeCallbackCadastradas
  - criaNovaUrlDeCallback
  - deletarUmCallback
---

# Manage MadeiraMadeira webhook callbacks

Base URL `https://marketplace.madeiramadeira.com.br`. Header `TOKENMM: <seller token>`.

Callbacks are how a seller stops polling. Registration is self-service through the API itself.

## Inspect what is registered

    GET /v1/callback    (consultaUrlsDeCallbackCadastradas)

Returns `{meta:{count}, data:[{id_callback, url, tipo, datahora_alteracao}]}`.

**Always run this first.** There is no update operation, and registering a type that already exists
is not a documented no-op.

## Register

    POST /v1/callback    (criaNovaUrlDeCallback)

    {"tipo": "PEDIDO_NOVO", "url": "https://seller.example.com/hooks/mm/order-new"}

Valid `tipo` values:

| tipo | what arrives |
|---|---|
| `PEDIDO_NOVO` | order placed, payment not yet authorised (status 1) |
| `PEDIDO_APROVADO` | payment authorised (status 3) |
| `PEDIDO_CANCELADO` | order cancelled (status 4) — outbound only, never accepted inbound |
| `PRODUTO_APROVADO` | a submitted product was approved and published |
| `FRETE` | synchronous shipping-quote request (see below) |

Order event payload:

    {"id_seller": "225", "order": "2371", "status": 1, "time": 1532567027}

Product event payload:

    {"id_seller": 456789, "sku": "sku123456", "aprovado": 1}

## Change a registration

There is no `PUT`. MadeiraMadeira's published instruction is to **delete and recreate**:

    DELETE /v1/callback/{tipo}    (deletarUmCallback)

then `POST` the new URL. This is fully reversible and is the one part of the API where the undo path
is unambiguous — but there is a gap between the delete and the create during which events are not
delivered. Sequence it during low traffic.

## The FRETE callback is different

`FRETE` is not a notification — it is a **synchronous request MadeiraMadeira makes of the seller**,
and the seller's endpoint must answer it.

Request MadeiraMadeira sends:

    {"destinationZip": "80320120", "volumes": [{"sku": "123456", "quantity": 1}]}

Response the seller must return:

    {"shippingQuotes": [{
       "shippingCost": 10.50,
       "deliveryTime": {"expedition": 2, "transit": 3, "total": 5},
       "shippingEstimatedId": "10203040",
       "shippingMethodId": "transportadora",
       "shippingMethodName": "ENTREGA RAPIDA",
       "shippingMethodDisplayName": "ERAPIDA"
    }]}

Return HTTP `200` on a successful quote and `404` when the seller does not deliver to that region.
Delivery times are in business days and `total` must equal `expedition + transit`.

MadeiraMadeira imposes a hard contract on this endpoint: **1500 ms maximum response time and 85%
minimum availability**, observable in the Seller Dashboard. Register `FRETE` only if the seller's
endpoint can actually meet that — otherwise upload shipping tables instead (see
`madeiramadeira-sync-price-stock`).

## Verifying inbound events

MadeiraMadeira documents **no signature, shared secret, HMAC, timestamp or replay protection** on
these callbacks, and no mutual TLS. A receiving endpoint cannot verify from the payload that an event
came from MadeiraMadeira. Do not act on an order event on the payload's word alone — read the order
back with `GET /v1/pedido/id/{id}` over the authenticated API before doing anything consequential.

No retry policy, delivery guarantee, ordering guarantee or dead-letter behaviour is documented
either, so treat delivery as best-effort and keep a periodic reconciliation poll as a backstop.
