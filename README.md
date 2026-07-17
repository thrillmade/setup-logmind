# setup-logmind

> Install the [logmind](https://github.com/thrillmade/logmind) CLI on a GitHub
> Actions runner.

`logmind` is a decision-logging tool for AI-assisted development. This action
fetches a signed, notarized release binary from the [logmind release
page](https://github.com/thrillmade/logmind/releases) and puts it on the
runner's `$PATH` so subsequent steps can call `logmind` directly.

## Usage

The recommended pattern is a **static pin** to an exact action version, with
`token: ${{ github.token }}` passed explicitly. Let
[Dependabot](#dependabot-keeps-you-current) bump the pin when new versions
ship.

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: thrillmade/setup-logmind@v1.0.2
    with:
      token: ${{ github.token }}
  - run: logmind check-links
```

By default this installs the **latest** logmind release. Pin the CLI version
too if you want fully reproducible CI:

```yaml
- uses: thrillmade/setup-logmind@v1.0.2
  with:
    version: v1.0.2
    token: ${{ github.token }}
```

### Always pass a token

Version resolution and the release download both call GitHub endpoints.
**Unauthenticated** requests to `api.github.com` are capped at **60/hour per
IP**, and GitHub-hosted runner IPs are shared across many concurrent jobs
org-wide — that budget is easy to exhaust from unrelated traffic. When it
happens, this action fails with:

```
curl: (22) The requested URL returned error: 403
##[error]Process completed with exit code 22.
```

on an otherwise-healthy job (see
[#4](https://github.com/thrillmade/setup-logmind/issues/4) and
[#5](https://github.com/thrillmade/setup-logmind/issues/5)). Passing a token
raises the ceiling to 5000/hour and effectively eliminates this failure
mode. `${{ github.token }}` needs no extra setup — it's the job's default
`GITHUB_TOKEN`, already scoped read-only by default.

Composite actions can't default an input to `${{ github.token }}`
themselves — GitHub Actions doesn't evaluate that expression at the point
composite input defaults are resolved — so this can't be made automatic; it
must be passed explicitly as shown above. As a best-effort fallback, the
action also reads a `GH_TOKEN` or `GITHUB_TOKEN` environment variable when
the input is left empty and the calling job happens to export one, but
don't rely on that — pass `token:` explicitly.

As defense in depth, every API call and every asset download also retries
with backoff (3 retries, sleeping 2s/5s/10s) on HTTP 403/429/5xx before
failing, including when a token is present — even the 5000/hour budget is
shared across everything else that token's identity does in that hour.
`token:` is what actually fixes the underlying scarcity; the retry logic
smooths over the remaining transient cases (secondary rate limits, brief
GitHub-side 5xx blips). HTTP 404 is never retried.

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
| `token`   | _(none)_                  | **Recommended:** `${{ github.token }}`. Used as `Authorization: Bearer` on every API call and download. Without it, requests are anonymous and capped at 60/hour per runner IP — shared runner IPs routinely exhaust that budget, causing intermittent `curl: (22) ... 403` failures. See [Always pass `token`](#always-pass-a-token) above. Falls back to a `GH_TOKEN`/`GITHUB_TOKEN` env var if unset. |

## Outputs

| Output    | Description                                                       |
| --------- | ----------------------------------------------------------------- |
| `version` | The resolved logmind version actually installed (e.g. `v1.0.1`). |

Use it downstream like:

```yaml
- id: lm
  uses: thrillmade/setup-logmind@v1.0.2
  with:
    token: ${{ github.token }}
- run: echo "Installed ${{ steps.lm.outputs.version }}"
```

## Other version pinning patterns

<details>
<summary><b>Major float (<code>@v1</code>)</b> — friendlier for first-five-minutes / scratch projects</summary>

```yaml
- uses: thrillmade/setup-logmind@v1
  with:
    version: v1   # or omit to track latest stable
    token: ${{ github.token }}
```

The `v1` tag moves to point at the newest `v1.x.y` release whenever we tag
one. This is convenient for hobby workflows, but the static-pin + Dependabot
pattern above is the supported path for production CI.

</details>

### Pin to a SHA for max security

For workflows that audit every action by SHA (we recommend this for any
workflow with secrets):

```yaml
- uses: thrillmade/setup-logmind@<full-sha-of-v1.0.2-tag>
  with:
    version: v1.0.2
    token: ${{ github.token }}
```

Find the SHA at <https://github.com/thrillmade/setup-logmind/releases/tag/v1.0.2>
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
it downloads the upstream [logmind installer](https://github.com/thrillmade/logmind/blob/main/installer/install.sh)
to a temp file and runs it, so the SHA-256 verification stays in one place.
On Windows it downloads the release zip directly, verifies it against the
same `SHA256SUMS` file, and unpacks `logmind.exe` into the install prefix.

Either way, `<prefix>/bin` is appended to `$GITHUB_PATH` so subsequent steps
in the same job can call `logmind` without qualifying the path.

Every network call (version-resolution API calls and both installer/asset
downloads) goes through a small retry-with-backoff wrapper: up to 3 retries
(4 attempts total), sleeping 2s/5s/10s, on HTTP 403/429/5xx. HTTP 404 fails
immediately without retrying. See [Always pass `token`](#always-pass-a-token)
above for why this exists and why `token:` is still the primary fix.

## Releases

- **`v1.0.2`** — fix: retry with backoff on rate-limited API and download
  calls, so a transient anonymous-quota 403 no longer kills the whole step.
  Every `api.github.com` call and every asset download (installer script,
  release zip, `SHA256SUMS`) now retries up to 3 times (2s/5s/10s backoff)
  on HTTP 403/429/5xx; HTTP 404 still fails immediately. The `token` input
  is now sent as `Authorization: Bearer` on downloads too, not just API
  calls, and falls back to a `GH_TOKEN`/`GITHUB_TOKEN` environment variable
  when the input is empty. README now recommends `token: ${{ github.token }}`
  in every usage example — see [Always pass `token`](#always-pass-a-token).
  Fixes [#4](https://github.com/thrillmade/setup-logmind/issues/4) and
  [#5](https://github.com/thrillmade/setup-logmind/issues/5).
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
