# Aria Package & Module System Design

## Design Principles

1. **Simplicity for humans** — No dependency hell. No 800MB `node_modules`. No security nightmares.
2. **Token efficiency for AI** — Short imports, no path verbosity, no aliasing gymnastics.
3. **Decoupled from filesystem** — Module names are declared in code, not dictated by folder structure.
4. **Reproducible builds** — Lock files, vendoring, and pinned versions by default.
5. **One way to do things** — No ambiguity about where code comes from or how to import it.

---

## The Module System

### Module Declaration

Every Aria source file declares which module it belongs to:

```rust
// File: src/auth/handlers/login.aria
mod auth

fn login(credentials: Credentials) -> Session ! AuthError {
    ...
}
```

The `mod auth` declaration **names the module**. The filesystem path (`src/auth/handlers/`) is purely organizational. The compiler doesn't care where the file lives — it cares what module it declares.

Multiple files can declare `mod auth`. They're all part of the same module:

```rust
// File: src/auth/handlers/login.aria
mod auth

fn login(...) -> Session ! AuthError { ... }

// File: src/auth/handlers/register.aria
mod auth

fn register(...) -> User ! AuthError { ... }

// File: src/auth/middleware.aria
mod auth

fn requireAuth(req: net.Request) -> User ! AuthError { ... }
```

All three files contribute to the `auth` module. One import gets you everything:

```
use auth

session := auth.login(creds)?
user := auth.register(info)?
```

### Why This Design

| Concern | Go's Model | Node.js Model | Aria's Model |
|---|---|---|---|
| Rename a folder | All imports break | All requires break | **Nothing breaks** |
| Import verbosity | `"github.com/x/y/internal/auth/handlers"` | `"../../auth/handlers"` | `use auth` |
| Multiple files, one module | Must be same directory | Each file is its own module | **Any files, any location** |
| Discoverability | Follow the path | Good luck | Convention + `aria where auth` |

### Convention: Folder Names Should Match Module Names

The compiler doesn't enforce that `mod auth` lives in a folder called `auth/`. But **by convention**, it should. `aria lint` will warn if they diverge without reason.

This gives humans the "look at the folder, know the module" experience they expect, while allowing reorganization without breakage.

---

## Project Structure

### The Manifest: `aria.toml`

Every Aria project has one file at the root: `aria.toml`. That's it. No `go.mod` + `go.sum` + `Makefile`. No `package.json` + `package-lock.json` + `.npmrc` + `tsconfig.json` + `webpack.config.js`.

```toml
[project]
name = "myapp"
version = "0.1.0"
aria = "0.1"              # minimum Aria compiler version
entry = "src/main.aria"   # entry point (default: src/main.aria)

[deps]
http = "1.2"              # short name, semver
postgres = "0.8"          # driver for db module
redis = "1.0"

[dev-deps]
mock = "0.3"              # only available in tests

[build]
target = "linux-amd64"    # default build target (optional)
optimize = "release"      # default optimization level (optional)
```

**Design decisions:**
- **TOML, not JSON/YAML** — TOML is unambiguous, simple, and generates with very few tokens. JSON lacks comments. YAML has parsing ambiguities.
- **Short dependency names** — `http = "1.2"`, not `"github.com/someorg/aria-http" = "v1.2.3"`. The registry resolves short names.
- **Separate dev-deps** — Test-only dependencies don't ship in production binaries. Period.
- **Entry point is explicit** — No guessing. No `main.go` convention that silently breaks if you rename it.

### Directory Layout

```
myapp/
├── aria.toml          # manifest
├── aria.lock          # lock file (auto-generated, committed to git)
├── src/
│   ├── main.aria      # entry point
│   ├── auth/
│   │   ├── login.aria
│   │   ├── register.aria
│   │   └── middleware.aria
│   ├── db/
│   │   ├── users.aria
│   │   └── migrations.aria
│   └── api/
│       ├── routes.aria
│       └── handlers.aria
├── test/              # integration tests (unit tests are inline)
│   └── api_test.aria
└── vendor/            # vendored dependencies (optional, `aria vendor`)
```

**Why this layout:**
- `src/` contains all source code. No ambiguity about what's source and what's config.
- Tests: **unit tests are inline** (in the same file as the code, using `test` blocks). Integration tests that span modules go in `test/`.
- `vendor/` is optional. When present, the compiler uses it instead of fetching.

---

## The Import System: `use`

### Importing Local Modules

```rust
// Import a module
use auth

// Import multiple modules on one line
use auth, db, api

// Import specific symbols from a module
use auth.{login, register}

// Alias (rarely needed, for conflicts)
use auth
use external_auth as oauth
```

### How the Compiler Resolves `use auth`

Resolution order is simple and deterministic:

1. **Tier 0 builtins** — `Option`, `Result`, `Map`, `Set`, `print`, etc. Always available, no import.
2. **Local modules** — Any `mod auth` declaration in the project's `src/` directory.
3. **Dependencies** — Packages listed in `aria.toml` under `[deps]`.
4. **Tier 1 stdlib** — `io`, `net`, `json`, `db`, `time`, `math`, `crypto`, `os`, `log`, `sync`, `test`.

**Local always wins.** If you have a local `mod json`, it shadows the stdlib `json`. This is intentional — it means you can wrap or replace stdlib modules without changing your import lines.

If there's ambiguity between a dependency and a local module (both named `auth`), the compiler errors and asks you to alias one:

```
error: module name 'auth' is ambiguous
  -> local module 'auth' (src/auth/login.aria)
  -> dependency 'auth' (v1.2.0)

hint: alias one of them:
  use auth                  // keeps local
  use dep.auth as extAuth   // aliases the dependency
```

**Why this matters for AI code generation:** The AI never has to guess whether an import is local, stdlib, or external. Write `use auth` and the compiler reports if it's wrong. Zero ambiguity = zero import bugs.

---

## Dependency Management

### The Registry: `aria.pkg`

Aria uses a **centralized package registry** (like crates.io, unlike Go's decentralized Git approach).

**Why centralized, not Git-based:**
- **Short names**: `http = "1.2"` vs `"github.com/aria-lang/http" = "v1.2.0"`
- **Discoverability**: One place to search, with verified packages
- **Security**: Registry can enforce signing, audit, and malware scanning
- **Simplicity**: `aria add http` — done

**Why not npm's model:**
- **No nested dependencies** — Dependencies are flat. If A depends on `json v1.2` and B depends on `json v1.3`, the compiler resolves to one version (highest compatible) or errors. No `node_modules` tree.
- **No arbitrary install scripts** — Packages are source code only. No `postinstall` scripts. No arbitrary code execution on `aria get`.
- **Maximum dependency count warning** — `aria check` warns if total transitive dependencies exceed a configurable threshold (default: 50). This is a social pressure mechanism against dependency bloat.

### Adding Dependencies

```bash
# Add a dependency
aria add http

# Add a specific version
aria add http@1.2

# Add a dev dependency
aria add --dev mock

# Remove a dependency
aria remove http

# Update all dependencies to latest compatible
aria update

# Update a specific dependency
aria update http
```

### Version Resolution

Aria uses **semantic versioning** with a minimal version selection strategy (like Go, not npm):

- `http = "1.2"` means `>=1.2.0, <2.0.0`
- `http = "1.2.3"` means exactly `1.2.3`
- `http = ">=1.2, <1.5"` for range constraints (rarely needed)

**Minimal version selection** means: use the **oldest** version that satisfies all constraints, not the newest. This makes builds reproducible without a lock file (though Aria still generates one for extra safety).

### The Lock File: `aria.lock`

Auto-generated by `aria add`, `aria update`, or `aria build` (on first run). **Committed to git.** This ensures every developer and CI system builds with identical dependency versions.

```toml
# aria.lock — auto-generated, do not edit

[lock]
aria = "0.1.0"

[[package]]
name = "http"
version = "1.2.4"
checksum = "sha256:a1b2c3d4e5f6..."
source = "registry"

[[package]]
name = "postgres"
version = "0.8.1"
checksum = "sha256:f6e5d4c3b2a1..."
source = "registry"
deps = ["net", "crypto"]
```

### Vendoring

```bash
# Copy all dependencies into vendor/
aria vendor

# Build using vendored dependencies (no network)
aria build --vendor
```

When `vendor/` exists and `--vendor` is passed, the compiler uses local copies exclusively. No network access during build. This is critical for:
- Air-gapped environments
- Reproducibility guarantees
- Auditing dependency source code

---

## Visibility & Access Control

### The Problem with Go's Approach

Go uses capitalization: `Exported` vs `unexported`. This is clever but causes real problems for AI generation:
- Casing must be exactly right on every single identifier
- A typo in casing silently changes visibility instead of being a syntax error
- There's no way to export to "just this package's friends" vs "everyone"

### Aria's Approach: Explicit, Keyword-Based

```rust
// Public — accessible from outside the module
pub fn login(creds: Credentials) -> Session ! AuthError { ... }

// Private — accessible only within this module (DEFAULT)
fn hashPassword(pw: str) -> str { ... }

// Module-internal — accessible to specified modules only
pub(api, test) fn createTestUser() -> User { ... }
```

**Rules:**
- **Everything is private by default.** If you don't say `pub`, it's module-internal.
- `pub` makes it public to all importers.
- `pub(mod1, mod2)` makes it accessible only to named modules. Great for "friend" access patterns like test utilities.
- Types follow the same rules:

```rust
// Public type
pub type User {
    pub name: str       // public field
    pub email: str      // public field
    passwordHash: str   // private field — not accessible outside `auth`
}
```

**Why this is better for AI generation:**
- Write `pub` or don't. No casing games.
- The compiler catches visibility violations. If an internal function is accidentally exposed, `aria check` reports it.
- `pub(test)` means test utilities can be generated that are accessible to test code but invisible to production consumers. No more `// exported for testing` hacks.

---

## Submodules

For larger projects, modules can have submodules:

```rust
// File: src/auth/oauth/google.aria
mod auth.oauth

pub fn googleLogin(token: str) -> Session ! AuthError { ... }
```

Importing:
```rust
// Import the submodule
use auth.oauth

session := auth.oauth.googleLogin(token)?

// Or import specific symbols
use auth.oauth.{googleLogin}

session := googleLogin(token)?
```

Submodules are part of their parent module's namespace. `auth.oauth` can access `auth`'s private members (it's a child). But `auth` cannot access `auth.oauth`'s private members (parent doesn't see into children by default).

This creates a clean hierarchy without filesystem coupling.

---

## Circular Dependency Prevention

The compiler **rejects circular module dependencies at compile time:**

```
error: circular dependency detected
  auth -> db -> auth

hint: extract shared types into a new module, e.g. 'types' or 'models'
```

This is non-negotiable. Circular dependencies create:
- Compilation ordering problems
- Reasoning complexity (for both AI and humans)
- Tightly coupled code that's hard to modify

The hint is important — the compiler doesn't just complain, it **suggests a fix**. This helps when generated code accidentally creates a cycle.

---

## Workspace Support (Multi-Project)

For monorepos or multi-binary projects:

```toml
# aria.toml at workspace root
[workspace]
members = [
    "cmd/server",
    "cmd/cli",
    "lib/core",
    "lib/auth",
]

[workspace.deps]
# shared dependency versions across all members
http = "1.2"
postgres = "0.8"
```

Each member has its own `aria.toml` but inherits workspace-level dependency versions. This prevents the "different versions of the same library in different packages" problem.

---

## CLI Summary

```bash
# Project management
aria init                    # create new project (interactive)
aria init --lib              # create library project
aria init --workspace        # create workspace

# Dependencies
aria add <package>           # add dependency
aria add <package>@<version> # add specific version
aria add --dev <package>     # add dev dependency
aria remove <package>        # remove dependency
aria update                  # update all to latest compatible
aria vendor                  # copy deps into vendor/

# Building
aria build                   # debug build
aria build --release         # optimized build
aria run                     # build + run
aria test                    # run all tests
aria check                   # type-check only (fastest)
aria lint                    # lint (including module naming conventions)

# Information
aria where <module>          # show which files define a module
aria deps                    # show dependency tree
aria deps --why <package>    # show why a transitive dep is included
aria outdated                # show available updates
```

---

## Design Rules Summary

| Rule | Rationale |
|---|---|
| Module name is declared in code, not by filesystem | Reorganize freely without breaking imports |
| Everything is private by default | Explicit `pub` prevents accidental exposure |
| Flat dependency tree | No nested version hell (npm's nightmare) |
| No install scripts | No arbitrary code execution on dependency fetch |
| Minimal version selection | Reproducible without lock file; lock file adds belt-and-suspenders |
| Lock file committed to git | Every build is identical, every time |
| Dependency count warning | Social pressure against bloat |
| Circular dependencies are compile errors | Prevents tangled codebases |
| `use` resolves in fixed order | Local → deps → stdlib. No ambiguity. |
| Short names everywhere | `use auth`, not `import "github.com/org/project/internal/auth"` |

---

## Token Impact Analysis

For a typical multi-module project:

| Component | Go Tokens | Node.js Tokens | Aria Tokens | Aria Savings vs Go |
|---|---|---|---|---|
| Import block (per file) | ~30-60 | ~40-80 | ~5-15 | 75% |
| Module declaration | ~5 | ~0 (implicit) | ~3 | 40% |
| Visibility annotations | ~0 (casing) | ~5-10 (export) | ~3-5 (pub) | N/A (but fewer bugs) |
| Dependency config | ~20-40 (go.mod) | ~100+ (package.json) | ~10-20 (aria.toml) | 50% |
| **Per-file overhead** | **~55-105** | **~145-190** | **~21-43** | **~60-70%** |

Across a 50-file project, that's roughly **2,000-3,000 fewer tokens** spent on project structure alone — tokens that can instead be spent on actual logic.