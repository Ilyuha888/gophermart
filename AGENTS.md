<!-- AI-log: written by an agent (Claude) from the owner's design and his metrics-alerting rules; the owner decides every rule here. -->

# AGENTS.md

Gophermart, the loyalty points service of the Yandex Practicum "Go advanced" graduation project,
built from the `go-musthave-diploma-tpl` template. One binary: an HTTP service where buyers
register, upload order numbers, see points the accrual system credits, and spend them. The
accrual system is a ready binary from the course. What the service does and every design
decision: `README.md` and the interactive page `docs/index.html`.

Module path is `github.com/Ilyuha888/gophermart`. `go.mod` comes with the first code PR.

## Layout

Directories marked *planned* do not exist yet; each appears with the PR that puts code in it.

- `cmd/gophermart/` — entry point: flags and env, wiring, signal handling. Directory name is
  fixed by CI.
- `cmd/accrual/` — the accrual system binaries from the template, one per OS. Not ours; do
  not edit.
- `internal/handler/` *planned* — chi router, middleware (gzip, request log, auth cookie → JWT
  → user id in the request context) and handlers: decode, map errors to HTTP codes, encode.
- `internal/service/` *planned* — the rules: Luhn check, password hash (bcrypt), JWT, "enough
  points?".
- `internal/storage/` *planned* — SQL over pgx, transactions, goose migrations.
- `internal/accrual/` *planned* — the poller (ticker, worker pool, 429 pauses the whole pool)
  and the accrual HTTP client.
- `docs/index.html` — the interactive design page, served by GitHub Pages from `/docs`.

Layers call one way: handler → service → storage (D1). The poller writes through storage.

Do not create a directory before something goes in it. Git does not track empty
directories, and an empty package is noise.

## Commands

Once `go.mod` exists:

```sh
go build -o cmd/gophermart/gophermart ./cmd/gophermart   # the binary CI tests
go test ./...                                            # tests
go test -cover ./...                                     # coverage, 60% required
go vet ./...                                             # what CI runs, minus the Practicum vettool
gofmt -l -w .                                            # format
```

The service needs `JWT_SECRET` in its environment; it does not start without it (D14).

## CI

Two workflows, both triggered by any pull request and by pushes to `main` or `master`.
`gophermart.yml` can also be started by hand. Neither has a branch-name rule.

- `statictest.yml` — `go vet` with the Practicum `statictest` vettool. Must be green.
- `gophermart.yml` — builds `cmd/gophermart/gophermart` from inside its directory, then
  `gophermarttest` starts that binary itself on `localhost:8080` with a PostgreSQL database
  URI, and starts the accrual binary `cmd/accrual/accrual_linux_amd64` on a random free port.
  The Test step will also set `JWT_SECRET`; that edit lands with the auth code.

After a template update (see `README.md`), re-check that the Test step still sets
`JWT_SECRET`.

## Git

- One branch and one pull request per step of the plan, as a stack: PR 1 targets `master`,
  each next PR targets the previous branch. The course reviewer reviews the top PR.
- Do not merge. The owner merges the whole stack after the course reviewer accepts it, with a
  merge commit, never squash. Fix a pushed lower branch with new commits only.
- Commit subject only, English, imperative.

## Conventions

- Prefer the standard library. Add a dependency in the PR that first needs it.
- Keep `main` thin: parse flags, build dependencies, call `run() error`. `os.Exit` skips
  deferred calls, so it belongs in `main` and nowhere deeper.
- Errors are returned, not panicked, and wrapped with `fmt.Errorf("...: %w", err)`.
- Comments carry a load-bearing why. A comment that restates the code gets deleted.
- Every package keeps its one-line package comment, and every exported identifier has a doc
  comment, as the task requires.
- Table-driven tests next to the code as `*_test.go`. The function is named
  `Test<Type>_<Method>_<Expected>`; each case states its own scenario in the subtest name.
  Test coverage stays at 60% or above, as the task requires.
- Package name equals directory name.
- Points are whole hundredths in `int64` and `BIGINT` (D5); converting to and from JSON numbers
  happens only at the edges: handler and accrual client.

## Working with the owner

The owner is learning Go here and stays the author. The full rules live in the owner's private
go-practicum repo, in AGENTS.md, section "How to work with the learner". In short:

- Structure is his: packages, names, signatures, interfaces, data model, DB schema. Bring 2–3
  options with a recommendation and wait for his pick.
- Explain Go on its own terms, without analogies to other languages.
- No slop: no defensive checks for cases that cannot happen, no one-caller helper layers, no
  comments that restate the code.
- AI-log: a file an agent wrote or substantially shaped gets a one-line comment at the top
  saying what came from AI and what the owner did.
- He pastes course material; agents do not browse the course or paste task text into
  external services.
- Autotest sources stay out of the design; test names, CI arguments and failure output come
  in at verification.
- Nothing is committed or pushed before he has read the diff.
