# Lullabot Support & Maintenance Coding Assessment

This repository contains a Drupal 11 site configured for local development and project automation with DDEV.

Drainpipe provides the project’s shared build and automation tooling, including Drupal tasks, static testing, generated GitHub and Tugboat integration, and creation of the deployment artifact sent to Pantheon. 

GitHub Actions runs the Drainpipe static test suite and validates Renovate configuration on pushes to main and pull requests. 

Tugboat provides hosted preview environments that build the project from GitHub and install Drupal from the committed configuration, allowing changes to be reviewed before deployment to Pantheon.

Pantheon’s Terminus CLI is used to manage the hosted environment from the command line, including authentication, backups, Drupal commands, and deployment operations. Running Terminus with `ddev exec` is recommended because Drainpipe installs and configures it inside DDEV, providing a consistent, project-specific toolchain without requiring a separate host installation.

## Contents

- [Environments](#environments)
- [Repository structure](#repository-structure)
- [Ignore-file strategy](#ignore-file-strategy)
- [Local setup](#local-setup)
  - [Requirements](#requirements)
  - [Setup](#setup)
- [General workflow](#general-workflow)
- [Drupal configuration management](#drupal-configuration-management)
- [Testing](#testing)
- [Tugboat previews](#tugboat-previews)
- [Pantheon deployment](#pantheon-deployment)
  - [Deployment approach](#deployment-approach)
  - [Authentication](#authentication)
  - [Build and deploy](#build-and-deploy)
  - [First installation](#first-installation)
- [Deploying a configuration change](#deploying-a-configuration-change)

## Environments

| Environment | URL |
|---|---|
| Tugboat preview | https://main-atjxecdnsx2mmxz7uknzboel5c3hj7ar.tugboatqa.com/ |
| Pantheon Dev | https://dev-lullabot-cr-project.pantheonsite.io |
| Source repository | https://github.com/HammerHankinson/lullabot-cr-project |

## Repository structure

The project uses Drupal's recommended Composer layout:

```text
config/sync/             Exported Drupal configuration
web/                     Drupal document root
.github/                 GitHub Actions and Drainpipe actions
.tugboat/config.yml      Generated Tugboat configuration
.ddev/config.yaml        Local DDEV configuration
Taskfile.yml             Project and Drainpipe tasks
composer.json            PHP dependencies and Drainpipe integration
pantheon.yml             Site specific Pantheon configuration
pantheon.upstream.yml    Pantheon owned upstream configuration
```

Composer generated dependencies, including `vendor`, Drupal core, and contributed extensions, are not committed to the source repository. They are installed locally and in CI from `composer.lock`.

Pantheon is an exception because this project deploys a complete build artifact to Pantheon's Git repository. Drainpipe creates that artifact using `.drainpipeignore`.

## Ignore file strategy

This project uses two files with related but different purposes:

- `.gitignore` controls which local and Composer generated files are excluded from the GitHub source repository.
- `.drainpipeignore` controls which files are excluded from the complete Pantheon deployment artifact.

Composer generated runtime dependencies are ignored by Git but included in the Pantheon artifact. Local secrets, DDEV configuration, test tooling, and CI configuration are excluded from the Pantheon artifact.

## Local setup

### Requirements

Git and DDEV should be installed before starting:

Composer, Drush, Task, and Pantheon's Terminus CLI do not need to be installed separately on the host. They are installed or made available within DDEV when the project dependencies are installed.

Access to the Pantheon project and an authenticated Terminus session are also needed to download a database backup. Drainpipe provides Terminus inside DDEV, so its commands should be run with `ddev exec`.

### Setup

- Clone repo from [Github](git@github.com:HammerHankinson/lullabot-cr-project.git) (don't use the git repo from Pantheon)
- From the project root, run `ddev start`
- Run `ddev composer install` to install the project’s Composer dependencies, including Drupal, Drush, and Drainpipe.
- Authenticate Terminus by following the [authentication instructions](#authentication) - Create .env, set TERMINUS_MACHINE_TOKEN, and run:
```bash
ddev task pantheon:auth
```
- Download a recent database dump from Pantheon. To ensure it contains the latest database state, create a fresh backup first:
```bash
ddev exec terminus backup:create lullabot-cr-project.dev --element=database

ddev exec terminus backup:get lullabot-cr-project.dev \
  --element=database \
  --to=database.sql.gz
```
- Run `ddev import-db --file=database.sql.gz`
- Synchronize [Pantheon's uploaded files](https://docs.pantheon.io/guides/local-development/configuration#transfer-your-files) so local images and documents match the database. [Rsync or SFTP](https://docs.pantheon.io/guides/sftp/rsync-and-sftp) maybe be required for large file transfers.
- The trailing slash on `files/` copies the directory's contents rather than the directory itself. This command does not delete local-only files.
- Run `ddev drush deploy --yes` to make sure your local environment is up to date

## General workflow

- Make site changes locally and use [Drupal configuration management](#drupal-configuration-management) to export, review, and commit configuration changes.
- Run the project's [tests](#testing) before pushing changes to GitHub; GitHub Actions runs the same static test suite for pushes and pull requests.
- Push changes to GitHub to rebuild the [Tugboat preview](#tugboat-previews) for review.
- Build and push a separate artifact for [Pantheon deployment](#pantheon-deployment), then run Drupal deployment operations to apply database and configuration updates.
- Follow the complete [configuration-change deployment workflow](#deploying-a-configuration-change) when promoting exported configuration to both Tugboat and Pantheon.

## Drupal configuration management

Drupal configuration is stored in `config/sync`.

After making a configuration change through Drupal's administrative interface, export it:

```bash
ddev drush config:export --yes
```

Review the exported changes before committing:

```bash
git status --short
git diff -- config/sync
```

To import committed configuration locally:

```bash
ddev drush config:import --yes
ddev drush cache:rebuild
```

Configuration should be reviewed like application code. Unrelated or environment specific configuration should not be committed accidentally.

## Testing

Run the complete Drainpipe static test suite with:

```bash
ddev task test:static
```

The suite includes:

- Composer validation
- YAML and configuration linting
- PHP_CodeSniffer
- PHPStan
- PHPUnit static initialization
- A clean working tree check

The clean working tree check expects all intended changes to be committed or ignored. For that reason, run the complete suite after creating the commit being tested.

GitHub Actions runs the same static test task for repository changes.

## Tugboat previews

Drainpipe generates `.tugboat/config.yml` from the DDEV configuration during Composer installation.

The project defines a dedicated Tugboat synchronization task:

```yaml
sync:tugboat:
  desc: "Install Drupal from exported configuration in Tugboat"
  cmds:
    - task: drupal:install
```

This ensures that a new Tugboat preview installs Drupal from `config/sync` before Drainpipe runs database updates.

Changes pushed to the GitHub `main` branch trigger the configured Tugboat preview rebuild. Pull request preview behavior is controlled through the Tugboat repository settings.

Because `.tugboat/config.yml` is generated by Drainpipe, project specific changes should be made through the supported Drainpipe or Tugboat override mechanisms rather than by editing the generated output directly.

## Pantheon deployment

### Deployment approach

GitHub is the canonical source repository. Pantheon has a separate Git repository that receives a complete production artifact.

This distinction is important:

- GitHub contains the project source and `composer.lock`.
- Generated dependencies such as `vendor` and Drupal core are ignored.
- Pantheon Integrated Composer is disabled with `build_step: false`.
- Drainpipe runs the production Composer build locally or in CI.
- Drainpipe creates a filtered snapshot and pushes that snapshot to Pantheon.

Pantheon's Dev environment deploys from its `master` branch, while this GitHub repository uses `main`.

### Authentication

Create a Pantheon machine token and add it to the ignored local `.env` file:

```dotenv
TERMINUS_MACHINE_TOKEN=replace-with-machine-token
PANTHEON_SITE_ID=replace-with-pantheon-site-uuid
```

Never commit `.env` or the machine token.

Restart DDEV and authenticate:

```bash
ddev restart
ddev task pantheon:auth
```

Confirm that Terminus can access the site:

```bash
ddev exec terminus site:list
```

Pantheon Git deployment also requires an SSH public key registered with the Pantheon account. Load local SSH identities into DDEV with:

```bash
ddev auth ssh
```

### Build and deploy

Run the production build:

```bash
ddev task build
```

This installs the locked Composer dependencies without development packages.

Create a uniquely named deployment snapshot:

```bash
ddev task snapshot:directory \
  directory=/tmp/lullabot-cr-pantheon-release
```
The snapshot must be unique as Drainpipe explicitly fails when the target directory already exists.

Use a new path for every release, such as:
```bash
ddev task snapshot:directory \
  directory=/tmp/lullabot-cr-pantheon-release-20260727-1
```

The snapshot excludes local, CI, secret, and test files according to `.drainpipeignore`. Before deployment, confirm that it does not contain secrets or local tooling:

```bash
ddev exec sh -lc '
  cd /tmp/lullabot-cr-pantheon-release &&
  test ! -e .env &&
  test ! -e .ddev &&
  test ! -e .github &&
  test ! -e .task &&
  test -f vendor/autoload.php &&
  test -f web/core/core.api.php &&
  test -f pantheon.yml &&
  test -f pantheon.upstream.yml &&
  echo "Release snapshot validation passed."
'
```

Deploy the artifact using Drainpipe:

```bash
ddev exec 'task deploy:git directory=/tmp/lullabot-cr-pantheon-release branch=master remote="[PANTHEON_GIT_REMOTE]" message="Deploy Drupal release to Pantheon"'
```

The remote Git URL is available from the Pantheon dashboard under the Dev environment's connection information.

After Pantheon finishes processing the Git push, apply Drupal's deployment operations:

```bash
ddev exec terminus drush lullabot-cr-project.dev -- deploy --yes
```

Restore local development dependencies after producing the artifact:

```bash
ddev composer install
```

Finally, verify the local repository:

```bash
ddev task test:static
git status --short
```

### First installation

A newly provisioned Pantheon database may not contain an installed Drupal site. For the initial deployment only, install Drupal from the exported configuration:

```bash
ddev exec terminus drush lullabot-cr-project.dev -- \
  site:install --existing-config --yes
```

This command replaces an existing Drupal database. It should only be used for a new or intentionally disposable environment.

## Deploying a configuration change

A typical configuration only change follows this sequence:

```bash
# Export and review Drupal configuration.
ddev drush config:export --yes
git diff -- config/sync

# Commit and test.
git add config/sync
git commit -m "Describe the Drupal configuration change"
ddev task test:static

# Push the source repository.
git push origin main
```

The GitHub push runs CI and triggers the Tugboat rebuild.

Pantheon then receives a separately generated artifact:

```bash
ddev task build
ddev task snapshot:directory directory=/tmp/lullabot-cr-pantheon-release
ddev exec 'task deploy:git directory=/tmp/lullabot-cr-pantheon-release branch=master remote="[PANTHEON_GIT_REMOTE]" message="Deploy Drupal configuration update"'
ddev exec terminus drush lullabot-cr-project.dev -- deploy --yes
ddev composer install
```
