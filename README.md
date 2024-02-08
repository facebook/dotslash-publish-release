# dotslash-publish-release

This GitHub action can create [DotSlash](https://dotslash-cli.com/) files for
executables that you have published as part of a [GitHub release](
https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases).
The newly generated DotSlash files will be added to the existing release.

This action is designed to run after the [GitHub Actions workflows](
https://docs.github.com/en/actions/using-workflows) that are responsible for
uploading your primary release artifacts via `gh release upload` or equivalent.

## Example

If you had separate workflows for each platform such as `linux-release`,
`macos-release`, and `windows-release`, then you could define a new GitHub
action under `.github/workflows/dotslash.yml` as follows:


```yaml
name: Generate DotSlash files

on:
  workflow_run:
    # These must match the names of the workflows that publish
    # artifacts to your GitHub release.
    workflows: [linux-release, macos-release, windows-release]
    types:
      - completed

jobs:
  generate-dotslash-files:
    name: Generating and uploading DotSlash files
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: facebook/dotslash-publish-release@v1
        # This is necessary because the action uses
        # `gh release upload` to publish the generated DotSlash file(s)
        # as part of the release.
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          # Additional file that lives in your repo that defines
          # how your DotSlash file(s) should be generated.
          config: .github/workflows/dotslash-config.json
          # Tag for the release to to target.
          tag: ${{ github.event.workflow_run.head_branch }}
```

Note the `config` line that specifies a path to a JSON file in your repo that
determines what DotSlash files to generate. For example, if this GitHub action
were defined in the [facebook/hermes](https://github.com/facebook/hermes)
repository on GitHub, and the contents of
`.github/workflows/dotslash-config.json` were as follows:

```json
{
  "outputs": {
    "hermes": {
      "platforms": {
        "macos-x86_64": {
          "regex": "^hermes-cli-darwin-",
          "path": "hermes"
        },
        "macos-aarch64": {
          "regex": "^hermes-cli-darwin-",
          "path": "hermes"
        },
        "linux-x86_64": {
          "regex": "^hermes-cli-linux-",
          "path": "hermes"
        },
        "windows-x86_64": {
          "regex": "^hermes-cli-windows-",
          "path": "hermes.exe"
        }
      }
    }
  }
}
```

Then this action would have added the following DotSlash file named `hermes` to
the [v0.12.0 release](https://github.com/facebook/hermes/releases/tag/v0.12.0):

```json
#!/usr/bin/env dotslash

{
  "name": "hermes",
  "platforms": {
    "macos-x86_64": {
      "size": 10600817,
      "hash": "blake3",
      "digest": "25f984911f199f9229ca0327c52700fa9a8db9aefe95e84f91ba6be69902436a",
      "format": "tar.gz",
      "path": "hermes",
      "providers": [
        {
          "url": "https://github.com/facebook/hermes/releases/download/v0.12.0/hermes-cli-darwin-v0.12.0.tar.gz"
        },
        {
          "type": "github-release",
          "repo": "https://github.com/facebook/hermes",
          "tag": "v0.12.0",
          "name": "hermes-cli-darwin-v0.12.0.tar.gz"
        }
      ]
    },
    "macos-aarch64": {
      "size": 10600817,
      "hash": "blake3",
      "digest": "25f984911f199f9229ca0327c52700fa9a8db9aefe95e84f91ba6be69902436a",
      "format": "tar.gz",
      "path": "hermes",
      "providers": [
        {
          "url": "https://github.com/facebook/hermes/releases/download/v0.12.0/hermes-cli-darwin-v0.12.0.tar.gz"
        },
        {
          "type": "github-release",
          "repo": "https://github.com/facebook/hermes",
          "tag": "v0.12.0",
          "name": "hermes-cli-darwin-v0.12.0.tar.gz"
        }
      ]
    },
    "linux-x86_64": {
      "size": 47099598,
      "hash": "blake3",
      "digest": "8d2c1bcefc2ce6e278167495810c2437e8050780ebb4da567811f1d754ad198c",
      "format": "tar.gz",
      "path": "hermes",
      "providers": [
        {
          "url": "https://github.com/facebook/hermes/releases/download/v0.12.0/hermes-cli-linux-v0.12.0.tar.gz"
        },
        {
          "type": "github-release",
          "repo": "https://github.com/facebook/hermes",
          "tag": "v0.12.0",
          "name": "hermes-cli-linux-v0.12.0.tar.gz"
        }
      ]
    },
    "windows-x86_64": {
      "size": 17456100,
      "hash": "blake3",
      "digest": "7efee4f92a05e34ccfa7c21c7a05f939d8b724bc802423d618db22efb83bfe1b",
      "format": "tar.gz",
      "path": "hermes.exe",
      "providers": [
        {
          "url": "https://github.com/facebook/hermes/releases/download/v0.12.0/hermes-cli-windows-v0.12.0.tar.gz"
        },
        {
          "type": "github-release",
          "repo": "https://github.com/facebook/hermes",
          "tag": "v0.12.0",
          "name": "hermes-cli-windows-v0.12.0.tar.gz"
        }
      ]
    }
  }
}
```

Note that each entry in `platforms` in the `dotslash-config.json` is reflected
in the `platforms` section of the generated DotSlash file. Each config entry
takes a `"name"` or a `"regex"` to use to identify the appropriate artifact in
the release and the `"path"` indicates the `"path"` that should be used for the
artifact in the generated DotSlash file.

The `dotslash-publish-release` action defaults to using BLAKE3 as the hash
function, so it takes responsibility for computing the `size` and `digest`
values. It also tries to "guess" the appropriate value of `"format"` based on
the suffix of the URL, though this can also be specified explicitly, which is a
bit safer:

```json
{
  "outputs": {
    "hermes": {
      "platforms": {
        "macos-x86_64": {
          "regex": "^hermes-cli-darwin-",
          "format": "tar.gz",
          "path": "hermes"
        },
        ...
```

By default, `dotslash-publish-release` generates both the HTTP provider as well
as the `github-release` provider for each entry in the DotSlash file. Either of
these can be disabled via top-level `"exclude-http-provider"` and
`"exclude-github-release-provider"` properties, respectively. For example, if
you are using this action in a private GitHub repo, then you probably want to
disable the HTTP provider:

```json
{
  "exclude-http-provider": true,
  "outputs": {
    "hermes": {
      "platforms": {
        "macos-x86_64": {
          "regex": "^hermes-cli-darwin-",
          "format": "tar.gz",
          "path": "hermes"
        },
        ...
```

The generated DotSlash file would reflect this change:

```json
#!/usr/bin/env dotslash

{
  "name": "hermes",
  "platforms": {
    "macos-x86_64": {
      "size": 10600817,
      "hash": "blake3",
      "digest": "25f984911f199f9229ca0327c52700fa9a8db9aefe95e84f91ba6be69902436a",
      "format": "zst",
      "path": "hermes",
      "providers": [
        {
          "type": "github-release",
          "repo": "https://github.com/facebook/hermes",
          "tag": "v0.12.0",
          "name": "hermes-cli-darwin-v0.12.0.tar.gz"
        }
      ]
    },
    ...
```

## Config File Details

The most important part of the config file is the top-level `"outputs"` entry.
Each key in this entry will be the name of the generated DotSlash file that is
added to the release.

The `"platforms"` map for each entry requires that the keys are [platforms that
are recognized by DotSlash](
https://dotslash-cli.com/docs/dotslash-file/).

Each platform entry recognizes the following properties:

* One of `regex` or `name` is required to identify the file in the release that
  should be used as the DotSlash artifact for the platform.
* `path` is required and is used as the corresponding `path` value in the
  DotSlash file.
* `format` is optional, but recommended. It must be a valid [DotSlash artifact
  format](
  https://dotslash-cli.com/docs/dotslash-file/#artifact-format), such as
  `tar.gz`. If the artifact is not compressed, then `"format": null` must be
  specified explicitly in the config JSON.
* `hash` must be one of `"blake3"` or `"sha256"`, but it defaults to `"blake3"`,
  so it is optional.

## License

dotslash-publish-release is [MIT licensed](./LICENSE).
