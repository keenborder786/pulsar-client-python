<!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.

-->

# Release Process

This page contains instructions for Pulsar committers on how to perform a release for the Pulsar Python client.

## Preparation

> **Note**
>
> The term `major/minor releases` used throughout this document is defined as follows:
> - Major releases refer to feature releases, such as 3.0.0, 3.1.0, and so on.
> - Minor releases refer to bug-fix releases, such as 3.0.1, 3.0.2, and so on.
>
> This guide use `X.Y.Z` or `X.Y` to represent the actual versions like `3.0.0` or `3.0`.

For major releases, you should create a new branch named `branch-X.Y` once all PRs with the X.Y.0 milestone are merged. If some PRs with the X.Y.0 milestone are still working in progress and might take much time to complete, you can move them to the next milestone if they are not important. In this case, you'd better notify the author in the PR.

For minor releases, if there are no disagreements, you should cherry-pick all merged PRs with the `release/X.Y.Z` labels into `branch-X.Y`. After these PRs are cherry-picked, you should add the `cherry-picked/branch-X.Y` labels.

Sometimes some PRs cannot be cherry-picked cleanly, you might need to create a separate PR and move the `release/X.Y.Z` label from the original PR to it. In this case, you can ask the author to help create the new PR.

For PRs that are still open, you can choose to delay them to the next release or ping other committers to review so that they can be merged.

## Requirements

If you haven't already done it, [create and publish the GPG key](https://pulsar.apache.org/contribute/create-gpg-keys/) to sign the release artifacts.

## Upgrade the C++ client dependency

During the development, the C++ client dependency might be downloaded from an unofficial release. But when releasing the Python client, the dependency must be downloaded from an official release. You should modify the base url in [dep-url.sh](./build-support/dep-url.sh).

Example: https://github.com/apache/pulsar-client-python/pull/62 

## Cut the candidate release

After all necessary PRs are cherry-picked to `branch-X.Y`, you should cut the release by pushing a tag.

```bash
git checkout branch-X.Y
git push origin branch-X.Y
# N starts with 1
git tag vX.Y.Y-candidate-N
git push origin vX.Y.Y-candidate-N
```

Then, [create a new milestone](https://github.com/apache/pulsar-client-python/milestones/new) for the next major release.

## Sign and stage the candidate release

After a tag is pushed, a workflow will be triggered to build the Python wheels in GitHub Actions. You can find it in https://github.com/apache/pulsar-client-python/actions/workflows/ci-build-release-wheels.yaml

For example, https://github.com/apache/pulsar-client-python/actions/runs/3709463737 is the workflow of `v3.0.0-candidate-3`, the workflow id is 3709463737. Remember the workflow id, which will be passed as an argument of `stage-release.sh` in the following step. Once the workflow is completed, the wheels will be available for downloading.

Generate a GitHub token by following the guide [here](https://github.com/settings/tokens). The `repo` and `workflow` checkboxes must be selected. Then export the token as the environment variable:

```bash
export GITHUB_TOKEN=<your-token>
```

Edit `~/.gnupg/gpg.conf` to ensure the default GPG key is from your Apache mail (`<your-name>@apache.org`):

```
default-key <key fingerprint>
```

Make sure `curl`, `jq`, `unzip`, `gpg`, `shasum` commands are available. Then you should run the following commands in an empty directory:

```bash
svn co https://dist.apache.org/repos/dist/dev/pulsar pulsar-dist-dev-keys --depth empty
cd pulsar-dist-dev-keys
svn mkdir pulsar-client-python-X.Y.Z-candidate-N && cd pulsar-client-python-X.Y.Z-candidate-N
# PROJECT_DIR is the directory of the pulsar-client-python repository 
$PROJECT_DIR/build-support/stage-release.sh vX.Y.Z-candidate-N $WORKFLOW_ID
svn add *
svn ci -m "Staging artifacts and signature for Python client X.Y.Z-candidate-N"
```

## Start the vote for the candidate

Send an email to dev@pulsar.apache.org to start the vote for the candidate:

```
To: dev@pulsar.apache.org
Subject: [VOTE] Pulsar Client Python Release X.Y.Z Candidate N

This is the third release candidate for Apache Pulsar Client Python,
version X.Y.Z.

It fixes the following issues:
https://github.com/apache/pulsar-client-python/milestone/<milestone-id>?closed=1

*** Please download, test and vote on this release. This vote will
stay open for at least 72 hours ***

Python wheels:
https://dist.apache.org/repos/dist/dev/pulsar/pulsar-client-python-X.Y.Z-candidate-N/

The supported python versions are 3.7, 3.8, 3.9 and 3.10. The
supported platforms and architectures are:
- Windows x86_64 (windows/)
- glibc-based Linux x86_64 (linux-glibc-x86_64/)
- glibc-based Linux arm64 (linux-glibc-arm64/)
- musl-based Linux x86_64 (linux-musl-x86_64/)
- musl-based Linux arm64 (linux-musl-arm64/)
- macOS universal 2 (macos/)

The tag to be voted upon: vX.Y.Z-candidate-N
(<commit-id>)
https://github.com/apache/pulsar-client-python/releases/tag/vX.Y.Z-candidate-N

Pulsar's KEYS file containing PGP keys you use to sign the release:
https://dist.apache.org/repos/dist/dev/pulsar/KEYS

Please download the Python wheels and follow the README to test.
```

Once there are at least 3 binding +1s, the vote will be ready to close and you can continue the following steps. If there is something wrong with the candidate, you need to fix it and repeat the steps from the **Cut the next candidate** section again.

## Move main branch to next version

You need to move the main version to the next iteration.

```bash
git checkout main
sed -i.bak 's/X.Y.0a1/X.Y+1.0a1/' pulsar/__about__.py
git add pulsar/__about__.py
git commit -m "Bumped version to X.Y.0a1"
```

Since this needs to be merged into main, you need to follow the regular process and create a Pull Request on GitHub.

## Promote the release

Ask a PMC member to promote the release:

```bash
svn move -m "Release Apache Pulsar Client Python X.Y.Z" \
  https://dist.apache.org/repos/dist/dev/pulsar/pulsar-client-python-X.Y.Z-candidate-N \
  https://dist.apache.org/repos/dist/release/pulsar/pulsar-client-python-X.Y.Z
```

## Upload the wheels to PyPI

1. You need to create an account on PyPI: https://pypi.org/account/register/
2. Ask anyone that has been a release manager before to add you as a maintainer for [pulsar-client](https://pypi.org/manage/project/pulsar-client/releases/) on PyPI
3. Once you have completed the following steps in this section, you can check if the wheels are uploaded successfully in Download files. Remember to switch to the correct version in Release history).

Then, upload the wheels to PyPI:

```bash
sudo python3 -m pip install twine
./build-support/upload-pypi.sh https://dist.apache.org/repos/dist/release/pulsar/pulsar-client-python-X.Y.Z
```

To verify the wheels have been uploaded successfully, you can try installing the wheel:

```bash
python3 -m pip install pulsar-client==X.Y.Z
```

## Write release notes

Push the official tag:

```bash
git checkout vX.Y.Z-candidate-N
git tag vX.Y.Z
git push origin vX.Y.Z
```

Then, [create a release](https://github.com/apache/pulsar-client-python/releases/new). Choose the `vX.Y.Z` tag and click the `Generate release notes` button to generate the release note automatically. Here is an example release note: https://github.com/apache/pulsar-client-python/releases/tag/v3.0.0

Then, create a PR in [`pulsar-site`](https://github.com/apache/pulsar-site) repo to update the website. Here is an example: https://github.com/apache/pulsar-site/pull/343

## Generate the API documents

For minor releases, skip this section. For major releases, you should generate the HTML files into the [`pulsar-site`](https://github.com/apache/pulsar-site) repo:

```bash
git clone git@github.com:apache/pulsar-client-python.git
cd pulsar-client-python
git checkout vX.Y.0
# It's better to replace this URL with the URL of your own fork
git clone git@github.com:apache/pulsar-site.git
sudo python3 -m pip install pydoctor
pydoctor --make-html \
  --html-viewsource-base=https://github.com/apache/pulsar-client-python/tree/vX.Y.0 \
  --docformat=numpy --theme=readthedocs \
  --intersphinx=https://docs.python.org/3/objects.inv \
  --html-output=./pulsar-site/site2/website-next/static/api/python/X.Y.x \
  pulsar-client-python/pulsar
cd pulsar-site
git checkout -b py-docs-X.Y
git add .
git commit -m "Generate Python client X.Y.0 doc"
git push origin py-docs-X.Y
```

Then open a PR like: https://github.com/apache/pulsar-site/pull/342

## Announce the release

Use your Apache email account (`user@apache.org`) to send an email like:

```
To: dev@pulsar.apache.org, users@pulsar.apache.org, announce@apache.org
Subject: [ANNOUNCE] Apache Pulsar Client Python X.Y.Z released

The Apache Pulsar team is proud to announce Apache Pulsar Client
Python version X.Y.Z.

Pulsar is a highly scalable, low latency messaging platform running on
commodity hardware. It provides simple pub-sub semantics over topics,
guaranteed at-least-once delivery of messages, automatic cursor management for
subscribers, and cross-datacenter replication.

You can download the source code and the Python wheels in:
https://archive.apache.org/dist/pulsar/pulsar-client-python-X.Y.Z/

The Python wheels were uploaded to PyPI as well so that they can be
installed by `pip install pulsar-client==X.Y.Z`.

Release Notes are at:
https://pulsar.apache.org/release-notes/versioned/client-python-X.Y.Z/

We would like to thank the contributors that made the release possible.

Regards,

The Pulsar Team
```

Send the email in plain text mode since the announce@apache.org mailing list will reject messages with text/html content. In Gmail, there's an option to set Plain text mode in the ⋮/ More options menu.
