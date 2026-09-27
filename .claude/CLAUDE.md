# VectorCore SMSC: project context for Claude

> Generated from a review of `majiddarvishan/vectorcore-smsc` (fork of `svinson1121` / VectorCore Mobile upstream), version `0.4.0b`.

## What this is
A multi-interface **SMSC / IP-SM-GW** written in Go (1.25). It accepts SMS from SMPP, SIP/3GPP ISC (IMS), SIP SIMPLE and Diameter SGd (SMS-in-MME), routes them, and delivers with store-and-forward, retries, expiry and delivery reports. Ships a REST API (Huma/chi, OpenAPI), Prometheus metrics and an embedded React UI (`web/`, built with Vite).

## Build / run / test
- `make` = `make ui` (npm build into `web/dist`) + `make build` (`go build -ldflags "-X main.version=..." -o bin/smsc ./cmd/smsc`)
- Run: `bin/smsc -c config/smsc.yaml` (default `-c` is `config.yaml`, which does NOT exist in repo). `-d` debug, `-v` version.
- Test: `make test` / `go test ./...` (there are fuzz tests in codec packages).
- UI dev: `make dev-ui` (Vite proxies `/api`, `/metrics`, `/health` to :8080).
- Go module path is still `github.com/svinson1121/vectorcore-smsc`; keep imports consistent with it.

## Layout
- `cmd/smsc/main.go`: wiring of everything (store, registry, routing, SMPP, SIP, Diameter, forwarder, retry, sweeper, API, hot reload of peers).
- `internal/codec`: canonical `codec.Message` + per-interface codecs (`smpp`, `sip3gpp`, `sipsimple`, `sgd`, `tpdu` for GSM 03.40 TPDU/DCS).
- `internal/forwarder`: core dispatch (`forwarder.go`), `retry.go` (scheduler, 15s tick), `expiry.go` (sweeper, 60s tick).
- `internal/routing`: fallback routing rule engine + DB loader (hot reload).
- `internal/registry`: IMS registration cache, Sh refresh, S6c cache (TTL from `diameter.s6c_cache_ttl`).
- `internal/diameter`: peer FSM (`peer.go`), transport tcp/sctp, AVP codec, apps: `sh` (UDR), `s6c` (SRI-SR / ALSC / RDSM), `sgd` (TFR/OFR/ALR/RSR server+client).
- `internal/sip`: sipgo server; `isc` (3GPP SMS over IP, REGISTER/NOTIFY from S-CSCF), `simple` (SIP SIMPLE + IMDN).
- `internal/smpp`: PDU codec, link registry, `server` (bind auth w/ bcrypt + single allowed IP), `client` (outbound sessions manager, TLS optional).
- `internal/dr`: delivery report correlator (SMPP deliver_sm receipts, S6c RDSM reporting).
- `internal/store`: `Store` interface + `postgres` (LISTEN/NOTIFY) and `sqlite` (polling) backends, `schema.sql` in each.
- `internal/api`: REST handlers, OpenAPI, UI embedding. `internal/metrics`, `internal/numbering`, `internal/sgdmap`.
- `docs/API.md`, `docs/ROUTING.md`, `docs/BUILD.md`: authoritative behavior docs.

## Message flow (MT/MO)
Ingress -> decode to `codec.Message` -> normalize destination MSISDN (`numbering`) -> persist as `DISPATCHED` -> routing pass -> finalize.
Candidate order (fixed): `0 ims-local` (local IMS reg cache) -> `1 ims-sh` (live Sh UDR) -> `2 sgd-built-in` (S6c lookup -> MME mapping -> SGd OFR) -> `3+` DB fallback rules (only `smpp` / `sipsimple`, ascending priority).
States: `QUEUED, DISPATCHED, WAIT_TIMER, WAIT_EVENT, WAIT_TIMER_EVENT, DELIVERED, FAILED, EXPIRED`.
Retries: first retry +30s, then SF policy schedule or default `[30,300,1800,3600x5]`. Alerts (S6c ALSC / SGd ALR) requeue waiting messages via guarded transitions. Global lifetime `smsc.max_queue_lifetime` (default 168h).

## Conventions
- Logging: `log/slog`. Config: YAML (`internal/config`). DB change events via `store.Subscribe(table)` drive hot reload.
- Add a new egress: implement sender, add case in `deliverSelectedRoute`, error classification in `classifyRouteError`, and allow it in `fallbackDecisions` + API validation.

## Known gotchas (see FINDINGS.fa.md for details)
- REST API/UI have **no authentication** and bind to `::` by default.
- `make install` copies `config.yaml` which does not exist.
- `bin/smsc` (≈34MB) and `test-certs/` private keys are committed.
- `/api/v1/status` counts omit `WAIT_*` states.
- Only the first enabled Sh and first enabled S6c peer are used (no failover).
