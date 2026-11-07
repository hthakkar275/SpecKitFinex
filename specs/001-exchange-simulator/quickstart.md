# Quickstart and Validation Guide

This is a run guide for the future implementation described in [plan.md](plan.md), not a claim
that the service or commands already exist. This planning command creates no application source,
Maven build, wrapper, or test suites. After `$speckit-tasks` and implementation, use these scenarios
to validate the [OpenAPI contract](contracts/openapi.json) and [behavioral contract](contracts/api.md).

## Prerequisites and Build

- Select a JDK 21 installation; the current workspace's default Java is 11. `java -version` and
  `./mvnw -version` must both report Java 21 before building. Do not build using the current default.
- The implementation provides Maven Wrapper 3.9.11, the Boot 4.1.0 parent, and an executable JAR.
  Initial wrapper/dependency resolution requires Maven Central access. No database/container is needed.
- Install `curl` and `jq` for manual examples. Run shell examples from the repository root in Bash.
- Use synthetic data and a fresh process for the walkthrough so generated IDs and book state are known.

```bash
java -version
./mvnw -version
./mvnw clean verify
```

Expected: unit, contract, concurrency, and integration checks pass under JDK 21; package
`target/exchange-simulator-0.1.0-SNAPSHOT.jar`. The implementation configures ordinary verification
to exclude performance tests; `-Pperformance` enables the separate capacity runner.

## Start a Local In-Memory Instance

Choose separate administrator and Broker passwords of 16–64 ASCII characters. The reads below
prompt without echoing secrets; do not store real credentials in checked-in fixtures.

```bash
export FINEX_ADMIN_USERNAME=admin
read -r -s FINEX_ADMIN_PASSWORD
export FINEX_ADMIN_PASSWORD
java -Xms512m -Xmx2560m -jar target/exchange-simulator-0.1.0-SNAPSHOT.jar \
  --server.address=127.0.0.1 --server.port=8080
```

Use another Bash terminal for the remaining examples, setting the same administrator environment
variables there. Management metrics use a separate loopback-only port 8081; no database settings.
Default mode must fail startup without the required administrator credentials. Network deployment
requires TLS; the loopback walkthrough is the sole plaintext example.

## Liveness and Administrator Reference Data

```bash
curl -sS -i http://127.0.0.1:8080/status
```

Expected HTTP 200, `X-Correlation-ID`, JSON Content-Type, and all envelope messages plus
`success=true`, `responseCode=0`, `data.running=true`. There is no request body, Content-Type,
Accept, or Authorization header.

```bash
export FINEX_ADMIN_USERNAME=admin
read -r -s FINEX_ADMIN_PASSWORD
export FINEX_ADMIN_PASSWORD
INTERNAL=http://127.0.0.1:8080/internal
EXTERNAL=http://127.0.0.1:8080
PRODUCT_ID=$(curl -sS -u "$FINEX_ADMIN_USERNAME:$FINEX_ADMIN_PASSWORD" \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: demo-product-1' \
  -d '{"type":"EQUITY","symbol":"ABC","description":"Synthetic Equity","tickSize":"0.01"}' \
  "$INTERNAL/product/equity" | jq -er 'select(.success and .responseCode == 1001) | .data.id')
curl -sS -u "$FINEX_ADMIN_USERNAME:$FINEX_ADMIN_PASSWORD" \
  "$INTERNAL/product/equity?symbol=ABC" | jq .
```

Expected: create HTTP 201/code 1001; bodyless lookup HTTP 200/code 1003 with the same ID and
string tick `"0.01"`. A repeated create with the same key/body returns the same original ID/code.
A different payload using that key returns HTTP 200 with `success=false`, code 1002 and IDEMPOTENCY_CONFLICT.

Create buyer and seller Brokers. The command body is generated through `jq`, not shell JSON
interpolation. Retain their generated IDs for ownership checks; submitted orders need only productId.

```bash
read -r -s BROKER_PASSWORD
export BROKER_PASSWORD
BUYER_ID=$(jq -n --arg password "$BROKER_PASSWORD" \
  '{type:"BROKER",name:"Synthetic Buyer",username:"buyer",password:$password}' | \
  curl -sS -u "$FINEX_ADMIN_USERNAME:$FINEX_ADMIN_PASSWORD" \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: demo-buyer-1' \
  --data-binary @- "$INTERNAL/participant/broker" | \
  jq -er 'select(.success and .responseCode == 1007) | .data.id')
SELLER_ID=$(jq -n --arg password "$BROKER_PASSWORD" \
  '{type:"BROKER",name:"Synthetic Seller",username:"seller",password:$password}' | \
  curl -sS -u "$FINEX_ADMIN_USERNAME:$FINEX_ADMIN_PASSWORD" \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: demo-seller-1' \
  --data-binary @- "$INTERNAL/participant/broker" | \
  jq -er 'select(.success and .responseCode == 1007) | .data.id')
```

Expected: each Broker creation is HTTP 201/code 1007, returns username but no password/hash.
The participant cannot GET the internal product or participant catalog (403/999999/FORBIDDEN).
Its external product lookup is permitted and requires no body or media headers.

## Match, Inspect, and Amend

```bash
SELL_RESPONSE=$(jq -n --arg productId "$PRODUCT_ID" \
  '{productId:$productId,side:"SELL",type:"LIMIT",quantity:10,price:"100.005"}' | \
  curl -sS -u "seller:$BROKER_PASSWORD" -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: demo-sell-1' --data-binary @- "$EXTERNAL/order")
SELL_ID=$(jq -er '.data.id' <<< "$SELL_RESPONSE")
BUY_RESPONSE=$(jq -n --arg productId "$PRODUCT_ID" \
  '{productId:$productId,side:"BUY",type:"LIMIT",quantity:4,price:"100.01"}' | \
  curl -sS -u "buyer:$BROKER_PASSWORD" -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: demo-buy-1' --data-binary @- "$EXTERNAL/order")
curl -sS -u "seller:$BROKER_PASSWORD" "$EXTERNAL/order/$SELL_ID" | jq .
curl -sS -u "buyer:$BROKER_PASSWORD" "$EXTERNAL/orderBook/$PRODUCT_ID" | jq .
curl -sS -u "buyer:$BROKER_PASSWORD" "$EXTERNAL/order/trades" | jq .
```

Expected: seller starts BOOKED/version 1, normalized price `"100.01"`, code 1013. Buyer fills
4 at the resting `"100.01"`, code 1018 and status FILLED. Seller becomes PARTIALLY_BOOKED_FILLED,
traded 4/booked 6/version 2. Book shows an offer of 6, no bids, absent spread/midpoint. Trade read
code is 0 and both sides see the same trade. API times match the English/UTC millisecond pattern.

```bash
curl -sS -u "seller:$BROKER_PASSWORD" -X PATCH "$EXTERNAL/order/$SELL_ID" \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: demo-stale-amend-1' \
  -d '{"expectedVersion":1,"quantity":8}' | jq .
AMENDED=$(curl -sS -u "seller:$BROKER_PASSWORD" -X PATCH "$EXTERNAL/order/$SELL_ID" \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: demo-amend-2' \
  -d '{"expectedVersion":2,"quantity":8}')
curl -sS -u "seller:$BROKER_PASSWORD" -X DELETE "$EXTERNAL/order/$SELL_ID" \
  -H 'Idempotency-Key: demo-cancel-1' | jq .
```

Expected: stale amendment 200/1014/ORDER_VERSION_CONFLICT (`success=false`), no state change. Fresh amendment
200/1019, original 10/effective 8/traded 4/booked 4/version 3, retaining queue priority. Cancellation
requires no version/body/media header, returns 200/1017, PARTIALLY_CANCELLED, removed 4, version 4.
Replaying `demo-amend-2` with its original body still returns its original version-3 outcome even
after cancellation; a new cancellation key fails 200/1014/ORDER_ALREADY_CANCELLED (`success=false`). No replay adds
trades, audit mutations, or versions.

## Required Automated Scenarios

Ordinary `./mvnw verify` must cover the following, without waiting for real midnight or taking
performance measurements as unit-test timing assertions:

- All spec stories and all numeric codes; HTTP 200/`success=false` for every handled business
  rejection (including not-found/version/validation/idempotency outcomes), and distinct request/
  security/system HTTP failures; exact envelope and failure reasons across controllers,
  authentication handlers, routing/media/parser failures, and unexpected exceptions.
- FIFO at equal prices/timestamps, multi-level resting-price fills, short market liquidity,
  positive-price tick rounding including 0.05 ticks, and all seven stored statuses.
- Numeric JSON prices rejected, quantity integers enforced, version errors before terminal state,
  wrong owner masked as missing, unsupported entity code 9000, and no stored rejected order.
- Concurrent fill/cancel/amend groups, duplicate keys across products, injected failure at every
  journal mutation, and exact restoration of queues, versions, audit, indexes, and counters.
- Injected clock crosses UTC midnight during staging: expire and restage; reads never display an
  eligible prior-day remainder. Expiry and cancellation still succeed using lifecycle reserves.
- More than 500 order trades via bounded continuation; stale book cursors and stable trade pages.
- Credential-cache cold/expiry behavior, password rotation/deletion during in-flight requests,
  restart invalidation, body/header limits, bounded admission, and no credential logging.

Restart the process after saving an order ID: the old state is gone, the public status remains
code 0, and the old ID is not found after re-registering test identities. This is expected memory
mode behavior, not a durability test failure. Do not advertise restart recovery or multiple replicas.

## Separate Capacity Validation

The implementation supplies a `performance` Maven profile and a seeded integration/load runner;
these are future tasks, not included in this planning output. Run the client from a separate load
host with server allocation 4 CPU/8 GiB and maximum heap 2,560 MiB. Use TLS for a remote target.
Pass credentials through environment variables, not the command line or checked-in configuration.

```bash
./mvnw -Pperformance verify \
  -Dfinex.load.target=https://exchange-test.example \
  -Dfinex.load.seed=20261106 -Dfinex.load.clients=100 \
  -Dfinex.load.durationSeconds=900 -Dfinex.load.mode=baseline
./mvnw -Pperformance verify \
  -Dfinex.load.target=https://exchange-test.example \
  -Dfinex.load.seed=20261106 -Dfinex.load.clients=1000 \
  -Dfinex.load.durationSeconds=60 -Dfinex.load.offeredRate=2000 \
  -Dfinex.load.mode=overload
```

Replace the example host with the test service; use a disposable fresh instance for each dataset.
The runner provisions 10 products/100 Brokers/10,000 open orders/100 populated levels per product,
uses the exact spec request mix and 50% hot-product skew, and maintains valid targets/liquidity.
Warm up for 120 seconds, reporting warm-up separately; run cold authentication and staggered-cache
expiry tests separately. Measure completed valid business requests, not just offered traffic or
HTTP responses. In particular, HTTP 200 with `success=false` is a rejected business
outcome and never contributes to the useful-throughput target. Reports must show at least 1,000 successful business requests/s for 900 seconds,
p95 ≤100 ms, p99 ≤250 ms, unexpected errors <0.1%, and peak RSS ≤4 GiB. Baseline offered load
must have headroom so separately reported expected rejections do not inflate or mask that target.
The burst must preserve invariants, keep response p99 ≤2 seconds, return explicit overload codes,
and regain baseline within 30 seconds. Include timeouts, rejections, auth verification load,
JDK/OS/hardware, memory/GC/JFR, lock waits, and exact fixture schedule in the saved report.

## Completion Evidence

A reviewer needs successful `verify`, the schema/HTTP and reference-model results, race and
failure-injection results, and baseline/overload reports. The in-memory adapter contract suite
becomes the acceptance baseline for a later persistent implementation. No performance, durability,
or production-readiness claim follows from the planning documents alone.
