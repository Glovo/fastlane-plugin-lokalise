<div align="center">

# 🛠️ fastlane-plugin-lokalise

**fastlane plugin for Lokalise · download `.strings` · sync store metadata · add keys**

This repo is a Ruby gem: a [fastlane](https://docs.fastlane.tools/) plugin with three actions that talk to the
Lokalise API. It is Glovo's fork of [BasouKazuma/fastlane-plugin-lokalise](https://github.com/BasouKazuma/fastlane-plugin-lokalise),
which ports [lokalise/lokalise-fastlane-actions](https://github.com/lokalise/lokalise-fastlane-actions) to a plugin you import as a gem.
It is not a service and is not deployed. Glovo's versions are not on rubygems.org. It does not upload to the App Store or
Google Play itself; fastlane's `deliver` and `supply` actions do that.

[![Ruby](https://img.shields.io/badge/Ruby-gem-CC342D?logo=ruby&logoColor=white)](fastlane-plugin-lokalise.gemspec)
[![fastlane](https://img.shields.io/badge/fastlane-%3E%3D_2.112.0-00F200?logo=fastlane&logoColor=black)](fastlane-plugin-lokalise.gemspec)
[![Version](https://img.shields.io/badge/version-2.0.17-blue)](lib/fastlane/plugin/lokalise/version.rb)
[![RuboCop](https://img.shields.io/badge/RuboCop-0.49.1-000000?logo=rubocop&logoColor=white)](fastlane-plugin-lokalise.gemspec)
<br/>
[![Lokalise API](https://img.shields.io/badge/Lokalise_API-v2_and_v1-6E5AE6)](#external-integrations)
[![Install](https://img.shields.io/badge/install-git_source-lightgrey?logo=github)](#adding-a-lokalise-step-to-a-fastfile-lane)
<br/>
[![test files](https://img.shields.io/badge/test_files-1-lightgrey)](spec)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

</div>

---

## Table of contents

- [Overview](#overview)
- [External integrations](#external-integrations)
- [Adding a Lokalise step to a Fastfile lane](#adding-a-lokalise-step-to-a-fastfile-lane)
- [Build & local development](#build--local-development)
- [Testing](#testing)
- [Code quality](#code-quality)
- [Configuration](#configuration)
- [Repository layout](#repository-layout)
- [Contributing & conventions](#contributing--conventions)
- [Glossary](#glossary)
- [Ownership & support](#ownership--support)

---

## Overview

An app's Fastfile calls the **`lokalise`** action. The action asks the Lokalise API v2 for a `.strings` bundle, downloads the
ZIP and unpacks it into a `destination` folder of `<lang>.lproj/Localizable.strings` files. A **file strategy**
(`override`, `merge` or `update`) decides how each downloaded file is combined with the one already on disk. The two other
actions move App Store / Google Play metadata between `fastlane/metadata` and Lokalise, and create keys in Lokalise.

Glovo added 41 commits on top of the upstream repo between 2020-01-21 and 2022-06-28: API v2 support, file strategies,
`tags`, `escape_percent`, the `api.lokalise.com` domain fix and "only write files that already exist" (PR #9). The last
commit on `master` is 2022-06-28.

**Note:** the commands in version 2.x are not compatible with version 1.x.

| Action | Lokalise API | Purpose |
|---|---|---|
| [`lokalise`](lib/fastlane/plugin/lokalise/actions/lokalise_action.rb) | v2 | Downloads `.strings` and `.stringsdict` files into the destination folder |
| [`lokalise_metadata`](lib/fastlane/plugin/lokalise/actions/lokalise_metadata_action.rb) | v1 | Downloads store metadata from Lokalise into `fastlane/metadata`, or uploads the files `deliver` / `supply` generated there to Lokalise |
| [`add_keys_to_lokalise`](lib/fastlane/plugin/lokalise/actions/add_keys_to_lokalise_action.rb) | v1 | Uploads a list of keys to Lokalise |

> [!WARNING]
> **`lokalise_metadata` and `add_keys_to_lokalise` are not compatible with V2 of the Lokalise API and will not work.**
> They still call the v1 endpoints under `https://api.lokalise.com/api/`. Only `lokalise` uses API v2.

| | |
|---|---|
| Language / build | Ruby gem ([gemspec](fastlane-plugin-lokalise.gemspec)), Bundler, Rake |
| Runtime | Loaded by fastlane inside an app's lanes; platforms `ios`, `android`, `mac` |
| Upstream API | Lokalise (`https://api.lokalise.com`) |
| Distribution | Git source in the app's `Pluginfile` (tags `1.0.0` … `v2.0.17`; `master` is one PR ahead, see [Releasing](#releasing)) |
| Owner | None declared (no `CODEOWNERS`, no `catalog-info.yaml`) |

> [!NOTE]
> The README of [Glovo/fastlane-plugin-fetch_localized_strings](https://github.com/Glovo/fastlane-plugin-fetch_localized_strings)
> says that repo was copied from this one and simplified to fetch strings from the in-house translations service, and that
> the Lokalise-specific code moved to a separate private repository. Check with the mobile teams before building on this one.

---

## External integrations

| Integration | Upstream | Direction | Purpose | Auth |
|---|---|---|---|---|
| Lokalise API v2 | `POST https://api.lokalise.com/api2/projects/<project_identifier>/files/download` | → out | `lokalise`: request a `strings` bundle | `x-api-token` header from `api_token` |
| Bundle download | `GET` on the `bundle_url` returned by the call above | → out | `lokalise`: fetch the ZIP | None sent |
| Lokalise API v1 | `POST https://api.lokalise.com/api/string/list`, `string/set`, `language/add` | → out | `lokalise_metadata`, `add_keys_to_lokalise` | `api_token` form field |

- **`lokalise` logs errors and carries on.** A non-2xx response, a response without `bundle_url`, or a download that is not
  `application/zip` / `application/octet-stream` is reported with `UI.error`, which does not raise. The lane keeps going
  and the files stay as they were.
- **The v1 actions raise** on an `error` status or a malformed response, so the lane fails.
- `lokalise` saves the ZIP as `lokalisetmp/a.zip` in the working directory (`merge` and `update` also stage files there)
  and deletes `lokalisetmp/` once unpacking finishes.

---

## Adding a Lokalise step to a Fastfile lane

This section assumes you know fastlane's Fastfile and how to work with it. If you don't, read the
[fastlane documentation](https://docs.fastlane.tools/) first.

1. Add the plugin to the app's `fastlane/Pluginfile` with the git source. `Glovo/sample-mobile-ios` uses this line:
   ```ruby
   gem 'fastlane-plugin-lokalise', git: "https://github.com/Glovo/fastlane-plugin-lokalise"
   ```
2. Make sure the app's `Gemfile` loads the `Pluginfile`, with the same `eval_gemfile` lines as this repo's [`Gemfile`](Gemfile). Then run `bundle install`.
3. Export `LOKALISE_API_TOKEN` and `LOKALISE_PROJECT_ID` in the environment that runs fastlane. Don't write the token into the Fastfile.
4. Check that `destination` exists and already holds the `<lang>.lproj/Localizable.strings` files you want refreshed (see [`lokalise` file strategies](#lokalise-file-strategies)).
5. Call the action in a lane:
   ```ruby
   lokalise(
     destination: "<path-to-the-folder-with-lproj-dirs>",
     file_strategy: "merge",   # override (default) | merge | update
   )
   ```
6. Run the lane: `bundle exec fastlane <lane>`.

> [!WARNING]
> **`fastlane add_plugin lokalise`** and a plain **`gem 'fastlane-plugin-lokalise'`** install the gem from rubygems.org.
> There, `fastlane-plugin-lokalise` stops at **2.0.6** and is the upstream author's build, without any of Glovo's changes
> (checked 2026-10-29). Use the `git:` source from step 1 to get this fork.

<details><summary><b>Upload metadata to the App Store or Google Play</b></summary>

> `lokalise_metadata` uses API v1 and does not work against V2 (see [Overview](#overview)). This recipe is kept for reference.

First download the metadata from Lokalise into the `fastlane/metadata` directory:

```ruby
lokalise_metadata(
    action: "download_from_lokalise",
    api_token: "<api-token>",
    project_identifier: "<number>.<number>",
)
```

Then upload what is currently in the `fastlane/metadata` folder to the App Store with the
[deliver](https://docs.fastlane.tools/actions/deliver/) action:

```ruby
deliver(
    ...
)
```

Or to Google Play with the [supply](https://docs.fastlane.tools/actions/supply/) action:

```ruby
supply(
    ...
)
```

For Android, `download_from_lokalise` also needs `release_number` (an integer greater than 0). The changelog is written to
`changelogs/<release_number>.txt`.

</details>

<details><summary><b>Upload metadata to Lokalise</b></summary>

> `lokalise_metadata` uses API v1 and does not work against V2 (see [Overview](#overview)). This recipe is kept for reference.

First download the metadata from the App Store ([deliver](https://docs.fastlane.tools/actions/deliver/)):

```ruby
desc "Downloads metadata from App Store Connect"
lane :download_appstore_metadata do
    ENV["DELIVER_FORCE_OVERWRITE"] = "1"
    sh("fastlane deliver download_metadata")
    ENV["DELIVER_FORCE_OVERWRITE"] = "0"
end
```

Or from Google Play ([supply](https://docs.fastlane.tools/actions/supply/)):

```ruby
desc "Downloads metadata from Google Play"
lane :download_googleplay_metadata do
    # Replace metadata_path if not in the default location
    metadata_path = "metadata/android"
    sh("rm -rf #{metadata_path}")
    sh("pushd ..; fastlane supply init; popd")
end
```

Then upload what is currently in the `fastlane/metadata` folder to Lokalise:

```ruby
lokalise_metadata(
    action: "upload_to_lokalise",
    add_languages: true,
    override_translation: true
)
```

</details>

---

## Build & local development

> [!NOTE]
> You need **Ruby** and **Bundler**. The repo pins no Ruby version: CircleCI uses the `circleci/ruby:2.4.2` image,
> [`.travis.yml`](.travis.yml) lists `2.2.4`, and RuboCop targets 2.0. fastlane (`>= 2.112.0`) comes in as a development
> dependency from the [gemspec](fastlane-plugin-lokalise.gemspec).

```bash
bundle install         # development dependencies from the gemspec
bundle exec rake       # default task: spec + rubocop
rake install           # build the gem and install it locally
```

### Run locally

[`fastlane/Fastfile`](fastlane/Fastfile) has one lane, `test`, which calls `lokalise` with no arguments. Run it with
`bundle exec fastlane test`. It reads the token and project from `LOKALISE_API_TOKEN` and `LOKALISE_PROJECT_ID`.
`destination` has no environment variable, so pass it another way.

### Releasing

```bash
rake release           # bundler/gem_tasks: build, tag vX.Y.Z, push the tag and the gem
```

> [!WARNING]
> `rake release` also pushes to rubygems.org, where the `fastlane-plugin-lokalise` gem is the upstream author's (latest
> 2.0.6). Glovo's releases exist only as git tags (`v2.0.7` … `v2.0.17`), and consumers pin the git source.

PR #9 (2022-06-28) was merged after tag `v2.0.17` without a version bump. `master` (`235d3f1`) still reports 2.0.17
but differs from the tag, so pinning `tag: "v2.0.17"` and pinning `master` give different behaviour.

---

## Testing

> [!WARNING]
> The only spec, [`spec/lokalise_action_spec.rb`](spec/lokalise_action_spec.rb), is the plugin template's placeholder.
> It calls `LokaliseAction.run(nil)` and expects the message `"The lokalise plugin is working!"`, which no code prints.
> It fails against the current action.

```bash
bundle exec rake spec  # RSpec only
bundle exec rspec      # RSpec directly; options come from .rspec
```

- [`.rspec`](.rspec) loads `spec_helper`, prints documentation format and writes JUnit XML to `test-results/rspec/rspec.xml`.
- [`spec/spec_helper.rb`](spec/spec_helper.rb) starts SimpleCov. The minimum-coverage line is commented out. `coverage` and
  `test-results` are git-ignored.
- [`.circleci/config.yml`](.circleci/config.yml) has one `build` job that runs on every push: `bundle install`, then
  `bundle exec rake`, then it stores `test-results`. The last commit on `master` has no CI status on GitHub.

---

## Code quality

| Tool | Role |
|---|---|
| RuboCop 0.49.1 + `rubocop-require_tools` | Lint, configured in [`.rubocop.yml`](.rubocop.yml). The config names fastlane paths such as `spaceship/` and `supply/` that don't exist in this repo |
| Rake | [`Rakefile`](Rakefile) default task runs `spec` and then `rubocop` |

There are no git hooks.

---

## Configuration

Each action reads its settings from fastlane options. Some options fall back to environment variables:

| Variable | Example / default | Notes |
|---|---|---|
| `LOKALISE_API_TOKEN` | — | `api_token` for all three actions. Keep it in CI secrets; never commit it |
| `LOKALISE_PROJECT_ID` | `<number>.<number>` | `project_identifier` for all three actions |
| `FASTLANE_PLATFORM_NAME` | `ios` or `android` | `platform` for `lokalise_metadata` |

<details><summary><b><code>lokalise</code> options</b></summary>

| Option | Default | Notes |
|---|---|---|
| `api_token` | env `LOKALISE_API_TOKEN` | Required |
| `project_identifier` | env `LOKALISE_PROJECT_ID` | Required |
| `destination` | — | Required; must be an existing directory |
| `clean_destination` | `false` | Empties `destination` before unpacking (see the warning below) |
| `languages` | all | Array of language codes, sent as `filter_langs` |
| `tags` | all keys | Array; only keys with these tags (`include_tags`) |
| `include_comments` | `false` | Include comments in exported files |
| `use_original` | `false` | Use original filenames/formats; the bundle structure is then ignored |
| `export_empty_as` | `empty` | `empty`, `base` or `skip`: what to do with empty translations |
| `export_sort` | `last_added` | `first_added`, `last_added`, `last_updated`, `a_z` or `z_a` |
| `escape_percent` | `false` | printf placeholders only: export universal `[%]` placeholders as `%%` |
| `file_strategy` | `override` | `override`, `merge` or `update` (see below) |

The request always uses format `strings` and bundle structure `%LANG_ISO%.lproj/Localizable.%FORMAT%`.

</details>

<details><summary><b><code>lokalise_metadata</code> options</b></summary>

| Option | Default | Notes |
|---|---|---|
| `platform` | env `FASTLANE_PLATFORM_NAME` | `ios` or `android` |
| `api_token` | env `LOKALISE_API_TOKEN` | Required |
| `project_identifier` | env `LOKALISE_PROJECT_ID` | Required |
| `action` | — | Required: `download_from_lokalise` or `upload_to_lokalise` |
| `metadata_path` | `fastlane/metadata/` (iOS), `fastlane/metadata/android/` (Android) | Where metadata files are read and written |
| `add_languages` | `false` | Add missing languages to Lokalise before uploading |
| `override_translation` | `false` | `true` uploads everything. `false` uploads only keys that are empty in Lokalise |
| `release_number` | — | Android `download_from_lokalise` only; integer greater than 0 |

Store fields map to Lokalise keys with platform mask 16 ("Other"):

| Store | Lokalise keys | Files |
|---|---|---|
| App Store | `appstore.app.name`, `.description`, `.keywords`, `.promotional_text`, `.release_notes`, `.subtitle`, `.marketing_url`, `.privacy_url`, `.support_url` | `<lang>/<field>.txt` |
| Google Play | `googleplay.app.title`, `.full_description`, `.short_description`, `.changelogs` | `<lang>/<field>.txt`, `<lang>/changelogs/<release_number>.txt` |

Language codes are translated between store and Lokalise formats by two maps in
[`lokalise_metadata_action.rb`](lib/fastlane/plugin/lokalise/actions/lokalise_metadata_action.rb) (for example `en-US` → `en`, `zh-Hans` → `zh_CN`, `iw-IL` → `he`).

</details>

<details><summary><b><code>add_keys_to_lokalise</code> options</b></summary>

| Option | Default | Notes |
|---|---|---|
| `api_token` | env `LOKALISE_API_TOKEN` | Required |
| `project_identifier` | env `LOKALISE_PROJECT_ID` | Required |
| `platform_mask` | `1` | Platform mask to assign to keys: 1 is iOS, 2 is Android, 4 is Web and 16 is Other |
| `keys` | — | Required; array of non-empty strings |

</details>

### `lokalise` file strategies

| `file_strategy` | Effect on an existing `.strings` file |
|---|---|
| `override` (default) | Replaces the file with the downloaded one |
| `merge` | Keeps local keys, adds new keys from Lokalise, and Lokalise values win |
| `update` | Updates only keys that already exist locally; new keys from Lokalise are ignored |

Files that don't end in `.strings` (for example `.stringsdict`) are always overridden. `merge` and `update` read one
`"key" = "value";` pair per line and rewrite the file sorted by key, so comments and blank lines are not kept as they were.

> [!WARNING]
> Since PR #9 (2022-06-28), **only files that already exist under `destination` are written**. Downloaded files that
> aren't there yet (for example a new language) are skipped without a message. With `clean_destination: true` the folder is emptied first, so nothing is
> written even though the action reports success
> ([`lokalise_action.rb`](lib/fastlane/plugin/lokalise/actions/lokalise_action.rb), `unzip_file`).

---

## Repository layout

```text
.
├── lib/fastlane/plugin/
│   ├── lokalise.rb                        # loads every file in actions/ and helper/
│   └── lokalise/
│       ├── actions/                       # ⭐ the three fastlane actions
│       │   ├── lokalise_action.rb         # download .strings (API v2) + file strategies
│       │   ├── lokalise_metadata_action.rb  # store metadata <-> Lokalise (API v1)
│       │   └── add_keys_to_lokalise_action.rb  # create keys (API v1)
│       ├── helper/lokalise_helper.rb      # template helper, not called anywhere
│       └── version.rb                     # gem version
├── spec/                                  # RSpec placeholder spec + spec_helper
├── fastlane/                              # Fastfile with a `test` lane; empty Pluginfile
├── fastlane-plugin-lokalise.gemspec       # gem metadata and dev dependencies
├── Gemfile · Rakefile                     # Bundler; rake = spec + rubocop
├── .circleci/config.yml · .travis.yml     # CI configs
└── .rubocop.yml · .rspec                  # lint and test settings
```

`test-results/`, `coverage/`, `*.gem`, `Gemfile.lock` and `fastlane/README.md` are git-ignored.

---

## Contributing & conventions

1. Branch from `master` and open a PR; changes land as GitHub merge commits.
2. Run `bundle exec rake` before pushing (expect the placeholder spec to fail, see [Testing](#testing)).
3. **Don't add a runtime dependency** on `fastlane` in the gemspec: that would cause a circular dependency (gemspec comment).
4. After a merge, bump [`version.rb`](lib/fastlane/plugin/lokalise/version.rb) in a commit named `vX.Y.Z` and tag it `vX.Y.Z`. History follows this up to `v2.0.17`.
5. When you add or change an option in `available_options`, update the option tables under [Configuration](#configuration).
6. Never put a Lokalise API token in the repo, a Fastfile or an example.

There is no `CODEOWNERS` file.

---

## Glossary

| Term | Meaning |
|---|---|
| fastlane | Ruby tool that automates mobile builds and releases with lanes defined in a `Fastfile` |
| Pluginfile | `fastlane/Pluginfile`, the Gemfile fragment that lists fastlane plugins |
| deliver / supply | fastlane actions that upload metadata to the App Store / Google Play |
| Lokalise | Translation management service the actions call |
| `.strings` / `.stringsdict` | Apple localization files (key-value strings / plural rules) inside `<lang>.lproj/` |
| Platform mask | Lokalise key platform: 1 iOS, 2 Android, 4 Web, 16 Other |
| File strategy | How `lokalise` combines a downloaded `.strings` file with the local one |

---

## Ownership & support

- **Team:** none declared. There is no `CODEOWNERS` file and no `catalog-info.yaml`.
- **Access:** the GitHub teams [`ios`](https://github.com/orgs/Glovo/teams/ios) and [`developers`](https://github.com/orgs/Glovo/teams/developers) have push access.
- **Upstream:** [BasouKazuma/fastlane-plugin-lokalise](https://github.com/BasouKazuma/fastlane-plugin-lokalise) (fork parent, MIT, see [`LICENSE`](LICENSE)).
- **Related:** [Glovo/fastlane-plugin-fetch_localized_strings](https://github.com/Glovo/fastlane-plugin-fetch_localized_strings), the simplified copy for the in-house translations service.
