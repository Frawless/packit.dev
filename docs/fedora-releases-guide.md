---
title: How to do Fedora releases with Packit
sidebar_position: 8
---
# How to do Fedora releases with Packit

Let's split the release process into single steps:
1. [New upstream release](#new-upstream-release)
2. [Upload archive to lookaside cache](#upload-archive-to-lookaside-cache)
3. [Update dist-git content](#update-dist-git-content)
4. [Koji builds](#koji-build-job)
5. [Bodhi updates](#bodhi-update-job)

Doing Fedora releases with Packit means utilising these jobs:
1. [`propose_downstream`](#propose-downstream-job) or [`pull_from_upstream`](#pull-from-upstream-job)
2. [`koji_build`](#koji-build-job)
3. [`bodhi_update`](#bodhi-update-job)

Every job takes care of a different part of the release process.

## Propose downstream or pull from upstream

There are two jobs that can help you to get your new release to Fedora.
They differ in the way they are triggered and configured but share the implementation.

The push workflow is configured and started in the upstream repository,
unlike the pull workflow that is configured in dist-git.

Here are the key differences between the two:

<table>
<tr>
<th></th>
<th>propose-downstream</th>
<th>pull-from-upstream</th>
</tr>

<tr>
<th>Packit Service</th>
<td><p>Have a <code>.packit.yaml</code> in <b>upstream</b> repo:</p>


```yaml
jobs:
- job: propose-downstream:
    ...
```

  <p>Triggered by a new release in <b>upstream project</b>.</p>
  <p>It creates <i>dist-git</i> pull requests with the content of the release.</p>
</td>
<td><p>Have a <code>.packit.yaml</code> in <b>dist-git</b> repo (main or rawhide branch):</p>

```yaml
jobs:
- job: pull-from-upstream:
    ...
```

  <p>Triggered by a new release in <b>upstream project</b>.</p>
  <p>It creates <i>dist-git</i> pull requests with the content of the release and the packit config taken from dist-git main/rawhide branch.</p>
</td>
</tr>

<tr>
<th>Packit CLI</th>
<td><p>Have a <code>.packit.yaml</code> in <b>upstream</b> repo, clone repo and run:</p>

  ```
  packit propose-downstream
  ```

  <p>It creates <i>dist-git</i> pull requests with the content of the release and the packit config taken from local clone.</p>
</td>
<td><p>Have a <code>.packit.yaml</code> in <b>dist-git</b> repo, clone repo and run:</p>

  ```
  packit pull-from-upstream
  ```

  <p>It creates <i>dist-git</i> pull requests with the content of the release and the packit config taken from local clone.</p>
</td>
</tr>
</table>

### Resolving specfile and version

<table>
<tr>
<th></th>
<th>propose-downstream</th>
<th>pull-from-upstream</th>
</tr>

<tr>
<th>Packit Service</th>
<td>
  <p>Version is retrieved from <b>upstream project release event</b>.</p>
  <p>Specfile is retrieved from <b>upstream repo</b>.</p>
</td>
<td>
  <p>Version is retrieved from <b>https://release-monitoring.org event</b>.</p>
  <p>Specfile is retrieved from <b>dist-git repo</b>.</p>
</td>
</tr>

<tr>
<th>Packit CLI</th>
<td>
  <p>Version is retrieved from the <b>latest upstream project release tag</b> if not <b>specified</b>.</p>
  <p>Specfile is retrieved from the <b>upstream repo</b> unless the <code>--local-project</code> option is used.</p>
</td>
<td>
  <p>Version is retrieved from the <b>latest upstream project release tag</b> if not <b>specified</b>.</p>
  <p>Specfile is retrieved from the <b>local dist-git repo clone</b>.</p>
</td>
</tr>
</table>

## Propose downstream job
For enabling the propose downstream job, you need to have
[Packit Service installed](/docs/guide/#1-set-up-packit-integration)
and have a `propose_downstream` job in the configuration file for the given upstream repository
(this job is also run by default if there is no `jobs` section
in the configuration, see [jobs configuration](/docs/configuration/#packit-service-jobs)).
The [propose_downstream job](/docs/configuration/upstream/propose_downstream) should be then configured like this:

```yaml
jobs:
- job: propose_downstream
  trigger: release
  dist_git_branches:
    - main
```
You can adjust the `dist_git_branches` field to include the
dist-git branches you want to update and also utilise [aliases](/docs/configuration/#aliases) 
instead of using hardcoded versions.

#### New upstream release
The process of releasing a new version starts in the upstream repository by creating a 
new upstream release. Packit gets the information about the newly created release (not a git tag) from GitHub/GitLab,
loads the config from the release commit and if there is a `propose_downstream` job
defined, the workflow begins. If you want to restrict what releases with corresponding tags Packit should react on, 
you can utilise the configuration options [`upstream_tag_include`](/docs/configuration/#upstream_tag_include) and
[`upstream_tag_exclude`](/docs/configuration/#upstream_tag_exclude).

#### Upload archive to lookaside cache
The upstream archive needs to be downloaded by Packit first and then uploaded to the lookaside cache.
By default, Packit downloads sources defined in the specfile that contain URLs.
You can override these URLs via [`sources`](/docs/configuration#sources) configuration key.

For Python packages, you can use a 
[GitHub action](https://packaging.python.org/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows)
([example setup of Packit itself](https://github.com/packit/packit/blob/main/.github/workflows/pypi-publish.yml))
that automatically builds and uploads the archive to PyPI 
on each new release. Then during propose downstream, Packit tries to download the archive from the provided URL.
If the download fails because the upstream archive is not available at the time of running the job, 
the job is scheduled to be retried later.

If you don't want Packit to upload sources to lookaside cache before the pull request is opened,
set `upload_sources` to `false`. By disabling the upload, you need to take care of this yourself
and the builds triggered by dist-git CI will fail because of the missing archive in the lookaside cache.

#### Update dist-git content
After saving the archive in the lookaside cache,
Packit updates the dist-git content (mainly `sources` file and spec file) via pull requests for the specified branches.
You can configure which files in the upstream repo should be copied to dist-git during an update
via [`files_to_sync`](/docs/configuration/#files_to_sync) configuration key.

The version in the spec file is set to the version that Packit gets from the upstream tag 
corresponding to the release that triggered the job. If the version and tag differ, 
you can specify the [`upstream_tag_template`](/docs/configuration/#upstream_tag_template)
configuration option so that Packit can extract the correct version.

If you use [`copy_upstream_release_description: true`](/docs/configuration/#copy_upstream_release_description),
the changelog entry will use the GitHub/GitLab release description field.
(Just make sure the formatting is compatible with spec file.
E.g. use `-` instead of `*` for lists to not create multiple changelog entries.)
There is also [`sync_changelog`](/docs/configuration/#sync_changelog) configuration option to enable syncing 
the whole changelog.
You can also utilize a [custom `changelog-entry` action](/docs/configuration/actions#syncing-the-release).

        actions:
          changelog-entry:
            - bash -c 'echo "- New release ${PACKIT_PROJECT_VERSION}"'



Be aware that Packit does not sign-off its commits so it can't open pull requests
if the ` Enforce signed-off commits in pull-request` option is set in the dist-git project settings.

During proposing a new update, you will get updates of the job status via commit statuses/checks
on the release commit. These will provide links to our dashboard where you can find all the information about 
the job including the logs. You can also check all propose downstream runs in 
[this view](https://dashboard.packit.dev/jobs/propose-downstreams).

![Dashboard view for propose downstreams](img/fedora-releases-guide/propose-downstream-dashboard.png)

After Packit successfully creates the dist-git pull requests, 
it's on downstream CI systems and maintainer(s) to check the changes and merge
the pull requests.

### Retriggering
Users with write or admin permissions to the repository can retrigger an
update via a comment in any open issue in the upstream repository:

    /packit propose-downstream

## Pull from upstream job

:::tip New feature

Starting January 2023, we have provided a new way to get fresh
upstream releases in Fedora Linux.

:::

The [`pull_from_upstream` job](/docs/configuration/downstream/pull_from_upstream) is
defined in dist-git only and provides the `propose_downstream`
functionality. This means that Packit doesn't need to be set up in the
upstream project: everything is configured in Fedora dist-git. So when a new
upstream release happens and
[release-monitoring.org](https://release-monitoring.org/) detects it, you'll
get dist-git pull requests with it automatically. 

Bodhi updates created by the [`bodhi_update` job](/docs/configuration/downstream/pull_from_upstream) as well as [automatic Bodhi updates](https://fedora-infra.github.io/bodhi/6.0/user/automatic_updates.html) will close the Bugzilla opened by 
the Upstream Release Monitoring automatically when they reach stable.
Packit adds the Bugzillas numbers to the commit message and the changelog in this form `- Resolves rhbz#xz`.
There is also an env variable with the list of bugs to be closed
`PACKIT_RESOLVED_BUGS` that you can use in the case you want to customize the changelog creation through an action
as shown below.

If you want to restrict what releases with corresponding tags Packit should react on, 
you can utilise the configuration options [`upstream_tag_include`](/docs/configuration/#upstream_tag_include) and
[`upstream_tag_exclude`](/docs/configuration/#upstream_tag_exclude).

It is necessary to set the [`upstream_project_url`](/docs/configuration/#upstream_project_url) (upstream project Git repository URL) configuration option. However, upstream tarball URL is taken from the spec file or from [`sources`](/docs/configuration/#sources) (see below).
For customization of the job, you may need to define additional configuration options, most commonly:
- If the version from release monitoring and Git tag differ, 
you should specify the [`upstream_tag_template`](/docs/configuration/#upstream_tag_template).
- You can configure which files (if any) in the upstream repo should be copied to dist-git during an update
via [`files_to_sync`](/docs/configuration/#files_to_sync) configuration key.
- By default, Packit downloads sources defined in the spec file that contain URLs.
You can override these URLs via [`sources`](/docs/configuration#sources) configuration key.
- You may utilise some of the [actions](/docs/configuration/actions/#syncing-the-release)
for overriding the Packit default behaviour, for example:
  - for the changelog entry generation, if you do not want the default `git log` output, you can use your own command(s):
  
        changelog-entry:
          - bash -c 'echo "- New release ${PACKIT_PROJECT_VERSION}"'
          - bash -c '[ -z "$PACKIT_RESOLVED_BUGS" ] || echo ${PACKIT_RESOLVED_BUGS} | tr " " "\n" | sed "s/^/- Resolves /"'
  
  - for a custom commit message for commit created by Packit:

        commit-message:
          - bash -c 'echo -e "Rebase to new upstream release ${PACKIT_PROJECT_VERSION}\n"'
          - bash -c '[ -z "$PACKIT_RESOLVED_BUGS" ] || echo ${PACKIT_RESOLVED_BUGS} | tr " " "\n" | sed "s/^/- Resolves /"'


You can check all the job runs with details and logs in [this view](https://dashboard.packit.dev/jobs/pull-from-upstreams).
You can also configure a repository where we should
open issues in case of errors during the job via [`issue_repository`](/docs/configuration#issue_repository) 
configuration key.


![Dashboard view for pull_from_upstream](img/fedora-releases-guide/pull-from-upstream-dashboard.png)

### First setup
If you are interested in this functionality and want to try it out, we recommend triggering the job 
first time from a pull request to make sure Packit is correctly configured.

#### If there is a pending release
If there is a new release pending for your package (bugzilla has been opened by [release-monitoring.org](https://release-monitoring.org/) but no rebase done in dist-git yet), do the following:

- create a `rawhide`-based pull request with Packit configuration defining the [`pull_from_upstream` job](/docs/configuration/downstream/pull_from_upstream)
  - we recommend firstly setting the `dist_git_branches` for the job to one branch only (e.g. `fedora-rawhide`)
- comment `/packit pull-from-upstream --with-pr-config` on the pull request
- check the [dashboard](https://dashboard.packit.dev/jobs/pull-from-upstreams)
- if everything went well, review the pull request(s) in your dist-git repository created by Packit
- if you are happy with the results 
  - you can update the `dist_git_branches` to include the list of desired branches and trigger the syncing for all branches (using the same comment `/packit pull-from-upstream --with-pr-config`) 
  - merge the pull request 

#### If there is no pending release
If there is no pending release and your package has been rebased at least once in the past, you can still try the job using a new testing branch:

- create a branch pointing to a commit before the last rebase, name it e.g. `packit-test` and push it (directly to dist-git, not to your fork)
- create a `rawhide`-based pull request with Packit configuration defining the [`pull_from_upstream` job](/docs/configuration/downstream/pull_from_upstream)
- in the configuration, set the `dist_git_branches` option of the `pull_from_upstream` job to the name of the testing branch
- comment `/packit pull-from-upstream --with-pr-config` on the pull request
- check the [dashboard](https://dashboard.packit.dev/jobs/pull-from-upstreams)
- if everything went well, review the pull request in your dist-git repository created by Packit
- if you are happy with the results, you can change the `dist_git_branches` option to whatever you want, merge your pull request and wait for the next upstream release


### Retriggering
Packagers with write access to the dist-git repository can retrigger the job
via a comment in any dist-git pull request:

    /packit pull-from-upstream

This will take the Packit configuration file from the default branch of the dist-git
  repository (`rawhide`), same as if the job was triggered by a new release. To use the configuration file from the dist-git pull request you are commenting on, you can add an argument:

    /packit pull-from-upstream --with-pr-config

## Keeping dist-git branches non-divergent
Packit currently syncs the release in a way that the branches become divergent (you can follow 
the request to change this behaviour [here](https://github.com/packit/packit/issues/1724)). 

However, if you wish to keep your dist-git branches in sync, you can configure Packit to propose updates exclusively 
to `rawhide` (by specifying `dist_git_branches: fedora-rawhide`) and you can locally merge it with the stable release branches. 
The following example demonstrates how to achieve this for a single branch (`f39` in this case):
```bash
# Clone the dist-git repository if you haven't done so already
fedpkg clone $PACKAGE
# or
git clone ssh://$YOUR_USER@pkgs.fedoraproject.org/rpms/$PACKAGE.git

# Alternatively, pull the rawhide changes only
git pull origin rawhide

# Switch to the desired branch and merge it with the updated rawhide branch
git checkout f39
git merge rawhide
git push origin f39
```
## Koji build job
After having the dist-git content updated, you can easily automate also building in Koji.
You can simply configure Packit to react to the new commits in your dist-git repository and create
Koji builds by having
a Packit configuration (when using `propose_downstream` job, you can configure Packit to sync the file) in your 
default branch (usually `rawhide`) of the dist-git repository that includes a `koji_build` job.
Then, if Packit is informed (via fedora-messaging bus) about a new commit in the configured dist-git branch, it submits a new build in Koji
like maintainers usually do. (The commits without any spec file change are skipped.)

By default, only merged pull requests created by Packit are being acted upon, but 
you can override this behaviour by specifying
`allowed_pr_authors` and/or `allowed_committers` in the [job configuration](/docs/configuration/downstream/koji_build). 

The [koji_build job](/docs/configuration/downstream/koji_build) can be configured like this:

```yaml
jobs:
- job: koji_build
  trigger: commit
  dist_git_branches:
    - fedora-all
```

There is no UI provided by Packit for the job,
but it is visible across Fedora systems (as you can see in the following image).
The koji build behaves as it was created manually, and you can utilise
[Fedora Notifications](https://apps.fedoraproject.org/notifications/about)
to be informed about the builds. Also, you can configure a repository where should we
open issues in case of errors during the job via [`issue_repository`](/docs/configuration#issue_repository) configuration key.

### Retriggering
You can retrigger a build by a comment in a dist-git pull request:

    /packit koji-build

The build will be triggered for the target branch of the pull request using the most recent commit on the target branch
(NOT the HEAD commit of the pull request). The user who posts this comment needs to be a packager.

If Packit created an issue in the configured `issue_repository`, you can place the same comment in that
issue to retrigger the builds (see [`issue_repository`](/docs/configuration#issue_repository) for details).


## Bodhi update job
Lastly, you can again similarly to Koji builds, configure Packit to react to successful Koji builds and create
Bodhi updates by having a Packit configuration in your 
default branch (usually `rawhide`) of the dist-git repository that includes a `bodhi_update` job.
Once Packit is informed (via fedora-messaging bus) about the successful Koji build for the configured branch,
it creates a new update for that branch in Bodhi for you.

The [bodhi_update job](/docs/configuration/downstream/bodhi_update) can be configured like this:

```yaml
jobs:
- job: bodhi_update
  trigger: commit
  dist_git_branches:
    - fedora-branched # rawhide updates are created automatically
```

The packit config is loaded from the commit the build is triggered from.
The `issue_repository` configuration key mentioned in the Koji build job applies here as well.

### Retriggering
You can retrigger an update by a comment in a dist-git pull request:

    /packit create-update

The update will be triggered for the target branch of the pull request. The user who
posts this comment needs to be a packager and have write access to the dist-git repository.

If Packit created an issue in the configured `issue_repository`, you can place the same comment in that
issue to retrigger the updates (see [`issue_repository`](/docs/configuration#issue_repository) for details).

## Full example

Let's take a look how the configuration file can look like when you define all three steps.
It's quite simple, isn't it?

:::tip Downstream configuration template

You can use our [downstream configuration template](/docs/configuration/downstream_configuration_template) 
for creating your Packit configuration in dist-git repository.

:::

:::tip Configuration validation

For validation of the configuration, you can utilise
Packit CLI command [`validate-config`](/docs/cli/validate-config) or our 
[pre-commit hooks](/posts/pre-commit-hooks#validate-config).

:::

```yaml
specfile_path: my_downstream_package_name.spec
files_to_sync:
    - my_downstream_package_name.spec
    - .packit.yaml

upstream_package_name: my_upstream_package_name
upstream_project_url: https://github.com/upstream/package
downstream_package_name: my_downstream_package_name

jobs:

- job: pull_from_upstream
  trigger: release
  dist_git_branches:
   - fedora-rawhide
  actions:
    changelog-entry:
    - bash -c 'echo "- New release ${PACKIT_PROJECT_VERSION}"'

- job: koji_build
  trigger: commit
  dist_git_branches:
    - fedora-rawhide

- job: bodhi_update
  trigger: commit
  dist_git_branches:
    - fedora-branched # rawhide updates are created automatically
```
