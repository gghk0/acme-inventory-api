# Acme Inventory API

Stock level and reservation service for the Acme commerce platform. Owns
authoritative SKU counts and issues time-limited holds so checkout can reserve
stock while payment is authorised.

- **Spec:** [`openapi.yaml`](./openapi.yaml)
- **Internal endpoint:** `https://inventory.internal.acme.example/v1`
- **Owner:** Acme Supply Chain Team

## Dependency map

### Upstream — services that call Inventory

| Caller | Operations consumed | Purpose |
|---|---|---|
| `acme-orders-api` | `POST /v1/reservations` | Hold stock at checkout |
| `acme-orders-api` | `DELETE /v1/reservations/{reservationId}` | Release hold on cancellation |

### Downstream — services Inventory calls

| Dependency | Operation | Purpose |
|---|---|---|
| `acme-warehouse-api` | `POST /v1/stock-sync` | Pull physical counts from the WMS |
| Slack *(external)* | `POST /services/T000/B000/XXXX` | Low-stock threshold alerting |

### Data stores

| Store | Engine | Tables / keys |
|---|---|---|
| `acme_inventory` | PostgreSQL 15 | `skus`, `stock_levels`, `reservations` |
| `reservation:{id}` | Redis | TTL keys, 15-minute expiry |

## Blast radius

- `POST /v1/reservations` is called **synchronously inside `POST /orders`** —
  its p99 latency is on the critical path for every checkout in the platform.
  It is the single biggest contributor to checkout tail latency.
- It is also the most common source of `409` responses in checkout: insufficient
  stock is a normal business outcome, not an error, and `acme-orders-api`
  surfaces it directly to the shopper.
- Reservation expiry depends on Redis TTLs. If the expiry sweeper fails, stock
  leaks as permanently held; the `reservations:reconcile` job (every 10 minutes)
  is the safety net, and if *that* fails the leak becomes permanent.

## Running the Postman CLI here

```bash
postman init
postman workspace create
git add .postman/resources.yaml && git commit -m "link workspace"
```

`postman init` adopts `openapi.yaml` at the repo root automatically. Do **not**
create the `postman/` directory by hand — `init` refuses to run if it already
contains source code.
