# 3. Continuous Integration - Bitrise

Date: 2020-04-22

## Status

Accepted

## Problem Description

We want a central tool to handle automatic builds of our apps.

## What We Are Evaluating

[Bitrise.io](https://www.bitrise.io/)

It's been approved by Procurement, and the [REDACTED] team has been using it.

## Alternatives

* [AppCenter](https://appcenter.ms/)
* [CircleCI](https://circleci.com/)

## Evaluation Plan

### Acceptance Criteria

* Must build and test our apps automatically when we create a pull request
* Must build, test, and sign our apps automatically when we merge to master
* Should be able to set version numbers on our apps based on the date
* Should have integration options, so that it can send an email, teams message, etc. based on results.
* Should be able to publish test reports.

### Duration of Evaluation

Indicate how long we will evaluate this technology before deciding to roll forward or roll back.

* **Start Date:** April 22, 2020
* **Expiration Date:** August 17, 2020

### Roll-forward Plan

Rolling forward - we have no other CI solution currently running, so we'd just continue using Bitrise.

### Roll-backward Plan

Rolling backward would just involve discontinuing using Bitrise, but we'd need to find another solution, since we don't want to have to run builds by hand.

## Evaluation Log

### 2020-04-22

Evaluation Started

### 2020-04-23

#### Disabiling Parallel Builds

For iOS, we're using Carthage for dependency management and we've got multiple projects within the same workspace. When we tried running the project in Bitrise with our usual scheme, we ended up getting errors that looked like this:

```
fatal error: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/lipo: Input file: /Users/vagrant/Library/Developer/Xcode/DerivedData/[[REDACTED]]/Build/Products/Debug-iphonesimulator/Cuckoo.framework.dSYM/Contents/Resources/DWARF/Cuckoo changed since opened (Undefined error: 0)
```

To fix this, we created another scheme, called `Bitrise`, which is the same as our main `[REDACTED]App` scheme, but it turns off the option to `Parallelize Build`. We haven't really noticed a performance difference with this change.

#### Cuckoo Generator

We've been evaluating Cuckoo for mocking. Since Swift doesn't support reflection, mocking is done by reading source and generating Swift files that contain the mocks. The code that generates the mocks is a separate executable file.

Normally the generator is either built when needed, or downloaded from Github.

Building the generator during a Bitrise build didn't work:

```
No Cuckoo Generator found.
~/git/ios/Carthage/Checkouts/Cuckoo ~/git/ios/Presenters
Building...
~/git/ios/Carthage/Checkouts/Cuckoo/Generator ~/git/ios/Carthage/Checkouts/Cuckoo
/Users/vagrant/git/ios/Carthage/Checkouts/Cuckoo/Generator: error: error: accessing build database "/Users/vagrant/git/ios/Carthage/Checkouts/Cuckoo/Generator/.build/manifest.db": disk I/O error
Build seems to have failed for some reason. Please file an issue on GitHub.
```

We also tried telling it to download it instead, but it couldn't figure out the download URL:

```
No Cuckoo Generator found.
~/git/ios/Carthage/Checkouts/Cuckoo ~/git/ios/DataAccess
Downloading latest version...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100   248  100   248    0     0   3169      0 --:--:-- --:--:-- --:--:--  3179
Error: Failed to fetch download URL for the Cuckoo Generator.
Command PhaseScriptExecution failed with a nonzero exit code
```

It's not clear why it was unable to build the generator or fetch the download URL, since these work on our local development workstations.

We worked around this by updating our `initialize.sh` script to fetch the file manually during initialization instead of automatically when the script detects that the generator isn't present.

```
curl -Lo Carthage/Checkouts/Cuckoo/cuckoo_generator https://github.com/Brightify/Cuckoo/releases/download/1.3.2/cuckoo_generator
chmod 744 Carthage/Checkouts/Cuckoo/cuckoo_generator
```

***Update:*** We rejected Cuckoo in favor of Mockingbird, so this is not an issue now.

#### Android

We knew that we'd have one particular problem with Android builds on Bitrise (or any remote CI server) - it requires one private dependency, the Design Tokens library. We made that available via Github Packages, and the build was green right away.

Github Packages probably isn't the right long-term solution. We might explore Azure DevOps since the [REDACTED] team is using that.

But in terms of registering a new Android app on Bitrise, it has worked flawlessly so far.

***Update:*** As a company, we're moving toward _Artifactory_, and debit-mobile-app was able to get in early to set up its dependencies to work remotely with Bitrise.

### 2020-04-28

We've created two workflows for each app:

* **pr-workflow** - Runs whenever a pull request is opened
* **primary** - Creates a debug build and publishes it for QA

There's a third, called **deploy**, which we haven't actually done anything with.

#### Pull Request State

We've configured Bitrise to run both the iOS and Android projects whenever a pull request is opened. However, the pull request check on Github links only to the corresponding build on _iOS_. It's not clear yet whether a failure on Android would cause the check to fail.

#### Merges to Master Trigger Both Projects

Even if the source is only changed for one platform, a merge to master triggers both platforms to build. An even if we only update documentation, it still causes a build for both platforms, despite not actually affecting anything in production.

Bitrise does support a feature called [Selective Builds](https://blog.bitrise.io/build-parts-of-your-mono-repo-separately) which can choose to only run a build when the change set matches a file path. For example, if the code set only contained changes to files under `/ios`, then it would know to run the iOS build and not the Android build. This feature requires a _Service Credential User_, which is basically a Github user token. In other words, Github _Deploy Keys_ won't work. Since that Service Credential User gives Bitrise the same level of access as the user's account, we would need to create a separate, dedicated email address, create a Github account for it, and then register that user's token with Bitrise.

***Update:*** We've been able to create the Service Credential User, and now only the correct platform is built.

