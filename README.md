# Reusable Workflows
A collection of workflows used throughout GovWifi so instead of maintaining 10 separate instances of code, we can reuse and maintain just the one copy.

## Ruby Updater
This workflow is designed to speed up the process of updating Ruby in our Docker containers.
It does this by trying to update the .ruby-version Dockerfile and GemfileLock, if this fails, it will remove the gemfile lock and recreate it.
We have about 10 repos that need constant updating, while this isn't 100% fool proof, it will fail if there are incompatibilities, it will at least help with the most common and easy updates.
This will create a branch and PR to review, fix and or merge as required.

use
```
# .github/workflows/upgrade-ruby.yml
name: Upgrade Ruby

on:
  workflow_dispatch:
  schedule:
    - cron: "0 0 15 * 0" # Runs monthly (in the middle to avoid any date clashes)

permissions:
  contents: write
  pull-requests: write

jobs:
  upgrade-ruby:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: govwifi/.github/workflow/update-ruby
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```