# Lokalise Fastlane Plugin

This project is a port of the [lokalise-fastlane-actions](https://github.com/lokalise/lokalise-fastlane-actions) project. The goal is for it to be packaged as a Fastlane Plugin that can be imported as a gem.

**Note:** The commands in version 2.x are not compatible with version 1.x.


## Install

This can be added with the following command:
```bash
fastlane add_plugin lokalise
```

Or added to the Pluginfile as:
```
gem 'fastlane-plugin-lokalise'
```


## Actions

### lokalise

This action downloads .strings and .stringsdict files to destination folder.

Parameters:

- `api_token`. Your API token. Can be set up using enviromental parameter `LOKALISE_API_TOKEN`
- `project_identifier`. Your Project ID. Can be set up using enviromental parameter `LOKALISE_PROJECT_ID`
- `destination`. Localization files destination.
- `clean_destination`. Cleans destination folder if set to `true` *(`false` by default)*.
- `languages`. Languages to download *(must be passed as array of strings, leave empty to download all)*.
- `include_comments`. Include comments in exported files.
- `use_original`. Use original filenames/formats.
- `export_empty_as`. Define the strategy for empty translations.
- `export_sort`. Define the strategy for sorting translations.

### lokalise_metadata

This action imports metadata from files generated by Deliver action and uploads iTunes Connect metadata using information from Lokalise.

- `platform`. The name of the platform you are generating metadata for (`android` or `ios`). Can be set up using enviromental parameter `FASTLANE_PLATFORM_NAME`
- `api_token`. Your API token. Can be set up using enviromental parameter `LOKALISE_API_TOKEN`
- `project_identifier`. Your Project ID. Can be set up using enviromental parameter `LOKALISE_PROJECT_ID`
- `action`. Action to perform *(can be `download_from_lokalise`, or `upload_to_lokalise`)*. 
- `add_languages`. Add missing languages to lokalise *(`false` by default)*.
- `override_translation`. Override translations in lokalise.
- `release_number`. Application release number. Required for Android actions.

### add_keys_to_lokalise

This actions allow you upload keys to Lokalise.

Parameters:

- `api_token`. Your API token. Can be set up using enviromental parameter `LOKALISE_API_TOKEN`
- `project_identifier`. Your Project ID. Can be set up using enviromental parameter `LOKALISE_PROJECT_ID`
- `platform_mask`. Platform mask to asign to keys *(1 is iOS, 2 is Android, 4 is Web and 16 is Other)*.
- `keys`. Keys to add *(must be passed as array of strings)*.


## How To

In this section we assume you know about Fastlane's Fastfile and how to interact with it. If you do not, please look over the Fastlane Documentation.
https://docs.fastlane.tools/

### Upload metadata to the App Store or Google Play

First you need to download the metadata from Lokalise into the `fastlane/metadata` directory:
```ruby
lokalise_metadata(
    action: download_from_lokalise,
    api_token: (string),
    project_identifier: (number.number),
)
```

Then you can upload what is currently in the `fastlane/metadata` folder to the App Store using the deliver action:
https://docs.fastlane.tools/actions/deliver/
```ruby
deliver(
    ...
)
```
Or for Google Play the supply action:
https://docs.fastlane.tools/actions/supply/
```ruby
supply(
    ...
)
```

### Upload metadata to Lokalise

First you need to download the metadata from the App Store:
https://docs.fastlane.tools/actions/deliver/
```ruby
desc "Downloads metadata from App Store Connect"
lane :download_appstore_metadata do
    ENV["DELIVER_FORCE_OVERWRITE"] = "1"
    sh("fastlane deliver download_metadata")
    ENV["DELIVER_FORCE_OVERWRITE"] = "0"
end
```

Or from Google Play:
https://docs.fastlane.tools/actions/supply/
```ruby
desc "Downloads metadata from Google Play"
lane :download_googleplay_metadata do
    # Replace metadata_path if not in the default location
    metadata_path = "metadata/android"
    sh("rm -rf #{metadata_path}")
    sh("pushd ..; fastlane supply init; popd")
end
```

Then you can upload what is currently in the `fastlane/metdata` folder to Lokalise using:
```ruby
lokalise_metadata(
    action: "update_lokalise_itunes",
    add_languages: true,
    override_translation: true
)
```

## Development

This can be locally built and installed as a gem by doing the following on the repo:
```bash
bundle install
rake install
```

This can be published using:
```bash
rake release
```
