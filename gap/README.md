# yap air-gap dependency bundle

Everything needed to build [OnlpLab/yap](https://github.com/OnlpLab/yap) inside an air gap.

| File | |
|---|---|
| `yap-deps.tar.gz` | the bundle, 84K |
| `yap-deps.tar.gz.sha256` | `da1bb97fdee1787b1e428574705ae1116ea64cfa68b1890271ab1f3ffb16e901` |

---

## Why the documented build fails

yap's README says `bunzip2 data/*.bz2` → `go get .` → `go build .`. On any modern Go you
get:

```
go: go.mod file not found in current directory or any parent directory
```

Two separate problems are hiding behind that one message.

**1. yap is a pre-modules project.** There is no `go.mod` anywhere in the repo, and its
internal imports are bare and non-domain-qualified — `main.go` imports `yap/app` and
`yap/webapi`, and there are 31 such `yap/...` imports. Those only resolve when the tree
sits at `$GOPATH/src/yap`. The repo's own `Dockerfile` confirms the intent: `FROM
golang:1.12-buster`, `ENV GOPATH=/yap`, code at `/yap/src/yap`. Since Go 1.16, module
mode is the default, so `go get` demands a `go.mod` and stops.

**2. The repo's `vendor/` is incomplete.** It ships only `github.com/gonuts/commander`
and `github.com/gonuts/flag`. Two more imports are *not* vendored:

| Package | Version in bundle | Imported by |
|---|---|---|
| `github.com/gorilla/mux` | v1.8.0 | `webapi/webapi.go` |
| `gopkg.in/yaml.v2` | v2.4.0 | `alg/transition/featurereader.go` |

Fetching those two was the only thing `go get .` ever did — which is why the documented
build **cannot** work offline, no matter how you fix the module mode. This bundle
supplies them so you can skip `go get` entirely.

Both are pure-stdlib leaves, so these two are the complete transitive closure.

`mux` is pinned to **v1.8.0, not v1.8.1**, deliberately: v1.8.1 declares `go 1.20`, newer
than the target's Go 1.19.2. v1.8.0 declares `go 1.12`. The API yap uses is identical.

---

## Install (in the gap)

```bash
shasum -a 256 -c yap-deps.tar.gz.sha256      # must print: OK
mkdir -p /tmp/yd && tar -xzf yap-deps.tar.gz -C /tmp/yd
/tmp/yd/install.sh /path/to/your/yap
```

`install.sh` copies the two packages into yap's existing `vendor/` with `cp -Rn`, so it
cannot clobber the `gonuts` packages already there, then verifies all four are present.

Then build:

```bash
export GOPATH=$HOME/go
export GO111MODULE=off
mkdir -p "$GOPATH/src"
cp -R /path/to/your/yap "$GOPATH/src/yap"    # must be exactly $GOPATH/src/yap
cd "$GOPATH/src/yap"
go build .
./yap
```

The path is mandatory, not stylistic — the bare `yap/...` imports resolve nowhere else.
**Never run `go get .`**; its only job is already done. `GO111MODULE=off` is what stops
Go from demanding the `go.mod` that started all this, and Go 1.19.2 supports it fully.

`data/*.bz2` must be un-bzip2'd for *runtime*, but is not needed to build.

### Module-mode fallback

Not needed on Go 1.19.2, but tested and included in case the toolchain changes:

```bash
cp /tmp/yd/fallback-module-mode/go.mod      "$GOPATH/src/yap/go.mod"
cp /tmp/yd/fallback-module-mode/modules.txt "$GOPATH/src/yap/vendor/modules.txt"
cd "$GOPATH/src/yap" && go build -mod=vendor .
```

`module yap` in `go.mod` is what makes the bare `yap/...` imports resolve locally. Use
*both* files — a hand-written `go.mod` with a mismatched `vendor/modules.txt` will fail
the consistency check.

The `gonuts/*` entries carry synthetic versions (`v0.1.0`); those packages predate Go
modules and have no real tag. Harmless — vendor mode does not checksum-verify vendored
code, it only requires `go.mod` and `modules.txt` to agree, which they do.

---

## ⚠️ Do not "fix" the build with stubs

The tempting shortcut — hand-writing stub packages for `yaml.v2` and `mux` until the
compiler is happy — **produces a binary that silently generates garbage.**

`conf/*.yaml` are not config trivia. They are the perceptron **feature templates**
(`jointstandard.yaml` is 13KB of feature definitions). The chain:

1. `LoadFeatureConf` calls `yaml.Unmarshal(conf, setup)` and **discards the error**
   (`alg/transition/featurereader.go:51`). A stub returning `nil` looks like success.
2. Empty `FeatureSetup{}` → `NumFeatures()` returns **0**.
3. `SetupExtractor` builds `util.NewEnumSet(0)` and `LoadFeatureSetup` iterates zero
   groups, registering no features (`app/vars.go:278`).
4. `dep.go:328` allocates `NewAvgMatrixSparse(0, ...)` — a weight matrix with no features.

The model file still loads, nothing errors, and you get confident-looking wrong output.
Affects `md`, `dep`, and `joint` in both CLI and API form (`app/md.go:317`,
`app/dep.go:224`, `app/jointparse.go:304`, `webapi/{md,dep,joint}.go`).

`hebma` is the sole exception — it is lexicon-based off `data/bgulex` and never reads the
YAML, so it keeps working. In a `hebma → md → dep` pipeline, only stage one survives.

A `mux` stub is narrower: its entire use is five lines in `webapi/webapi.go:164-170`
(`NewRouter`, five `HandleFunc`, `ListenAndServe`). A stub still binds :8000, but every
endpoint 404s. CLI-only users are unaffected.

### The 5-second check

Real feature loading logs one line per group. Run `md` or `dep` and watch stderr:

```
Loading feature group Past Morphemes Unigram
Loading MD transition dependent feature group ...
```

**No `Loading feature group` lines at all → the yaml stub is active and your output is
worthless.** Needs no reference data.

### Recovering from stubs

```bash
cd "$GOPATH/src/yap"
rm -rf vendor/gopkg.in/yaml.v2 vendor/github.com/gorilla/mux
mkdir -p /tmp/yd && tar -xzf yap-deps.tar.gz -C /tmp/yd && /tmp/yd/install.sh "$PWD"
```

The `rm -rf` matters — `install.sh` uses `cp -Rn` and will not overwrite stubs. Then
either delete the hand-written `go.mod` and build with `GO111MODULE=off`, or replace
*both* module-mode files as shown above.

---

## Verification status

Built and run against **go1.19.2 darwin/arm64**, fully offline (`env -i`, `GOPROXY=off`,
clean `GOPATH`) from the shipped tarball — not a staging copy:

| Check | Result |
|---|---|
| `shasum -c` on tarball | OK |
| Internal `SHA256SUMS.txt` | 27/27 OK |
| `install.sh` into pristine checkout | exit 0 |
| `go build .` (GOPATH mode) | **exit 0, no diagnostics** |
| `./yap` | prints command list (`api`, `dep`, `hebma`, `joint`, `ma`, `md`) |
| `go build -mod=vendor .` (module mode) | exit 0, binary runs |
| `go list -deps` | all four deps resolve from `<yap>/vendor/` |

`GOPROXY=off` means any network attempt would have been a hard error, not a silent
fallback — so this is a genuine offline result.

**Not verified:** the build test excluded `data/`, so this proves yap *compiles and
starts*, not that Hebrew parsing is end-to-end correct. The first real `hebma`/`md` run in
the gap closes that gap — use the `Loading feature group` check above as the signal.

Platform note: the bundle is **source only**, so it is architecture-independent. It builds
natively on linux/amd64, darwin/arm64, or anything else Go supports — unlike pip wheels or
npm native packages, no per-platform variant is needed.
