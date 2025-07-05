# ESOUI Addon Upload GitHub Action

![ESOUI Upload](https://img.shields.io/badge/ESOUI-Upload-blue?logo=github-actions&style=flat-square)
[![Used by](https://img.shields.io/badge/dynamic/json?color=success&label=Used%20by&query=repositoryCount&url=https://api.github.com/repos/m00nyONE/esoui-upload)](https://github.com/m00nyONE/esoui-upload/network/dependents)
[![GitHub release](https://img.shields.io/github/v/tag/m00nyONE/esoui-upload?label=release)](https://github.com/m00nyONE/esoui-upload/releases)


This GitHub Action automates the upload of your ESOUI (Elder Scrolls Online User Interface) addon to [esoui.com](https://esoui.com).
This only works for PC addons.

It wraps the upload process in a safe and reusable action, helping you keep your secrets secure and your workflows clean.

## Features

- ✅ Secure: API key is masked from logs.
- ✅ Validation: all inputs are validated.
- ✅ Simple: Clean and easy integration in your workflows.
- ✅ Customizable: Reusable across multiple addons or repositories.

## Usage

### 0. Prerequisite

- The addon needs to be uploaded manually to ESOUI once to get it's ID. You can do this here: https://www.esoui.com/downloads/upload-update.php
- generate an API key here: https://www.esoui.com/downloads/filecpl.php?action=apitokens

### 1. Reference the Action in Your Workflow

Example workflow step:

```yaml
- name: Upload to ESOUI
  uses: m00nyONE/esoui-upload@v1
  with:
    api_key: ${{ secrets.ESOUI_API_KEY }}
    addon_id: '123456'
    version: ${{ env.ADDON_VERSION }}
    zip_file: ${{ env.ZIP_FULL_NAME }}
    changelog_file: 'CHANGELOG.md'
    description_file: 'README_ESOUI.txt'
    compatibility: '11.0.0'
    test: false
```

### 2. Inputs

| Name             | Required | Default          | Description                                    |
|------------------|----------|------------------|------------------------------------------------|
| api_key          | true     | -                | Your ESOUI API key (stored as a GitHub secret) |
| addon_id         | true     | -                | The ESOUI Addon ID                             |
| version          | true     | -                | The version number of the addon                |
| zip_file         | true     | -                | Path to your zipped addon file                 |
| changelog_file   | false    | CHANGELOG.md     | Path to your changelog file                    |
| description_file | false    | README_ESOUI.txt | Path to your description file                  |
| compatibility    | false    | -                | Compatible version of the game                 |
| test             | false    | false            | Whether to use test API                        |

### 3. Secrets

Make sure to add your ESOUI API key to your repository’s secrets:
- Go to **Repository Settings > Secrets and variables > Actions > New repository secret**
- Name: ESOUI_API_KEY
- Value: (your ESOUI API key)

### 4. Full Workflow Example

This is an example based on HodorReflexes. Adapt it to your needs.

```yaml
name: ESOUI Release

on:
  push:
    branches:
      - release # to be changed to main with schedule

jobs:
  release:
    name: "release"
    runs-on: ubuntu-latest
    permissions: write-all
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Get current year, month and day
        run: |
          echo "BUILD_DATE_YEAR=$(date -u +'%Y')" >> $GITHUB_ENV
          echo "BUILD_DATE_MONTH=$(date -u +'%m')" >> $GITHUB_ENV
          echo "BUILD_DATE_DAY=$(date -u +'%d')" >> $GITHUB_ENV
          echo "BUILD_DATE_NUMBER=$(date +'%Y%m%d')" >> $GITHUB_ENV
          echo "BUILD_DATE_WITH_DOT=$(date +'%Y.%m.%d')" >> $GITHUB_ENV
          echo "BUILD_DATE_WITH_HYPHEN=$(date +'%Y-%m-%d')" >> $GITHUB_ENV

      - name: Create env variables
        run: |
          addon_name="HodorReflexes"
          version="${{ env.BUILD_DATE_WITH_HYPHEN }}"
          
          zip_name="${addon_name}-${version}.zip"
          
          echo "ADDON_NAME=$addon_name" >> $GITHUB_ENV
          echo "ADDON_VERSION=$version" >> $GITHUB_ENV
          echo "ZIP_FULL_NAME=$zip_name" >> $GITHUB_ENV

      - name: Replace placeholders with current date
        run: |
          sed -i "s/version = \"dev\"/version = \"${{ env.ADDON_VERSION }}\"/g" HodorReflexes.lua
          sed -i "s/## Version: dev/## Version: ${{ env.ADDON_VERSION }}/g" HodorReflexes.addon
          sed -i "s/## AddOnVersion: 99999999/## AddOnVersion: ${{ env.BUILD_DATE_NUMBER }}/g" HodorReflexes.addon

      - name: Create ZIP archive
        run: |
          REPO_FOLDER=$(pwd)
          TMP_FOLDER="/tmp/${{ env.ADDON_NAME }}"
          
          # Define the path to the ignore pattern file
          ignore_file=".build-ignore"
          
          # Read and process ignore patterns into a single line
          exclude_patterns=$(cat "$ignore_file" | awk '{print "--exclude " $0}' | tr '\n' ' ')
          
          # Make folder and copy content
          mkdir -p $TMP_FOLDER
          rsync -a --quiet $exclude_patterns "$REPO_FOLDER/" "$TMP_FOLDER/"
          
          # Make full version zip
          (cd /tmp && zip -r --quiet "$REPO_FOLDER/${{ env.ZIP_FULL_NAME }}" "${{ env.ADDON_NAME }}")

      - name: Extract latest changelog entry
        run: |
          awk '/^## / { if (!found) { found=1; print; next } else { exit } } found' CHANGELOG.md > latest_changes.md
          cat latest_changes.md

      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          name: "${{ env.ADDON_VERSION }}"
          commit: ${{ github.ref }}
          tag: "${{ env.ADDON_VERSION }}"
          artifacts: "${{ env.ZIP_FULL_NAME }}"
          artifactContentType: application/zip
          bodyFile: latest_changes.md
          allowUpdates: true
          makeLatest: true

      - name: Send to ESOUI
        uses: m00nyONE/esoui-upload@v1
        with:
          api_key: ${{ secrets.ESOUI_API_KEY }}
          addon_id: '2311'
          version: ${{ env.ADDON_VERSION }}
          zip_file: ${{ env.ZIP_FULL_NAME }}
          changelog_file: 'CHANGELOG.md'
```

### Notes
- Ensure your zip_file path is correct and points to a valid zipped addon file.
- Make sure your CHANGELOG.md is up-to-date before the run.

Enjoy automated ESOUI uploads! 🚀

