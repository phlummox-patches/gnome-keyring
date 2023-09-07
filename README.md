# gnome keyring patches for Ubuntu 22.04

The Gnome keyring (at the time of writing, version 40.0) running on Ubuntu
version 22.04  has a fairly egregious UI flaw – when unlocking keyrings, it
*pre-selects* the less secure option of "Automatically unlock this keyring
whenever I'm logged in", and if you accidentally *do* leave that option
selected, it's quite difficult to undo.

If you encounter the bug and try to work out what Gnome component is causing the
problem, you might do a process listing and discover that the program displaying
the "unlock" prompt seems to be `/usr/libexec/gcr-prompter`, part of the Gnome
[gcr][gcr] package,[^gcr] and you might guess that the responsible code is
somewhere in that package (maybe in the `gcr` or `ui` directories).
However, *that's not where the bug lies*. The gcr-prompter seems to be a fairly
"dumb" UI widget, and the flawed logic is in the Gnome keyring component, which
creates the gcr widget (and presumably the `gcr-prompter` process?) in some
mysterious way (probably via dbus, but I can't be bothered to wade through the
source to find out exactly).

[gcr]: https://gitlab.gnome.org/GNOME/gcr

[^gcr]: What the abbreviation "gcr" stands for is unclear, but
  probably "**G**nome **cr**yptography services".

Below are some historical details about the bug, then instructions on how to
rebuild the Ubuntu .deb packages for the Gnome keyring (plus related packages)
incorporating a patch by [Atul Anand][atul] which fixes the issue.
And if you happen to be running Ubuntu 22.04 (or
a distro using the 22.04 repos, like Linux Mint 21.2) on an x86-64 machine,
you can use the .deb files on the "Releases" page of this repo.

## Background (feel free to skip)

This bug was reported on Gnome's Bugzilla instance in [Bug 725641][bugz-725641]
in 2014, and a fix created by [Atul Anand][atul] in 2016, but the fix has never
been incorporated by the Gnome keyring maintainer, [Niels De Graef][niels]
([`@nielsdg`][nielsdg]).

[atul]: https://github.com/atulhjp
[bugz-725641]: https://bugzilla.gnome.org/show_bug.cgi?id=725641
[niels]: https://nielsdg.pages.gitlab.gnome.org/development-blog/about/
[nielsdg]: https://gitlab.gnome.org/nielsdg

Gnome's bugzilla closed down, and the bug got transferred by Niels to the Gnome
GitLab instance as [issue #46][gl-issue-46] in the "gcr" package (not actually where
the bug lies, so far as I can tell) in March 2014.

[gl-issue-46]: https://gitlab.gnome.org/GNOME/gcr/-/issues/46

Gnome keyring [issue #7][gl-issue-7], filed in 2018, points out that the issue
still hasn't been fixed, and mentions some related bugs:

- 2014 ["Automatically unlock this keyring whenever I'm logged in" should be unchecked by default](https://bugzilla.gnome.org/show_bug.cgi?id=740734)
- 2009 ["Automatically unlock when I log in" considered harmful](https://bugzilla.gnome.org/show_bug.cgi?id=576676)

[gl-issue-7]: https://gitlab.gnome.org/GNOME/gnome-keyring/-/issues/7

In [Merge request #38][gl-mreq-38] (from May 2021), a fix is again supplied,
this time by [Ryan Hendrickson][ryan], but it still hasn't been merged, and in a
[March 2023 post][disc-post] to Gnome's Discourse instance, Ryan wonders why,
but doesn't get any terribly good answers. (Other than "contact the maintainer" –
but if filing issues and proposing merge requests isn't a good way of
contacting a maintainer, then why on earth have them at all?)

[ryan]: https://github.com/rhendric
[gl-mreq-38]: https://gitlab.gnome.org/GNOME/gnome-keyring/-/merge_requests/38
[disc-post]: https://discourse.gnome.org/t/aging-gnome-keyring-mr-needs-review/14335

Anyway. Below are steps to re-build the .deb packages with Atul's fix incorporated.

If you install these, you'll have to "hold" the Gnome keyring packages at their current
version, and forego any new versions arriving from upstream -- but if the last
nine years are any guide, you won't be missing anything much.

## Steps to re-build .deb packages

These steps verified for Ubuntu 22.04 (Jammy) – probably they can be adapted
for other Ubuntu versions.

1.  Make a fresh directory somewhere, in which the Ubuntu source and .deb
    files will end up. It'll be cleaner to do this inside a Docker container,
    so that you don't clutter your system with the build dependencies of
    gnome-keyring.

    Anyhow, `cd` into your directory, and

    ```
    $ sudo apt-get update
    $ apt-get source gnome-keyring
    $ sudo apt-get build-dep gnome-keyring
    $ sudo apt-get install fakeroot build-essential devscripts quilt
    ```

2.  Running `apt-get source` will create a directory `gnome-keyring_40.0`, as
    well files with extensions `.dsc`, `.orig.tar.xz`, and `.debian.tar.xz`.
    The `.orig.tar.xz` file gets extracted into the directory, and the patches
    in the `.debian.tar.xz` file get automatically applied.

3.  We first want to un-apply the debian patches, so we can insert our own
    patch into them, and then apply the whole lot at once. So cd into
    `gnome-keyring_40.0` run `quilt pop -a`, and you should see a bunch of
    messages about removing patches and restoring files, ending with "No patches applied".

4.  Apply the patches from this repo -- `git apply /path/to/repo/ui_fixes.patch`,
    which adds a new patch in `debian/patches`, and then apply all the
    patches with `quilt push -a`.

    (Is all this necessary? idk. Probably. Could it be made
    simpler? Also probably.)

5.  Run `DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -us -uc`, and your .deb
    (and .ddeb) packages will be built.

