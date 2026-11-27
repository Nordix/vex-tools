# vex-tools

VEX (Vulnerability Exploitability eXchange) is a standardised way for a supplier to declare how known vulnerabilities relate to a specific product version (for example not affected, affected, fixed or under investigation). This tool generates CSAF VEX statements saying "these CVEs are fixed in our fork release", based on CVE references found in the release's commits, merged PRs and release notes.

It only emits the `fixed` status. It does not produce `not_affected`, `affected` or `under_investigation` statements.


## Requirements

- `bash` 4 or later, `git`, `jq`, `gh` (GitHub CLI) and `date` (coreutils)
- `gh` authenticated: set `GH_TOKEN`, or run `gh auth login`
- A full clone with tags. In GitHub Actions use `actions/checkout` with `fetch-depth: 0`

## Prerequisites

For the VEX to be complete and accurate:

- The repository is hosted on GitHub (`origin` is a github.com URL). It must be a fork if you want the upstream repo auto-detected; otherwise pass `--upstream-url`, or have the base tag already in the clone.
- Release tags follow `<upstream tag>-est-<number>`, for example `v1.7.3-est-1` or `camel-3.14.10-est-1`. The script strips the `-est` suffix to get the upstream base tag.
- The upstream base tag exists, either in the local clone or in the upstream repo. If it cannot be found, the script exits with an error.
- Each fix names its CVE (`CVE-YYYY-` followed by four or more digits) in at least one of: a commit message (subject or body), a merged pull request title or description, or the release notes. A fix that is not named anywhere is not detected.

## Usage

Run it against a clone of the fork, with the release tag checked out so the scanned history matches the release:

```bash
# VEX for a specific release
git checkout camel-3.14.10-est-1
./git-csaf-vex-gen-ci.sh . --release-tag camel-3.14.10-est-1

# Latest release tag in the repo (auto-detected), plus one file per CVE
./git-csaf-vex-gen-ci.sh . --per-cve

# Upstream repo cannot be detected from GitHub: give it explicitly
./git-csaf-vex-gen-ci.sh . --release-tag v5.3.39-est-1 \
  --upstream-url https://github.com/spring-projects/spring-framework.git
```

```
Usage: ./git-csaf-vex-gen-ci.sh [repo-path] [product-purl] [OPTIONS]
  repo-path       path to the git repo (default: .)
  product-purl    PURL for the product (default: pkg:generic/<repo-name>@<release-tag>)

Options:
  --release-tag TAG     the fork release tag; the base tag is derived from it
  --since-ref REF       tag or commit to scan from (overrides base-tag derivation)
  --output-dir DIR      where to write VEX files (default: vex-output)
  --upstream-url URL    upstream repo URL for fetching the base tag
  --per-cve             also write one VEX file per CVE
```

The run prints each CVE it finds together with where it came from (a commit, a PR number or a release tag), and a total. Compare that against what you expect for the release before publishing the VEX.

## Release tags

Release tags are expected to look like `<upstream tag>-est-<number>`, for example `v1.7.3-est-1` or `camel-3.14.10-est-2`. The script strips the `-est` suffix to get the upstream base tag (`camel-3.14.10-est-2` becomes `camel-3.14.10`). If the base tag is not in the local clone, it is fetched from the upstream repo (taken from the GitHub fork parent, or from `--upstream-url`).

If `--release-tag` and `--since-ref` are both omitted, the script uses the most recent release tag in the repo.

## How it works

The scan window runs from the upstream base tag (inclusive) to the release. CVE IDs are collected from three places, in this order:

1. **Commit messages** (subject and body).
2. **Merged pull requests**: title and description, for PRs whose merge commit is inside the scanned window.
3. **GitHub release notes** for tags inside the window.

A CVE is recorded once, by the first source that mentions it. This matters for PRs: a PR title can name CVEs that its commit message does not.

## Output

Files are written to the output directory (default `vex-output/`):

- `combined.csaf.json`: one CSAF 2.0 VEX document listing every CVE as `fixed` in the release
- `CVE-YYYY-NNNN.csaf.json`: one document per CVE (only with `--per-cve`)

Each vulnerability entry has a description note pointing at the commit, PR or release that referenced the CVE, and a `vendor_fix` remediation. The tracking ID is `nordix-<product>-vex-<date>`. The publisher is `EST` (`https://github.com/Nordix`) and the TLP label is `WHITE`.

If no CVE references are found, no VEX file is written.

## GitHub Actions

`generate-vex.yml` runs when a release is published, and can be started manually (`workflow_dispatch`). For a release it uploads `combined.csaf.json` as a release asset; for a manual run it uploads the output directory as a workflow artifact. It expects the script at `.github/scripts/git-csaf-vex-gen-ci.sh` in the repo it runs in.

Token permissions: `contents: write` to upload release assets (`contents: read` is enough if you only generate the file), and `pull-requests: read` for private repositories so the PR scan can list pull requests.

