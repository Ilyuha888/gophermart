<!-- AI-log: written by an agent (Claude) from the owner's plan; the owner chose the seven PRs and their order. -->

# Plan: the code stack

The service is built in seven pull requests, stacked: C1 targets `master`, each next PR targets
the previous branch. The course reviewer reviews the top PR; the owner merges the whole stack
with a merge commit after the course accepts it. Design and decisions D1–D14: `README.md` and
the [interactive page](index.html).

After C4 the link to the top PR goes to the course for first feedback; the course asks for a
first link within two weeks.

Every PR must keep these green: `go build ./...`, `go vet ./...`, `go test ./...`, `gofmt -l .`
prints nothing, and the `go vet test` workflow (Practicum statictest).

Branch names: `c01-skeleton`, `c02-storage`, `c03-auth`,
`c04-orders`, `c05-accrual`, `c06-balance`, `c07-finish`.

The "Autotest" line of each PR is an expectation. The autotest sources are not read; the first
CI run of each PR shows the real result.

## C1 skeleton

**Scope:** `go.mod`; config from flags and env: `-a`/`RUN_ADDRESS`, `-d`/`DATABASE_URI`,
`-r`/`ACCRUAL_SYSTEM_ADDRESS`, and `JWT_SECRET` (required); `log/slog` logger; thin `main` calling
`run() error`; an HTTP server on chi with no routes yet; graceful stop on SIGINT/SIGTERM;
`JWT_SECRET` env in the Test step of `.github/workflows/gophermart.yml` (D14); English
`cmd/gophermart/README.md`.

**Acceptance:**
- Without `JWT_SECRET` the binary exits with an error that names the variable.
- An env var wins over its flag, as in metrics-alerting (the task names both, not the order).
- SIGINT stops the server and the process exits 0.
- Tests: config parsing, table-driven: flags only, env over flag, missing `JWT_SECRET`.

**New Go:** none; the parts come from metrics-alerting (D8).

**Autotest:** the service starts, so the workflow gets past startup; the full-flow test is
expected red, as no endpoint exists. This run also shows whether `gophermarttest` passes
`JWT_SECRET` to the service: if the service fails at start, D14's CI step needs another way.

## C2 storage

**Scope:** pgx pool; goose migrations embedded in the binary: `users`, `orders`, `withdrawals`
with the columns and indexes of the Data tab; retry of the first connection, as in
metrics-alerting; a storage type with the queries the next PRs need, added as they need them.

**Acceptance:**
- On start the service applies migrations; a second start applies nothing and does not fail.
- `DATABASE_URI` unreachable at start: retries, then exits with an error.
- Tests run on a real PostgreSQL through `TEST_DATABASE_DSN` and skip without it.
- Money columns are `BIGINT` hundredths (D5); `withdrawals.order_number` is `UNIQUE` (D11).

**New Go:** the pgx native API (`pgxpool.Pool`, `pgx.Rows`, `pgconn.PgError`) instead of
`database/sql`. Embedded migrations are already known from metrics-alerting.

**Autotest:** expected red, as no endpoint exists.

## C3 auth

**Scope:** `POST /api/user/register` and `POST /api/user/login`; bcrypt hash (D7); a JWT with
the user id, valid 24 h, signed with `github.com/golang-jwt/jwt/v5`, in an HttpOnly cookie (D2); auth middleware that checks the cookie and
puts the user id in the request context.

**Acceptance:**
- Register: 200 and a cookie; 400 on bad JSON or an empty login or password; 409 on a taken
  login; 500 on a storage error.
- Login: 200 and a cookie; 400 on a bad request; 401 on a wrong login or password, the same
  answer for both.
- Middleware: no cookie, a bad signature or an expired token → 401; the handler behind it
  reads the user id.
- Tests: handlers with a fake service, middleware with valid, forged and expired tokens,
  storage on a real PostgreSQL.

**New Go:** the JWT library `github.com/golang-jwt/jwt/v5` (D302); bcrypt from `golang.org/x/crypto`. A
context value under a private key type is already known from metrics-alerting
(`signedKey` in its handler); new here is reading a typed value back with a type assertion.

**Autotest:** expected red: register and login answer, but the flow continues to orders.

## C4 orders

**Scope:** `POST /api/user/orders` (plain-text number) and `GET /api/user/orders`; Luhn check;
orders are saved as NEW for the poller of C5.

**Acceptance:**
- Upload: 202 new number; 200 already uploaded by this user; 409 uploaded by another user;
  400 empty body; 422 not digits or Luhn fails; 401 without auth; 500.
- List: 200 with the user's orders, newest first, `uploaded_at` in RFC3339, `accrual` only when
  there are points; 204 when there are none; 401; 500.
- Tests: Luhn, table-driven with valid and invalid numbers; both handlers; the insert that
  tells "mine", "someone else's" and "new" apart, on a real PostgreSQL.

**New Go:** none; Luhn is arithmetic on digits.

**Autotest:** expected red: orders stay NEW until C5 polls the accrual system.

**Then:** the link to the top PR (C4) goes to the course for first feedback.

## C5 accrual

**Scope:** accrual client `GET /api/orders/{number}`; a ticker that takes non-final orders
from the database and hands them to a worker pool (D3); 429 pauses the whole pool for
`Retry-After`; status mapping D10; stop with the service.

**Acceptance:**
- Mapping as on the Data tab: 204 keeps NEW; REGISTERED and PROCESSING → PROCESSING; INVALID
  and PROCESSED are final; PROCESSED saves the accrual.
- 429 with `Retry-After: N`: no worker sends a request for N seconds.
- 500 or a network error: the order stays and is asked again on the next tick.
- On SIGINT the poller stops; requests in flight finish; unfinished orders stay in the
  database for the next start.
- Tests: client against `httptest.Server` answering 200, 204, 429 and 500; the pause shared
  by all workers; mapping, table-driven.

**New Go:** sharing one pause between goroutines: every worker waits until a time that any
worker can move forward.

**Autotest:** expected red: orders get their points, but the flow continues to the balance.

## C6 balance

**Scope:** `GET /api/user/balance`, `POST /api/user/balance/withdraw`,
`GET /api/user/withdrawals`; balance computed from sums (D4); a withdrawal locks the user row.

**Acceptance:**
- Balance: 200 `{current, withdrawn}` as numbers with a fraction; 401; 500.
- Withdraw: 200; 400 on bad JSON or a sum ≤ 0 (D12); 402 when points are short; 422 when the
  number fails Luhn or was used before (D11); 401; 500.
- Withdrawals: 200, newest first, `processed_at` in RFC3339; 204 when there are none; 401; 500.
- Two withdrawals of one user at once cannot spend the same points: a test on a real
  PostgreSQL runs them in parallel and checks the balance never goes below 0.

**New Go:** a transaction with a row lock (`SELECT … FOR UPDATE`) through pgx: begin, commit,
rollback on every error path.

**Autotest:** expected green: every endpoint of the task exists from here. The task says a
client may compress requests; if the autotest does, it stays red until gzip in C7.

## C7 finish

**Scope:** gzip for requests and answers; test coverage at 60% or above; a doc comment on
every exported identifier; optional integration tests against a real database and the accrual
binary.

**Acceptance:**
- A gzipped request is read; an answer is gzipped when the client sends
  `Accept-Encoding: gzip`.
- `go test -coverprofile=cover.out ./... && go tool cover -func=cover.out` shows a total of
  60% or above.
- No exported identifier without a doc comment.

**New Go:** `go test -coverprofile` and `go tool cover`, not used in metrics-alerting. gzip is
ported from it (D8).

**Autotest:** expected green.
