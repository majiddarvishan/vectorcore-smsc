# VectorCore SMSC: project context for Claude

> Generated from a review of `majiddarvishan/vectorcore-smsc` (fork of `svinson1121` / VectorCore Mobile upstream), version `0.4.0b`.
> Deep review with all findings: `.claude/REVIEW.fa.md` (Persian). Overview: `.claude/FINDINGS.fa.md`.

## What this is
A multi-interface **SMSC / IP-SM-GW** written in Go (1.25). It accepts SMS from SMPP, SIP/3GPP ISC (IMS), SIP SIMPLE and Diameter SGd (SMS-in-MME), routes them, and delivers with store-and-forward, retries, expiry and delivery reports. Ships a REST API (Huma/chi, OpenAPI), Prometheus metrics and an embedded React UI (`web/`, built with Vite).

## Build / run / test
- `make` = `make ui` (npm build into `web/dist`) + `make build` (`go build -ldflags "-X main.version=..." -o bin/smsc ./cmd/smsc`)
- Run: `bin/smsc -c config/smsc.yaml` (default `-c` is `config.yaml`, which does NOT exist in repo). `-d` debug, `-v` version.
- Test: `make test` / `go test ./...` (fuzz tests exist in codec packages). Prefer `go test -race ./...`.
- UI dev: `make dev-ui` (Vite proxies `/api`, `/metrics`, `/health` to :8080).
- Go module path is still `github.com/svinson1121/vectorcore-smsc`; keep imports consistent with it.

## Layout
- `cmd/smsc/main.go`: wiring of everything (store, registry, routing, SMPP, SIP, Diameter, forwarder, retry, sweeper, API, hot reload of peers).
- `internal/codec`: canonical `codec.Message` + per-interface codecs (`smpp`, `sip3gpp`, `sipsimple`, `sgd`, `tpdu` for GSM 03.40 TPDU/DCS).
- `internal/forwarder`: core dispatch (`forwarder.go`), `retry.go` (scheduler, 15s tick), `expiry.go` (sweeper, 60s tick).
- `internal/routing`: fallback routing rule engine + DB loader (hot reload).
- `internal/registry`: IMS registration cache, Sh refresh, S6c cache (TTL from `diameter.s6c_cache_ttl`).
- `internal/diameter`: peer FSM (`peer.go`), transport tcp/sctp, AVP codec, apps: `sh` (UDR), `s6c` (SRI-SR / ALSC / RDSM), `sgd` (server+client).
- `internal/sip`: sipgo server; `isc` (3GPP SMS over IP, REGISTER/NOTIFY from S-CSCF), `simple` (SIP SIMPLE + IMDN).
- `internal/smpp`: PDU codec, link registry, `server` (bind auth w/ bcrypt + single allowed IP), `client` (outbound sessions manager, TLS optional).
- `internal/dr`: delivery report correlator (SMPP deliver_sm receipts, S6c RDSM reporting).
- `internal/store`: `Store` interface + `postgres` (LISTEN/NOTIFY) and `sqlite` (polling) backends, `schema.sql` in each.
- `internal/api`: REST handlers, OpenAPI, UI embedding. `internal/metrics`, `internal/numbering`, `internal/sgdmap`.
- `docs/API.md`, `docs/ROUTING.md`, `docs/BUILD.md`: authoritative behavior docs.

## Message flow (MT/MO)
Ingress -> decode to `codec.Message` -> normalize destination MSISDN (`numbering`) -> persist as `DISPATCHED` -> routing pass -> finalize.
Candidate order (fixed): `0 ims-local` -> `1 ims-sh` (live Sh UDR) -> `2 sgd-built-in` (S6c -> MME mapping -> SGd MT-Forward) -> `3+` DB fallback rules (only `smpp` / `sipsimple`, ascending priority).
States: `QUEUED, DISPATCHED, WAIT_TIMER, WAIT_EVENT, WAIT_TIMER_EVENT, DELIVERED, FAILED, EXPIRED`.
Retries: first retry +30s, then SF policy schedule or default `[30,300,1800,3600x5]`. Alerts (S6c ALSC / SGd ALR) requeue waiting messages. Startup resets all `DISPATCHED` to `QUEUED`.

## Naming trap
In code, SGd names are INVERTED vs 3GPP TS 29.338: `SendOFR`/`EncodeOFR`/`buildOFRRequest` actually send **MT-Forward (TFR)**; `DecodeTFR` decodes inbound **MO-Forward (OFR)**. Command codes are correct; only names are swapped.

## Conventions
- Logging: `log/slog`. Config: YAML (`internal/config`). DB change events via `store.Subscribe(table)` drive hot reload.
- Add a new egress: implement sender, add case in `deliverSelectedRoute`, error classification in `classifyRouteError`, and allow it in `fallbackDecisions` + API validation.
- Any change to message status MUST be conditional on the expected current status (see REVIEW ARCH-1).

## Top known issues (see REVIEW.fa.md)
- No auth on REST API, Diameter ingress (CER always accepted) or SIP REGISTER/NOTIFY.
- Unconditional status updates: expiry can overwrite DELIVERED; persist/terminal-update errors ignored.
- SQLite: RFC3339 vs `datetime('now')` text comparison breaks same-day retry/expiry.
- Disabled routing rules still match; registry reload never drops deleted entries.
- SMPP server ACKs `submit_sm` with ROK before decoding; no bind-state enforcement; no bind timeout.
- Diameter control messages bypass the single writer (interleaving risk); End-to-End ID wraps every 256.
- Unsynchronized maps in `main.go` hot reload (panic risk); shutdown does not wait for goroutines.
- `make install` copies non-existent `config.yaml`; `bin/smsc` and `test-certs/*.key` committed.
