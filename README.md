# manjaro-iso-action

Tooling to build and distribute Manjaro on via Github Actions

## Usage example

This action ...

- installs the prerequisites to build Manjaro
- builds a ready to go Manjaro iso
- calculates hashes for the resulting image

It optionally provides:

- GPG-signing
- Distribution to: Github Releases, CDN77, OSDN, SourceForge

The following example is a minimal "matrix strategy" setup, that builds minimal and full images for cinnamon, gnome and builds the images each on stable and testing repositories. Refer [here](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategymatrix) for more information on including / excluding permutations from matrix strategies.

All configuration options and defaults can be found [here](action.yml).

Instead of `manjaro/manjaro-iso-action@main`, please refer to the most current release (e.g. `manjaro/manjaro-iso-action@v1`).

```yaml
name: iso_build
on:
  workflow_dispatch:
  # remove if you don't want to build on a schedule
  schedule:
    - cron:  '30 6 1 * *'
  # remove if you don't want to build when commits are pushed to you main/master branch
  push:
    branches:
      - master
      - main

jobs:
  prepare-release:
    runs-on: ubuntu-20.04
    steps:
      # cancel already running instances of the same action on the currently working on branch
      - uses: styfle/cancel-workflow-action@0.9.0
        with:
          access_token: ${{ github.token }}
      - id: time
        uses: nanzm/get-time-action@v1.1
        with:
          format: 'YYYYMMDDHHmm'
    outputs:
      # generate a common tag to be used in all elements of the matrix strategy
      release_tag: ${{ steps.time.outputs.time }}      
  release:
    runs-on: ubuntu-20.04
    needs: prepare-release    
    strategy:
      matrix:
        ##### EDIT ME #####      
        EDITION: [cinnamon, gnome]
        BRANCH: [stable, testing]
        SCOPE: [minimal,full]
        ###################
    steps:
      # cancel already running instances of the same action on the currently working on branch
      - uses: styfle/cancel-workflow-action@0.9.0
        with:
          access_token: ${{ github.token }}
      - id: image-build
        uses: manjaro/manjaro-iso-action@main
        with:
          edition: ${{ matrix.edition }}
          branch: ${{ matrix.branch }}
          scope: ${{ matrix.scope }}
          version: "21.0"
          kernel: linux510
          code-name: "Ornara"
          # providing a release-tag allows for github releases
          release-tag: ${{ needs.prepare-release.outputs.release_tag }}
      # delete the github release in case of cancellation or failure
      # refer to .github/workflows/cleanup-test-release.yml for rollback strategies concerning the other distribution channels
      - name: rollback github release
        if: ${{ failure() || cancelled() }}
        run: |
          echo ${{ github.token }} | gh auth login --with-token
          gh release delete ${{ needs.prepare-release.outputs.release_tag }} -y --repo ${{ github.repository }}
          git push --delete origin ${{ needs.prepare-release.outputs.release_tag }}
```

### gpg signing

```yaml
- id: image-build
  uses: manjaro/manjaro-iso-action@main
  with:
    ...
    gpg-secret-key-base64: ${{ secrets.GPG_SECRET_KEY_BASE64 }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```

### caching

to get an idea how caching might work, please refer [here](.github/workflows/test.yml)

## Distribution channels

all distribution channels can be configured by setting / leaving out of their configuration variables.

### github release

```yaml
- id: image-build
  uses: manjaro/manjaro-iso-action@main
  with:
    ...
    release-tag: ${{ needs.prepare-release.outputs.release_tag }}
- name: rollback github release
  if: ${{ failure() || cancelled() }}
  run: |
    echo ${{ github.token }} | gh auth login --with-token
    gh release delete ${{ needs.prepare-release.outputs.release_tag }} -y --repo ${{ github.repository }}
    git push --delete origin ${{ needs.prepare-release.outputs.release_tag }}
```

### cdn77 release

```yaml
- id: image-build
  uses: manjaro/manjaro-iso-action@main
  with:
    ...
    cdn77-host: ${{ secrets.CDN_HOST }}
    cdn77-user: ${{ secrets.CDN_USER }}
    cdn77-pwd: ${{ secrets.CDN_PWD }}
- name: rollback cdn77 upload
  if: ${{ failure() || cancelled() }}
  run: |
    sshpass -p "${{ secrets.CDN_PWD }}" rsync --delete -vaP --stats \
        -e ssh $(mktemp) ${{ secrets.CDN_USER }}@${{ secrets.CDN_HOST }}:/www/${{ env.edition }}/${{ env.version }}
```

### SourceForge release

```yaml
- id: image-build
  uses: manjaro/manjaro-iso-action@main
  with:
    ...
    sf-project: manjarolinux
    sf-user: ${{ secrets.SF_USER_NAME }}
    sf-key: ${{ secrets.SF_PRIV_SSHKEY }}
- name: rollback sourceforge upload
  if: ${{ failure() || cancelled() }}
  env:
    SSH_AUTH_SOCK: /tmp/ssh_agent.sock
  run: |
    rsync --delete -vaP --stats \
        -e ssh $(mktemp) ${{ secrets.SF_USER_NAME }}@frs.sourceforge.net:/home/frs/project/manjarolinux/${{ env.edition }}/${{ env.version }}
```

### osdn release

```yaml
- id: image-build
  uses: manjaro/manjaro-iso-action@main
  with:
    ...
    osdn-project: manjaro
    osdn-user: ${{ secrets.OSDN_USER_NAME }}
    osdn-key: ${{ secrets.OSDN_PRIV_SSHKEY }}
- name: rollback osdn upload
  if: ${{ failure() || cancelled() }}
  env:
    SSH_AUTH_SOCK: /tmp/ssh_agent.sock
  run: |
    rsync --delete -vaP --stats \
        -e ssh $(mktemp) ${{ secrets.OSDN_USER_NAME }}@storage.osdn.net:/storage/groups/m/ma/manjaro/${{ env.edition }}/${{ env.version }}
```