# A collection of reusable Github Actions and Workflows
A collection of reusable Github Actions and workflows used throughout GovWifi, so instead of maintaining 10 separate instances of code, we can reuse and maintain just the one copy.

## Ruby Updater Action
This action is designed to speed up the process of updating Ruby in our Docker containers.
It does this by trying to update the .ruby-version Dockerfile and GemfileLock, if this fails, it will remove the gemfile lock and recreate it.
We have about 10 repos that need constant updating, while this isn't 100% fool proof, it will fail if there are incompatibilities, it will at least help with the most common and easy updates.
This will create a branch and draft PR to review, fix and or merge as required.
It takes 3 parameters, 1 Required, 2 Optional
github-token: (REQUIRED) The github token (in secrets) that gives the bot permissions to create branch and PR's
main-branch: (OPTIONAL) The base branch name, as we are moving from `master` to `main` this will allow flexibility in the `base` branch name.
ruby-path: (OPTIONAL) The path were the Ruby version cnofig files are located, defaults to root. (no leading or training /)

use
```yaml
# .github/workflows/ruby-updater.yml
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
      - name: Check out code
        uses: actions/checkout@v4
      - name: run updater
        uses: govwifi/shared-actions-workflows/.github/actions/ruby-updater@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          main-branch: master
          ruby-path: acceptance_tests
```

## Ruby Updater Action for multi environments
This is similar to the above action, but split into separate actions, the check action will check the ruby version, and the file system to find all the ruby versions.
This is then sent to the ruby updater action, which will loop through all the environments and update the versions.
laster the commit action, commits the changes and raises a draft PR, [more about that in the Checks not run section below](#Checks-not-run)

It takes 2 parameters, 1 Required, 1 Optional
github-token: (REQUIRED) The github token (in secrets) that gives the bot permissions to create branch and PR's
main-branch: (OPTIONAL) The base branch name, as we are moving from `master` to `main` this will allow flexibility in the `base` branch name.

This is slight different from above, as in this one the script will automatically find the path

```yaml
name: Upgrade Ruby with multiple environments

on:
  workflow_dispatch:
  schedule:
    - cron: "0 0 15 * 0" # Runs monthly (in the middle to avoid any date clashes)

permissions:
  contents: write
  pull-requests: write

jobs:
  check-ruby-setup:
    runs-on: ubuntu-latest
    outputs:
      TARGET_RUBY_VERSION: ${{ steps.check-ruby.outputs.TARGET_RUBY_VERSION }}
      NEW_RUBY: ${{ steps.check-ruby.outputs.NEW_RUBY }}
      COMMIT_BRANCH: ${{ steps.setup-branch.outputs.COMMIT_BRANCH }}
      RUBY_PATHS: ${{ steps.check-ruby.outputs.RUBY_PATHS }}
    steps:

    - name: Check out code
      uses: actions/checkout@v4

    - name: Check Ruby Version
      id: check-ruby
      uses: govwifi/shared-actions-workflows/.github/actions/check-ruby

    - name: Setup github branch for Ruby Updates
      id: setup-branch
      if: ${{ steps.check-ruby.outputs.NEW_RUBY }} == 'true'
      uses: govwifi/shared-actions-workflows/.github/actions/setup-branch
      env:
        TARGET_RUBY_VERSION: ${{ steps.check-ruby.outputs.TARGET_RUBY_VERSION }}

  upgrade-ruby:
    runs-on: ubuntu-latest
    needs: [check-ruby-setup]
    steps:

    - name: Check out code
      uses: actions/checkout@v4

    -  name: Update Ruby Version
       env:
         TARGET_RUBY_VERSION: ${{ needs.check-ruby-setup.outputs.TARGET_RUBY_VERSION }}
         COMMIT_BRANCH: ${{ needs.check-ruby-setup.outputs.COMMIT_BRANCH }}
         NEW_RUBY: ${{ needs.check-ruby-setup.outputs.NEW_RUBY}}
         RUBY_PATHS: ${{ needs.check-ruby-setup.outputs.RUBY_PATHS}}
       ## Only update if the Ruby Version has changed.
       if: ${{ env.NEW_RUBY }} == 'true'
       uses: govwifi/shared-actions-workflows/.github/actions/update-ruby

    - name: Commit and Raise PR
      env:
         TARGET_RUBY_VERSION: ${{ needs.check-ruby-setup.outputs.TARGET_RUBY_VERSION }}
         COMMIT_BRANCH: ${{ needs.check-ruby-setup.outputs.COMMIT_BRANCH }}
         NEW_RUBY: ${{ needs.check-ruby-setup.outputs.NEW_RUBY}}
      uses: govwifi/shared-actions-workflows/.github/actions/commit-changes
      ## Only update if the Ruby Version has changed.
      if: ${{ env.NEW_RUBY }} == 'true'
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        main-branch: master
```

## Checks not run
PR's created by github tokens will not run further workflows, this is a design decision taken by Github.
See [Triggering Workflows](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#triggering-further-workflow-runs)
Workarounds include [PAT](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token), [SSH](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#push-using-ssh-deploy-keys) [Machine Account](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#push-pull-request-branches-to-a-fork) or [GitHub App](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#authenticating-with-github-app-generated-tokens)
At this stage these haven't been implemented by this action, we will need to make a decision on which direction to go if we want to do this


## Publish Artifact Workflow.
This workflow will version, release and publish an artifact onto github to be used in a calling resource, see [Shared Frontend](https://github.com/GovWifi/govwifi-shared-frontend), at this time this shared version hasn't been implemented, but here as an example of how a shared workflow could be used.
As this is an example, it's untested, no warranty it given or implied as to if it works.