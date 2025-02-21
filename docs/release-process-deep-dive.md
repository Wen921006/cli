# Release Process Deep Dive

## Motivation

The current release workflow and associated scripts were created before the current set of maintainers, and all maintainers from that time have left. On a number of occasions (releasing a MacOS installer, moving to Azure HCS signing, updating expired GPG key) the current maintainers have spent time investigating the release workflow. This document is intended to serve as a guide for future maintainers who need to understand the release process.

## High Level Overview

From a high level, the release workflow:
 * Is triggered by a `workflow_dispatch` event (typically a result of running `./script/release`)
 * Builds, packages and signs artifacts in parallel for Linux, MacOS and Windows
 * GPG signs our Debian and Red Hat repository artifacts
 * Builds and updates the [marketing site](https://cli.github.com), [manual](https://cli.github.com/manual) and repo packages
 * Creates GitHub Attestations for the artifacts
 * Creates a GitHub Release and attaches the artifacts
 * Bumps the `gh` [homebrew-core formula](https://github.com/Homebrew/homebrew-core/blob/2df031cbd8f7bc9b9a380e941ccefcf3c8f3d02b/Formula/g/gh.rb)

## Jobs Deep Dive

This section will deep dive into each job in the [`deployment.yml` workflow](https://github.com/cli/cli/blob/537a22228cd6b42b740d7f1c09f47c45bb1dab30/.github/workflows/deployment.yml).

// TODO Screenshot of the deployment workflow

- [validate-tag-name](#validate-tag-name)
- [linux](#linux)
- [macos](#macos)
- [windows](#windows)
- [release](#release)

### <a id="validate-tag-name">[validate-tag-name](https://github.com/cli/cli/blob/537a22228cd6b42b740d7f1c09f47c45bb1dab30/.github/workflows/deployment.yml#L31-L39)</a>

<details>

```yml
  validate-tag-name:
    runs-on: ubuntu-latest
    steps:
      - name: Validate tag name format
        run: |
          if [[ ! "${{ inputs.tag_name }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Invalid tag name format. Must be in the form v1.2.3"
            exit 1
          fi
```
</details>

The purpose of this job is to prevent releases that are tagged incorrectly, by ensuring they conform to the major.minor.patch form of semantic versioning, preceeded by a `v`. [Reference](https://github.com/cli/cli/pull/10121).

### <a id="linux">linux</a>

<details>

```yml
  linux:
    needs: validate-tag-name
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    if: contains(inputs.platforms, 'linux')
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: 'go.mod'
      - name: Install GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: "~1.17.1"
          install-only: true
      - name: Build release binaries
        env:
          TAG_NAME: ${{ inputs.tag_name }}
        run: script/release --local "$TAG_NAME" --platform linux
      - name: Generate web manual pages
        run: |
          go run ./cmd/gen-docs --website --doc-path dist/manual
          tar -czvf dist/manual.tar.gz -C dist -- manual
      - uses: actions/upload-artifact@v4
        with:
          name: linux
          if-no-files-found: error
          retention-days: 7
          path: |
            dist/*.tar.gz
            dist/*.rpm
            dist/*.deb
```
</details>




### <a id="macos">macos</a>

### <a id="windows">windows</a>

### <a id="release">release</a>
