Kubler Overlay
==============

[Gentoo](https://www.gentoo.org/get-started/about/) ebuild overlay used by [Kubler](https://github.com/edannenberg/kubler).
This is somewhat container centric, packages with active `minimal` use flag probably don't have very well tested init
scripts. Feel free to open an issue or, even better, a PR. :yum:

## Installation

### Manually

The repo comes with a ready [repos.conf](https://wiki.gentoo.org/wiki//etc/portage/repos.conf) file you just need to add
to your `/etc/portage/repos.conf/` dir:

    curl -sL https://raw.githubusercontent.com/edannenberg/kubler-overlay/master/kubler.conf > /etc/portage/repos.conf/kubler.conf
    emerge --sync

### Using eselect repository

    # if not already installed
    emerge app-eselect/eselect-repository
    # ..then add the overlay 
    eselect repository add kubler git https://github.com/edannenberg/kubler-overlay.git
    emerge --sync

## Ebuild Development with Kubler

Can be used for any Gentoo Portage overlay, example is for this repo:

1. Create a new namespace, let's call it `edev`

```
    $ kubler new namespace edev
```

2. Create a new builder, choose `kubler/bob` as `IMAGE_PARENT`:

```
    $ kubler new builder edev/bob
```

Edit the new builder's `build.sh` and add your overlay:

```
configure_builder() {
    # we overwrite this with a local host mount later, but this takes care of the initial overlay setup in the builder for us
    add_overlay kubler https://github.com/edannenberg/kubler-overlay.git
    # just for convenience
    echo 'cd /var/db/repos/kubler' >> ~/.bashrc
}
```

3. Create a new image, let's call it `bench`, use `kubler/bash` as `IMAGE_PARENT`:

```
    $ kubler new image edev/bench
```

Then edit the new image's `build.conf` and configure the builder and overlay path you want to mount in the builder:

```
    BUILDER="edev/bob"
    BUILDER_MOUNTS=("/home/foo/projects/kubler-overlay:/var/db/repos/kubler")
```

4. Start an interactive build container and get tinkering:

```
    $ kubler build -i edev/bench
    # ebuild dev-lang/foo/foo-0.4.0.ebuild manifest merge 
```

Git Workflow
============

Following is the workflow used for this repository in order to keep our modifications separate, yet regularly be able to integrate upstream changes. This workflow can be referred to as a `rebase` workflow, because we keep rebasing our branch so our changes appear behind the commits of the upstream branch.

## Remotes

Remotes are set up as follows:

- `upstream`: the original developer's repository
- `origin`: our forked repository on Github
- `repo`: our local repository, making it easier for multiple local computers to work on the same project

Setting the remotes:

```
git remote rename origin upstream
git remote add origin git@github.com:inatic/kubler_images.git
git remote add repo ../repo.git
```

## Branches

- `master`: the upstream branch, which is kept as a clean, local mirror of the `upstream/master` branch. No local development is done here.
- `my-custom-changes`: All our custom work is done in this branch.

## Initial Setup

If you are starting fresh or need to reset your repository to the structure described above, follow the next steps.

Create a branch for your custom changes:
```
git checkout -b my-custom-changes
```

Ensure your master branch cleanly tracks the upstream repository:
```
git checkout master
git fetch upstream
git reset --hard upstream/master
```

Push both branches to your own repositories. The `-u` flag sets a tracking relationship between the local and remote branch, so in the future you can just use `git pull` and `git push` without specifying the remote repository (here `origin`).
```
git push origin master --force
git push local master --force
git push origin my-custom-changes -u
git push local my-custom-changes
```

## Upstream Changes

The following steps explain how to integrate `upstream` changes into the custom branch.

Start off by fetching the upstream changes, which downloads the latest commits without applying any changes locally.
```
git fetch upstream
```

Update the local `master` branch, which fast-forwards it to exactly match the state of `upstream/master`.
```
git checkout master
git reset --hard upstream/master
```

Rebase your custom branch on top of the updated `master` branch. This is the most important step, it takes your commits from `my-custom-changes` and reapplies them after all the new commits from the upstream repository.
```
git checkout my-custom-changes
git rebase master
```

The rebase process will pause if there are any conflicts, and git will notify you of the files that are concerned.

- Open the conflicted files and look for the merge markers (`<<<<<<<`, `=======`, `>>>>>>>`).
- Edit the files to resolve the confict.
- Stage the resolved files with `git add <filename>`.
- Continue the rebase with `git rebase --continue`.
- If you get stuck, you can always cancel with `git rebase --abort`

To push your updated branch, you must perform a "force push" as rebasing rewrites your branch's history. Using `--force-with-lease` is safer than `--force`, because the latter unconditionally overwrites the remote branch with the local branch, while the former only overwrites if the remote branch loks exaclty like expected, meaning nobody else wrote their work to it in the mean time. A 'lease' is a temporary and conditional right to use something, and here it is conditioned on no other user having pushed any commits the the remote branch after your latest fetch.

```
git push origin my-custom-changes --force-with-lease
```
