# CommandBox Semantic Release

## Automatic version management, package publishing, and changelogs.

### Thanks and Prior Art

* [semantic-release](https://github.com/semantic-release/semantic-release)
* [How to Write an Open Source Javascript Library](https://egghead.io/courses/how-to-write-an-open-source-javascript-library) on [Egghead.io](https://egghead.io) by [Kent C. Dodds](https://kentcdodds.com).

### Why? / Benefits

* Stop thinking about what the next version should be.
* Releases happen on Continuous Integration (CI) servers. You don't even need
  to be at a computer. Releases from your smartphone just by merging a pull request.
* Automatic changelog generation.

The package was written due to a desire to be able to release a new version of a library
after merging a pull request without having to open my computer and run some scripts
locally. In fact, due to this module, I can simply squash and merge a pull request
and the appropriate version will be released automatically.

### Setup

CommandBox Semantic Release ships with powerful and sensible defaults for using
GitHub for remote source control, Travis CI for continuous integration, and ForgeBox
for package management. The setup steps here will take you through setting up this
workflow. If you have different workflow needs, please check out
[Extending CommandBox Semantic Release](#Extending-CommandBox-Semantic-Release) below.

#### Setup Steps

These setup steps only need to happen once per repository set up with CommandBox Semantic Release.

* Host your repository on GitHub.
* Have a [ForgeBox](https://www.forgebox.io) account
* [Activate the repository](https://docs.travis-ci.com/user/getting-started/#To-get-started-with-Travis-CI) on Travis CI.
* Configure your `box.json`.
* Add a `.travis.yml` file.
* Adding encrypted enviornment variables to Travis CI
    * `GH_TOKEN`
    * `TRAVIS_TOKEN`
    * `FORGEBOX_TOKEN`
* Follow the [conventional changelog](https://github.com/conventional-changelog/conventional-changelog) commit message format.

##### Host your repository on GitHub.

Check out GitHub's help site for [getting started](https://help.github.com/articles/create-a-repo/).

##### Have a [ForgeBox](https://www.forgebox.io) account

ForgeBox is the package manager for CFML. [Sign up for a free account](https://www.forgebox.io/security/registration) to get started.

##### Activate the repository on Travis CI.

Sign up for Travis CI and make sure your repository created above [is activated.](https://docs.travis-ci.com/user/getting-started/#To-get-started-with-Travis-CI)

##### Configure your `box.json`.

Make sure your `box.json` file has the current version of the package. Remember
that CommandBox Semantic Release will increment this version on every successful build.

> **Note**: You do **not** want any package scripts that will commit or push code to source control.
> This will be handled as part of CommandBox Semantic Release.

##### Add a `.travis.yml` file.

Below is a sample `.travis.yml` file that will test your code on multiple CF engines.
It assumes TestBox will be used in combination with a `/tests/runner.cfm` file to
run your project's tests.

> Technically, testing is not a requirement for CommandBox Semantic Release. If
> your project does not have tests you can run the semantic release process (currently
> found in the `after_success` block) as your `script`.

```
language: java
sudo: required
jdk:
- oraclejdk8
cache:
  directories:
  - "$HOME/.CommandBox"
env:
  matrix:
  - ENGINE=adobe@2016
  - ENGINE=adobe@11
  - ENGINE=lucee@5
  - ENGINE=lucee@4.5
before_install:
- sudo apt-key adv --keyserver keys.gnupg.net --recv 6DA70622
- sudo echo "deb http://downloads.ortussolutions.com/debs/noarch /" | sudo tee -a
  /etc/apt/sources.list.d/commandbox.list
install:
- sudo apt-get update && sudo apt-get --assume-yes install commandbox
- box install
before_script:
- box server start cfengine=$ENGINE port=8500
script:
- box testbox run runner='http://127.0.0.1:8500/tests/runner.cfm'
after_success:
  - box install commandbox-semantic-release
  - box config set endpoints.forgebox.APIToken=${FORGEBOX_TOKEN}
  - box semantic-release
notifications:
  email: false
```

##### Adding encrypted enviornment variables to Travis CI

You will need the following environment variables available to your build on Travis CI:

* `GH_TOKEN`

A GitHub personal access token with `repo` scopes. This is used to push changes
made on your CI server (like version updates and changelogs) back to GitHub.

* `TRAVIS_TOKEN`

A Travis CI personal access token. This is used to check the status of other jobs
in the build to avoid cutting a new release if only one job of a build fails. It
also prevents more than one job in a single build releasing a new version.

* `FORGEBOX_TOKEN`

A ForgeBox API token. This is used to publish a new release in ForgeBox. If the
package in question is a private package, the token will be needed to retrieve
the current version information as well.

Since these are API tokens, you want to make sure they are encrypted and hidden
in your builds so you don't accidentally leak your tokens.

Please refer to Travis CI's documentation for defining encrypted enviornment variables
[in your `.travis.yml` file](https://docs.travis-ci.com/user/environment-variables/#Defining-encrypted-variables-in-.travis.yml)
or here for defining them [in repository settings](https://docs.travis-ci.com/user/environment-variables/#Defining-Variables-in-Repository-Settings).
Either location is valid and will protect you from accidentally leaking your API Tokens.

##### Follow the conventional changelog commit message format.

Conventional Changelog (in a nutshull) means crafting your commit messages in
the [following format](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#commit-message-format):

```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```

This format has two features.

First, these commit messages let us know what change is next. Here are the high
level rules:

1. If the footer contains `BREAKING CHANGE:`, then a major release is issued.
2. If any of the changes have a type of `feat` (for feature), then a minor release is issued.
3. Otherwise, a patch release is issued.

Second, these commit messages allow you to easily create a nice, sectioned changelog for
your repository. If you do not follow the commit message format, this feature will
still work — your changelog will just not be formatted nicely.

### Usage

Now that the setup is done, it's time to use CommandBox Semantic Release!

By default, CommandBox Semantic Release will build a release only if certain
conditions are met. (These are specified in the `VerifyConditions` plugin):

1. The build is not due to a previous build. (This is determined using a special commit message.)
2. The build is running on Travis CI. (This prevents accidental builds on your local system.)
3. The build is not for a pull request.
4. The build is not for a tag.
5. The branch being built is for our target branch. (The default target branch is `master`.)
6. The job being built is the first job in the matrix. (This prevents multiple
   jobs from each cutting a release.)
7. All jobs passed their build.

This means that the easiest way to start our automated releases is to push the
`master` branch to GitHub. Then, assuming all your tests pass, you will see a
section at the end with the release information.

```
$ box install commandbox-semantic-release
$ box config set endpoints.forgebox.APIToken=${FORGEBOX_TOKEN}
Set endpoints.forgebox.APIToken = [secure]
$ box semantic-release
Polling.................
  ✓  Conditions verified
  ✓  Retrieved last version:  3.4.0
  ✓  Next release type:  minor
  ✓  Next version number:  3.5.0
  ✓  Release verified
  ✓  Release published
  ✓  Notes generated
  ✓  Source control updated
  ✓  Release publicized
```

This will automatically generate a `CHANGELOG.md` file and tag the release on GitHub.

```md
# 24-Jan-2018 — 04:28:37 UTC

### feat

* **Collection:** Add push, unshift, and splice methods ([e7c3efb](https://github.com/elpete/cfcollection/commit/e7c3efb50e0fe249cb531a9c3327724ab896b87d))

### fix

* **box.json:** Revert version to the actual version ([41f1185](https://github.com/elpete/cfcollection/commit/41f1185ddd439d06b361d5874962ff77c2b6458e))
```

### `dryRun`

The `dryRun` flag can be used to see the next version of your application without
actually publishing a new release.

### Extending CommandBox Semantic Release

Though CommandBox Semantic Release is built to work out of the box on
GitHub and Travis CI with sensible defaults, it is incredibly
configurable and extensible.

#### Setting CommandBox Semantic Release settings

```
after_success:
  - box install commandbox-semantic-release
  - box config set endpoints.forgebox.APIToken=${FORGEBOX_TOKEN}
  - box config set modules.settings.commandbox-semantic-release.versionPrefix = ""
  - box config set modules.settings.commandbox-semantic-release.plugins.GenerateNotes = "MyCustomNotesGenerator@commandbox-semantic-release-custom-notes"
  - box semantic-release
```

### FAQ

#### I don't want to release on every commit. How do I batch commits for a later release?

An easy way to accomplish this is to have your pull requests merge to a branch different
than the CommandBox Semantic Release `targetBranch` (which defaults to `master`).

For instance, if you have people merge their changes to `development` only when
you merge `development` into `master` will a new release be cut. The new release
will contain all the changes since the last version and increment the version number
appropriately.

Please note, though, that one of the goals and features of CommandBox Semantic Release
is to free you from needing to think about when to cut a release. The philosophy is
to merge in any completed feature or bug fix and let semantic versioning do its thing.

#### How do I force users to use the special commit message syntax?

First off, you don't have to. GitHub allows you, as a maintainer, to squash and merge a pull request.
As part of this process you will have the chance to change the commit message.

If you would like to enforce this convention for your team or others who have direct
commit access (so they are not going through a pull request), you can use tools
like [Commitizen](https://github.com/commitizen/cz-cli) to generate the commit
message for you or [CommandBox Githooks](https://www.forgebox.io/view/commandbox-githooks)
to verify the commit message format on `preCommit`.

#### I don't use GitHub, Travis CI, ForgeBox, etc. Can I still use this package?

Absolutely! You'll need to bring your own plugins, but check out
[Extending CommandBox Semantic Release](#Extending-CommandBox-Semantic-Release)
above to find out how.

#### I want to cut a manual release (because reasons). Will I be able to use CommandBox Semantic Release after that?

Yes. The next version number is, by default, based on commits between your
current `HEAD` and the last version found on ForgeBox. As long as ForgeBox
always has the correct version, you should be fine.