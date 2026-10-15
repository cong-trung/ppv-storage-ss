# Copilot instructions for File Browser

File Browser (`github.com/filebrowser/filebrowser/v2`) is a Go backend plus a Vue 3 + TypeScript frontend in `frontend/`. The project was **archived on 2026-09-01** and is maintenance-only, so keep changes small, surgical and covered by regression tests. Many recent changes are security fixes, so preserve scope, symlink and rule checks.

## Build, test, lint

Tasks are defined in `Taskfile.yml` ([Taskfile](https://taskfile.dev/)). CI (`.github/workflows/ci.yaml`) uses Go 1.26.x, Node 24 and pnpm 10.

```bash
task build                 # frontend (pnpm install + build) then backend binary
task build:frontend        # frontend/dist only
task build:backend         # go build with version ldflags
task docs:cli:generate     # regenerate docs/cli from the cobra commands (go run . docs)

# Backend
go build ./... && go vet ./...
go test --race ./...                                   # CI command
go test ./http -run TestResourcePostRejectsDanglingSymlinkWriteEscape   # single test
golangci-lint run                                      # config in .golangci.yml (gocritic, govet, revive)

# Frontend (run inside frontend/)
pnpm install --frozen-lockfile
pnpm run dev               # Vite dev server, proxies /api (and the /api/command websocket) to 127.0.0.1:8080
pnpm run build             # runs vue-tsc typecheck, then vite build into frontend/dist
pnpm run typecheck
pnpm run lint              # eslint src/ (CI); pnpm run lint:fix / pnpm run format for Prettier
pnpm run test              # vitest run
pnpm exec vitest run src/utils/__tests__/upload.test.ts        # single file
pnpm exec vitest run -t "test name substring"                  # single test by name
```

`frontend/assets.go` embeds `frontend/dist/*` with `//go:embed` (build tag `!dev`). `go build` still works with an empty `dist/` because it contains `.gitkeep`, but the server has no UI until the frontend is built. For frontend work, run the Go server on :8080 and open the UI through the Vite dev server.

## Architecture

- **Entry/CLI (`cmd/`)**: cobra + viper. `cmd/root.go` starts the server. Every flag can also be set through a config file or an `FB_`-prefixed env var (e.g. `--disable-preview-resize` → `FB_DISABLE_PREVIEW_RESIZE`). Commands get the opened DB through `withViperAndStore` / `withStore` in `cmd/utils.go`. Other subcommands (`config`, `users`, `rules`, `cmds`) edit the same BoltDB.
- **Two settings types (`settings/`)**: `settings.Settings` holds global, DB-persisted settings (auth method, signing key, global rules, branding, defaults, tus), loaded on every request. `settings.Server` holds process/server config (root, base URL, address, `EnableExec`, thumbnails, `FollowExternalSymlinks`, …) and comes from flags or env, also saved through `Storage.SaveServer`.
- **Persistence (`storage/`)**: `storage.Storage` aggregates `Users`, `Share`, `Auth` and `Settings` stores. Each domain package (`users`, `share`, `auth`, `settings`) defines a store or backend interface, and `storage/bolt` implements them with storm/BoltDB.
- **HTTP (`http/`, package `fbhttp`)**: routes are all registered in `NewHandler` in `http/http.go` (gorilla/mux). Each handler has the signature `func(w, r, d *data) (int, error)` and is wrapped by `handle(...)` (the `monkey` helper). That wrapper loads `Settings` on every request, builds `data`, logs errors, and writes the error response when the returned status is non-zero. Return `0, nil` once you have written the response yourself.
  - Wrap handlers in `withUser` (JWT from the `X-Auth` header, or the `auth` cookie on GET; it also canonicalizes the request path), `withAdmin`, or `withPermShare`. Public share routes (`http/public.go`, `withHashFile`) skip `withUser`, so they canonicalize paths and rebase the FS themselves through `data.checkerPrefix`.
  - Authorization runs through `data.Check` / `data.CheckRules`, which apply global rules and then user rules (`rules/`). `Check` also hides dotfiles, while `CheckRules` does not. Call these on every path a handler touches, including paths reached during recursive walks.
- **Filesystem confinement (`files/`)**: each user's `Fs` is an afero FS built by `files.NewFs`. It returns a `ScopedFs` that refuses symlinks escaping the scope, or a bare `BasePathFs` when `FollowExternalSymlinks` is set. `users.Storage.Get(root, followExternal, id)` attaches it. Do file I/O through `d.user.Fs`, never through `os`.
- **Auth providers (`auth/`)**: implementations of the `Auther` interface (`json`, `proxy`, `hook`, `none`). To add one, implement it in `auth/`, register it in the CLI config parser (`cmd/config.go`, `addConfigFlags` for any new flags), and add it to `authBackend.Get` in `storage/bolt/auth.go`. Home-directory creation for signup, proxy and hook provisioning is centralized in `settings.CreateUserHome`, so reuse it instead of duplicating that logic.
- **Uploads**: chunked uploads use tus (`http/tus_handlers.go`) with an in-memory or Redis upload cache (`upload_cache_*.go`). Preview thumbnails go through `img/` and `diskcache/`.
- **Command runner/hooks (`runner/`, `/api/command` websocket)**: disabled by default (`--disable-exec`). It is a known-insecure area that will not be fixed (see `README.md`, `docs/command-execution.md`).
- **Frontend (`frontend/src`)**: Vue 3 with Pinia stores (`stores/`), vue-router, vue-i18n, and API clients in `api/`. Server-side values are injected into `public/index.html` by Go's `html/template` with custom `[{[ ... ]}]` delimiters (`http/static.go`). They are exposed as `window.FileBrowser.*` and re-exported from `src/utils/constants.ts`. A new value must be added in two places: the data map in `http/static.go` and `constants.ts`. The `window.FileBrowser` type in `src/types/global.d.ts` is `any`, so it needs no change.

## Conventions

- Commit messages and PR titles follow Conventional Commits (`fix(scope): …`, `feat: …`, `test(scope): …`, `chore: …`). PR titles are checked in CI. Releases use `commit-and-tag-version` (`task release`), and versions are `v2.x.y`.
- Go imports are grouped stdlib, then third-party, then `github.com/filebrowser/filebrowser/v2/...`. Project-specific errors are in `errors/` (imported as `fberrors`).
- Write regression tests with the existing harnesses rather than new ones. In `http/` tests, `signToken` and `scopedUserStorage` (in `resource_test.go`) and the `customFSUser` wrapper (in `public_test.go`) build a real BoltDB in `t.TempDir()` and call handlers via `handle(...)` with `httptest`. `auth/proxy_test.go` has `mockUserStore`. Symlink tests `t.Skip` when symlinks cannot be created.
- Security fixes: make one commit per fix, reference the advisory in the body (`Refs GHSA-xxxx-xxxx-xxxx`), and add a regression test that checks both the blocked path and the legitimate path. When parallel code paths have diverged, centralize the logic. The full advisory triage, CVSS, write-up and state playbook is in `.claude/CLAUDE.md`.
- Translations: `frontend/src/i18n/en.json` is the source locale. Transifex sync has stopped, so edit locale JSON files directly.
- Documentation is plain Markdown in `docs/`. `docs/cli` is generated, so regenerate it instead of editing it by hand.
