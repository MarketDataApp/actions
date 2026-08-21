# MarketData Shared GitHub Actions

Reusable GitHub Actions for the [Market Data](https://www.marketdata.app/) repositories.

This repository is **public because it has to be**: a workflow in a public repository
cannot read an action out of a private one, since the built-in `GITHUB_TOKEN` is scoped
to the repository it runs in. The SDK repositories are public, so this one is too.
Nothing here is sensitive.

## Actions

| Action | What it does |
|---|---|
| [`validate-context7`](#validate-context7) | Fails the build when a `context7.json` would be rejected by Context7's schema |

---

## `validate-context7`

### Why it exists

Context7 drops a config that fails its schema **silently**. The library still indexes, the
repository shows nothing, and the settings you wrote — `rules`, `excludeFolders` — are
simply not applied. The only trace is a line in Context7's own indexing log, which nobody
reads on a normal day.

That is not a hypothetical. A `rules` entry over the 255-character limit shipped to
`sdk-csharp` and was only noticed when someone happened to watch an indexing run.

### Usage

```yaml
- uses: MarketDataApp/actions/validate-context7@v1
```

With inputs:

```yaml
- uses: MarketDataApp/actions/validate-context7@v1
  with:
    path: context7.json        # default
    fail-if-missing: 'false'   # default
```

The action needs `actions/checkout` to have run, and `python3` — present on all
GitHub-hosted runners.

### Inputs

| Input | Default | Description |
|---|---|---|
| `path` | `context7.json` | File to validate, relative to the workspace |
| `fail-if-missing` | `false` | Fail when the file is absent. Off by default so the action is safe to add to a repository *before* it adopts a config |

### What it checks

Transcribed from [the published schema](https://context7.com/schema/context7.json):

- unknown properties — the schema sets `additionalProperties: false`
- `rules` entries over **255 characters**, and more than 50 of them
- `projectTitle` over 100, `description` over 200 or under 10
- `folders` / `excludeFolders` / `excludeFiles` entry lengths and item counts
- `excludeFiles` entries containing a path separator — the schema requires bare filenames
- duplicate entries in any array the schema marks `uniqueItems`
- `url` and `public_key` supplied without each other
- `disallow: true` combined with settings it would suppress

Malformed JSON exits `2` so it reads distinctly from a schema violation.

Errors are emitted as `::error file=...::` annotations, so they appear inline on the
pull request rather than only in the log.

### Why the limits are transcribed, not fetched

Fetching the live schema would make every CI run depend on `context7.com` being reachable,
and would let an upstream change break builds with no commit to point at. The constraints
are copied instead, with a comment in the script saying to re-check them if the schema
moves. Re-run [`test.yml`](.github/workflows/test.yml) after any such update.

### Running it locally

The script is plain Python with no dependencies:

```bash
python3 validate-context7/validate-context7.py path/to/context7.json
```

---

## Versioning

Consume actions at the **`@v1`** moving major tag. It is repointed at each backward-compatible
release, so fixes arrive without edits in consuming repositories.

Immutable tags (`v1.0.0`, `v1.1.0`, …) exist for pinning, and a commit SHA can be pinned
where reproducibility matters more than staying current. Dependabot's `github-actions`
ecosystem tracks whichever form is used.

Breaking changes get a `v2` tag; `v1` keeps working.

## Contributing

Every action is exercised against fixtures on ubuntu, Windows and macOS by
[`test.yml`](.github/workflows/test.yml). Consumers pin a moving tag, so a regression here
breaks other repositories with no warning — a new action, or a change to an existing one,
needs a matching test.

## License

MIT — see [LICENSE](LICENSE).
