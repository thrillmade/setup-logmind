# setup-logmind

> Install the [logmind](https://github.com/thrillmade/logmind) CLI on a GitHub
> Actions runner.

`logmind` is a decision-logging tool for AI-assisted development. This action
fetches a signed, notarized release binary from the [logmind release
page](https://github.com/thrillmade/logmind/releases) and puts it on the
runner's `$PATH` so subsequent steps can call `logmind` directly.

## Usage

The recommended pattern is a **static pin** to an exact action version. Let
[Dependabot](#dependabot-keeps-you-current) bump the pin when new versions
ship.

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: thrillmade/setup-logmind@v1.0.1
  - run: logmind check-links
```

By default this installs the **latest** logmind release. Pin the CLI version
too if you want fully reproducible CI:

```yaml
- uses: thrillmade/setup-logmind@v1.0.1
  with:
    version: v1.0.1
```

## Dependabot keeps you current

Add this to `.github/dependabot.yml` in your repo and Dependabot will open a
PR whenever `setup-logmind` (or any other `thrillmade/*` action) cuts a new
release:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      thrillmade-actions:
        patterns:
          - "thrillmade/*"
```

This is the path we recommend over moving major tags. Static pins keep your
workflow files an audit trail of what actually ran.

## Inputs

| Input     | Default                   | Description                                                                                                                          |
| --------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `version` | `latest`                  | logmind version to install. Accepts `latest`, an exact tag (`v1.0.1`), or a major float (`v1`) that resolves to the newest matching. |
| `prefix`  | `${RUNNER_TEMP}/logmind`  | Install prefix. The binary lands at `<prefix>/bin/logmind` and `<prefix>/bin` is appended to `$GITHUB_PATH`.                         |
| `token`   | _(none)_                  | Optional. GitHub token used for release-lookup API calls. Defaults to no auth — logmind is a public repo and anonymous GitHub API requests work fine (60 req/hour per runner IP is well above CI volume). Pass a token only if you need cross-repo rate-limit headroom. |

## Outputs

| Output    | Description                                                       |
| --------- | ----------------------------------------------------------------- |
| `version` | The resolved logmind version actually installed (e.g. `v1.0.1`). |

Use it downstream like:

```yaml
- id: lm
  uses: thrillmade/setup-logmind@v1.0.1
- run: echo "Installed ${{ steps.lm.outputs.version }}"
```

## Other version pinning patterns

<details>
<summary><b>Major float (<code>@v1</code>)</b> — friendlier for first-five-minutes / scratch projects</summary>

```yaml
- uses: thrillmade/setup-logmind@v1
  with:
    version: v1   # or omit to track latest stable
```

The `v1` tag moves to point at the newest `v1.x.y` release whenever we tag
one. This is convenient for hobby workflows, but the static-pin + Dependabot
pattern above is the supported path for production CI.

</details>

### Pin to a SHA for max security

For workflows that audit every action by SHA (we recommend this for any
workflow with secrets):

```yaml
- uses: thrillmade/setup-logmind@<full-sha-of-v1.0.1-tag>
  with:
    version: v1.0.1
```

Find the SHA at <https://github.com/thrillmade/setup-logmind/releases/tag/v1.0.1>
and pin to it. Dependabot natively supports SHA-pinned actions — it'll open
PRs with the SHA bumped and the comment annotated with the version.

## Platform support

| OS / arch       | Supported          |
| --------------- | ------------------ |
| Linux x86_64    | yes                |
| Linux arm64     | yes                |
| macOS arm64     | yes                |
| macOS x86_64    | yes                |
| Windows x86_64  | yes                |

Every release is exercised on `ubuntu-latest`, `macos-latest`, and
`windows-latest` in CI before the tag is published.

## How it works

The action is composite (shell-only — no Docker, no JS). On Linux and macOS
it pipes the upstream [logmind installer](https://github.com/thrillmade/logmind/blob/main/installer/install.sh)
so the SHA-256 verification stays in one place. On Windows it downloads the
release zip directly, verifies it against the same `SHA256SUMS` file, and
unpacks `logmind.exe` into the install prefix.

Either way, `<prefix>/bin` is appended to `$GITHUB_PATH` so subsequent steps
in the same job can call `logmind` without qualifying the path.

## Releases

- **`v1.0.1`** — fix: action no longer requires `GH_TOKEN`. v1.0.0 used `gh api`
  for release lookup, which needed `GH_TOKEN` exposed by the caller. Composite
  actions don't inherit `GH_TOKEN` automatically, so any consumer that didn't
  explicitly pass `with: token:` or `env: GH_TOKEN:` failed with
  `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN
  environment variable`. v1.0.1 switches the API lookups to `curl + jq` against
  the public logmind release endpoint, removing the `gh` dependency entirely.
  The `token` input is still accepted but no longer required.
- **`v1.0.0`** — initial release.

See [Releases](https://github.com/thrillmade/setup-logmind/releases) for the
full history.

## License

MIT. See [LICENSE](./LICENSE).
