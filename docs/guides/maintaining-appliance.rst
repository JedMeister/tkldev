Improving and Maintaining a TurnKey Appliance
=============================================

This guide covers how to fix a bug, add functionality, upgrade upstream
software, or make any other change — and get your changes included in
the official TurnKey library.

For example, say a new version of ``projectpier`` has been released, and
we want to update `TurnKey ProjectPier`_.

Fork and clone the source
--------------------------

As described in the TurnKey `Git Flow`_, GitHub is used to facilitate
collaboration, so the first thing to do is fork the source code:

* Log into GitHub, and browse to https://github.com/turnkeylinux-apps/projectpier/
* Click the ``fork`` button.

That's it. You've successfully forked the ``projectpier`` repository,
but so far it only exists on GitHub.

To be able to work on the project you'll need to clone it::

    cd /turnkey/fab/products
    git-clone git@github.com:USERNAME/projectpier.git

When a repository is cloned, it has a default ``remote`` called ``origin``
that points to your fork on GitHub, not the original repository it was
forked from. To keep track of the original repository, add another remote
called ``upstream``::

    cd projectpier
    git remote add upstream https://github.com/turnkeylinux-apps/projectpier.git

    # Fetch any new changes to the original repository
    git-fetch upstream

    # Merge any changes fetched into your working branch
    git merge upstream/master

Make your changes
-----------------

Create a branch
'''''''''''''''

The first thing to do is create a branch for the update. Note that you
have only one ``pull request`` per branch::

    git-checkout -b upgrade-to-vXX.YY

Perform the change
''''''''''''''''''

Read the release notes of the new version to see if there are any other
changes that might need to be made. Then make your changes — for example,
updating ``conf.d/downloads`` with a new download URL and checksum.

Test
''''

Build using the ``CHROOT_ONLY=y`` flag to take a quick shortcut. This
excludes boot-related packages and stops the build at the ``root.sandbox``
stage::

    make CHROOT_ONLY=y

If there were any issues during the build, fix them and run
``make CHROOT_ONLY=y`` again. The build will pick up from where it left
off and re-make the target that caused the issue.

Once the ``CHROOT_ONLY`` build is successful, test it::

    fab-chroot build/root.sandbox
    /etc/init.d/mysql start
    /etc/init.d/apache2 start
    /usr/lib/inithooks/firstboot.d/20regen-projectpier-secrets

    # on host system, browse to http://ip-of-tkldev-vm
    # verify there are no issues

    /etc/init.d/apache2 stop
    /etc/init.d/mysql stop
    exit

Note: If testing via the Hub you will need to adjust the Firewall rules
to open any relevant ports (e.g. 80 for http, etc).

Testing complete? Perform a cleanup::

    deck -D build/root.sandbox
    make clean

.. note::

    Not all changes can be tested reliably (or at all) inside chroot
    using the ``CHROOT_ONLY`` shortcut. For large changes or changes that
    may affect the boot process (e.g. inithooks) it is recommended to
    perform a full build and test the ISO in a VM instead.

Commit
''''''

Commit your changes::

    $ git-status
    # On branch upgrade-to-vXX.YY
    # Changed but not updated:
    #   modified:   conf.d/downloads

    $ git-add conf.d/downloads
    $ git-commit -m "Upgraded projectpier to version XX.YY"

Push changes to GitHub and submit a Pull Request
-------------------------------------------------

Push your changes to your GitHub repository::

    git-push origin upgrade-to-vXX.YY

Then send a ``pull request`` so the maintainer or one of the core
developers can review, sign off, and merge it into the official repository:

* Browse to https://github.com/USERNAME/projectpier/tree/upgrade-to-vXX.YY
* Click ``Pull Request``, describe the change and click ``Send pull request``.

If the maintainer requests changes, any new commits you push to that
branch will automatically be included in the open pull request.

.. _TurnKey ProjectPier: https://github.com/turnkeylinux-apps/projectpier/
.. _Git Flow: https://github.com/turnkeylinux/tracker/blob/master/GITFLOW.rst
