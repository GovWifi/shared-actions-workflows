# A collection of reusable Github Actions and Workflows
A collection of reusable Github Actions and workflows used throughout GovWifi, so instead of maintaining 10 separate instances of code, we can reuse and maintain just the one copy.


## Ruby Updater Action for multi environments
This action is designed to speed up the process of updating Ruby in our Docker containers.
It does this by trying to update the .ruby-version Dockerfile and Gemfile Lock, if this fails, it will remove the gemfile lock and recreate it.
We have about 10 repos that need constant updating, while this isn't 100% fool proof, it will fail if there are incompatibilities, it will at least help with the most common and easy updates.
This will create a branch and PR to review, to run the lint and test workflows, close and reopen PR, as Github Tokens can't be used to trigger other workflows due to Imposted Github Restriction,
 fix and or merge as required.

This workflow has several actions, check job will check the ruby version then search the repo to find all the ruby versions.
This is then sent to the ruby updater job, which will loop through all the environments and update the versions.
later the commit action, commits the changes and raises a PR, [more about that in the Checks not run section below](#Checks-not-run)

It takes 2 parameters, 1 Required, 1 Optional
github-token: (REQUIRED) The github token (in secrets) that gives the bot permissions to create branch and PR's
main-branch: (OPTIONAL) The base branch name, as we are moving from `master` to `main` this will allow flexibility in the `base` branch name.

# .github/workflows/ruby-updater.yml
```yaml
name: Upgrade Ruby in multiple environments

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
      uses: govwifi/shared-actions-workflows/.github/actions/ruby-version-check@main

    - name: Setup github branch for Ruby Updates
      id: setup-branch
      if: ${{ steps.check-ruby.outputs.NEW_RUBY }} == 'true'
      uses: govwifi/shared-actions-workflows/.github/actions/ruby-setup-branch@main
      env:
        TARGET_RUBY_VERSION: ${{ steps.check-ruby.outputs.TARGET_RUBY_VERSION }}

  upgrade-ruby:
    runs-on: ubuntu-latest
    needs: [check-ruby-setup]
    steps:

    - name: Check out code
      uses: actions/checkout@v4

    - name: Update Ruby Version
      env:
        TARGET_RUBY_VERSION: ${{ needs.check-ruby-setup.outputs.TARGET_RUBY_VERSION }}
        COMMIT_BRANCH: ${{ needs.check-ruby-setup.outputs.COMMIT_BRANCH }}
        NEW_RUBY: ${{ needs.check-ruby-setup.outputs.NEW_RUBY}}
        RUBY_PATHS: ${{ needs.check-ruby-setup.outputs.RUBY_PATHS}}
      ## Only update if the Ruby Version has changed.
      if: ${{ env.NEW_RUBY }} == 'true'
      uses: govwifi/shared-actions-workflows/.github/actions/ruby-updater-multi-env@main

    - name: Commit and Raise PR
      env:
         TARGET_RUBY_VERSION: ${{ needs.check-ruby-setup.outputs.TARGET_RUBY_VERSION }}
         COMMIT_BRANCH: ${{ needs.check-ruby-setup.outputs.COMMIT_BRANCH }}
         NEW_RUBY: ${{ needs.check-ruby-setup.outputs.NEW_RUBY}}
      uses: govwifi/shared-actions-workflows/.github/actions/ruby-commit-changes@main
      ## Only update if the Ruby Version has changed.
      if: ${{ env.NEW_RUBY }} == 'true'
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        main-branch: master

```

## Repos that use this.
[Dev Docs:](https://github.com/GovWifi/govwifi-dev-docs/)
[Product page:](https://github.com/GovWifi/govwifi-product-page/)
[Smoke Tests:](https://github.com/GovWifi/govwifi-smoke-tests/)
[User Signup API:](https://github.com/GovWifi/govwifi-user-signup-api/)
[Auth API:](https://github.com/GovWifi/govwifi-authentication-api/)
[Admin:](https://github.com/GovWifi/govwifi-admin/)
[Safe Restarter:](https://github.com/GovWifi/govwifi-safe-restarter/)
[Acceptance Tests:](https://github.com/GovWifi/govwifi-acceptance-tests/pull/195)
[FrontEnd:](https://github.com/GovWifi/govwifi-frontend/)
[Logging API:](https://github.com/GovWifi/govwifi-logging-api/)

## Known issues
With the above script, to get the Ruby version a Curl to the github api is performed, sometimes this can fail due to network errors, hopefully running this at night might help, but if it does fail, it stops the build.
Maybe a future fix would be to add a retry method into it, a battle for another day!

## Checks not run
PR's created by github tokens will not run further workflows, this is a design decision taken by Github.
See [Triggering Workflows](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#triggering-further-workflow-runs)
Workarounds include [PAT](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token), [SSH](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#push-using-ssh-deploy-keys) [Machine Account](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#push-pull-request-branches-to-a-fork) or [GitHub App](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#authenticating-with-github-app-generated-tokens)
At this stage these haven't been implemented by this action, we will need to make a decision on which direction to go if we want to fully automate this.


## Publish Artifact Workflow.
This workflow will version, release and publish an artifact onto github to be used in a calling resource, see [Shared Frontend](https://github.com/GovWifi/govwifi-shared-frontend), at this time this shared version hasn't been implemented, but here as an example of how a shared workflow could be used.
An example of this can be found in the .github/workflow directory
This is used on the SharedFrontend Repo.
