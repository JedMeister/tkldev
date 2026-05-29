Developing a New TurnKey Appliance
===================================

Before attempting to develop a new integration from scratch, it's
recommended to read over the source code of a few `existing apps`_ and
familiarise yourself with the `reference material`_. Bonus points if you
are already familiar with the `maintenance`_ process.

If you're looking for ideas, take a look at `candidates`_ on the `Wiki`_
for inspiration. Note that your integration doesn't have to be a
server-type app — it can be any Linux distribution (e.g., a desktop
system).

Requirements for official inclusion in TurnKey GNU/Linux
---------------------------------------------------------

* Contains free open source software.
* A completely automated build. No interactive steps.
* Follows recommended practices in `Debian Policy`_ and the `Filesystem
  Hierarchy Standard (FHS)`_.

High-level steps
----------------

1. Update the Tracker Wiki
2. Read upstream installation documentation
3. Decide on a name
4. Identify the closest existing integration
5. Clone and start hacking
6. Ensure a completely automated build (handle preseeding / web installers)
7. Implement inithooks
8. Test thoroughly
9. Package (changelog, readme, artwork)
10. Publish

Step 1: Update the Tracker Wiki
''''''''''''''''''''''''''''''''

The `Wiki`_ is used to track candidates as well as a whiteboard. To
facilitate the open development model, it's recommended that you:

* Create a whiteboard page for the integration you're developing.
* Create a candidate entry, and link it to the whiteboard.
* Keep the whiteboard updated with development state, ideas and resources.

Step 2: Read upstream documentation and take notes
'''''''''''''''''''''''''''''''''''''''''''''''''''

Understanding how to install the software, its dependencies and
configuration is obviously important. Things to look out for:

* Is there a recommended configuration for production deployments?
* What are the dependencies? Are they version specific?
* Are components available as packages in Debian? If so, add them to the ``plan``.
* Are components only available upstream? If so, note the download URLs and
  whether releases are GPG-signed. This information goes in ``conf.d/downloads``.
* Does the system require accessible public ports not pre-configured in Core?
  If so, define them in the ``Makefile`` and ``overlay/etc/confconsole/services.txt``.
* Can installation be performed via command line?

Step 3: Decide on a name
'''''''''''''''''''''''''

The name will be used as the repository name and hostname, so it should
be descriptive yet short. Preferably use the name of the main component
(e.g. ``wordpress``). In other cases, be creative (e.g. ``asp-net-apache``).

Step 4: Identify the closest existing integration
''''''''''''''''''''''''''''''''''''''''''''''''''

By reading the installation documentation you should be familiar with
the required base stack. Although Core will work as a starting point,
look for a TurnKey app whose code can be more easily adapted.

Choose a base that matches your stack:

====== ==================== ==============
Base   Example use case     URL
====== ==================== ==============
lamp   PHP web apps         https://github.com/turnkeylinux-apps/lamp
lapp   PHP + PostgreSQL     https://github.com/turnkeylinux-apps/lapp
node   Node.js apps         https://github.com/turnkeylinux-apps/nodejs
rails  Ruby on Rails        https://github.com/turnkeylinux-apps/rails
core   Anything else        https://github.com/turnkeylinux-apps/core
====== ==================== ==============

These common bases are included via the ``Makefile`` (see `Step 5`_ below).

Step 5: Clone and start hacking
''''''''''''''''''''''''''''''''

Clone the integration you want to base off and reinitialise the git
repository to reset the history::

    cd /turnkey/fab/products
    git-clone https://github.com/turnkeylinux-apps/core.git NEW_NAME

    cd NEW_NAME
    rm -rf .git
    git-init

Create the basic appliance directory structure::

    APP_NAME=myapp
    mkdir -p $APP_NAME/{overlay,conf.d,plan}
    cd $APP_NAME

**Makefile**

All TurnKey appliances must include ``$(FAB_PATH)/common/mk/turnkey.mk``.
Base-specific makefiles (e.g. for LAMP) must be included *before*
``turnkey.mk``, as they may modify how it behaves.

For a LAMP-based appliance::

    include $(FAB_PATH)/common/mk/turnkey/lamp.mk
    include $(FAB_PATH)/common/mk/turnkey.mk

Common Makefile flags:

``WEBMIN_FW_TCP_INCOMING``
    TCP ports allowed through the firewall.

``WEBMIN_FW_UDP_INCOMING``
    UDP ports allowed through the firewall.

``NONFREE=y``
    Enable the non-free Debian repo (install packages via plan).

``BACKPORTS=y``
    Enable Debian backports repo (install packages via plan).

``TKL_TESTING=y``
    Enable TurnKey 'testing' apt repo (install packages via plan).

``COMMON_CONF``
    Space-separated list of conf scripts to include from common.

``COMMON_OVERLAYS``
    Space-separated list of overlays to include from common.

``PHP_VERSION=XY``
    Configure PHP X.Y version third party apt repo (deb.sury.org).

**Start hacking**

It's usually easier to comment out code in conf.d scripts, rename the
overlay to something temporary, and build into root.sandbox for manual
integration and testing::

    make CHROOT_ONLY=y
    fab-chroot build/root.sandbox

    # start required services
    # perform installation and configuration manually
    # update conf.d and overlay so it can be done automatically
    # stop services

    exit

Then test your code::

    deck -D build/root.sandbox
    deck -D build/root.patched
    make CHROOT_ONLY=y

Rinse and repeat as needed.

For installing software in your appliance, see the `Installing Software`_ guide.

Step 6: Ensure a completely automated build
''''''''''''''''''''''''''''''''''''''''''''

Builds must be non-interactive and completely automated. The most common
issues are Debian packages that require preseeding, and web-based installers.

**Debian package pre-seeding**

For example, TurnKey `Drupal6`_ uses the Debian drupal6 package, but the
database setup cannot be completed during the build. In this case, preseed
the package and reconfigure it in the conf::

    debconf-set-selections << EOF
    drupal6 drupal6/dbconfig-reinstall boolean true
    EOF
    DEBIAN_FRONTEND=noninteractive dpkg-reconfigure drupal6

**Web-based installers**

When there is no command-line based installation, you sometimes need to
use the web-based installer. Automating this is usually done by scripting
``curl`` to perform the installation.

Firefox has a great extension called ``Live HTTP Headers``, which allows
you to perform the installation with the browser while capturing all the
requests and responses in a log. Using this log, it's easy to script the
installation. For example, in `Joomla25`_::

    URL="http://127.0.0.1/installation/index.php"
    CURL="curl -c /tmp/cookie -b /tmp/cookie"

    $CURL $URL --data "jform%5Blanguage%5D=en-US&task=setup.setlanguage&$SEC=1"
    $CURL ${URL}?view=preinstall
    $CURL ${URL}?view=database
    $CURL $URL --data "jform%5Bdb_type%5D=mysqli&jform%5Bdb_host%5D=localho...
    ...

You can also get creative. For example, in `WordPress`_ a
``turnkey-install.php`` file is created and called with ``curl`` to
perform the installation automatically.

Step 7: Implement inithooks
''''''''''''''''''''''''''''

Initialization hooks are an important part of the user experience and a
security mechanism. They handle things like regenerating secrets, setting
the admin email, password and domain on first boot.

See the `Inithooks`_ guide for full details on implementing inithooks.

When inithooks are required, use the following naming conventions::

    overlay/usr/lib/inithooks/bin/drupal7.py
    overlay/usr/lib/inithooks/firstboot.d/20regen-drupal7-secrets
    overlay/usr/lib/inithooks/firstboot.d/40drupal7

Inithooks are executed in alpha-numeric order. Secret regeneration should
be prefixed with ``20`` and application settings (email, passwords, domain)
with ``40``.

**Bonus: Welcome post / tklweb-cp**

To improve the user experience, a welcome page/post can be injected into
the database (e.g. `MediaWiki conf`_) or a TurnKey Web Control panel
created (e.g. `DomainController tklweb-cp`_).

**Bonus: TKLBAM profile overrides**

Each TurnKey app has a `TurnKey Backup and Migration`_ profile describing
what should and shouldn't be backed up. For example, to exclude bloat from
`Drupal7`_::

    $ cat overlay/etc/tklbam/overrides
    -mysql:drupal7/sessions
    -mysql:drupal7/cache
    -mysql:drupal7/cache_filter
    -mysql:drupal7/cache_menu
    -mysql:drupal7/cache_page
    -mysql:drupal7/cache_views
    -mysql:drupal7/search_dataset
    -mysql:drupal7/search_index
    -mysql:drupal7/search_total

Step 8: Testing
'''''''''''''''

To avoid nasty surprises, integrations should be well tested. After all
your development iterations, perform a clean build::

    deck -D build/root.sandbox
    make clean
    make

Then test ``build/product.iso`` in a VM (both live and installed).

Step 9: Packaging — changelog, readme and artwork
''''''''''''''''''''''''''''''''''''''''''''''''''

TurnKey apps follow a packaging convention:

* **changelog**: Use the changelog from the integration you based off
  (e.g. Core) as a starting point. The Debian devscripts package includes
  a helper::

    dch -i

  See the `Changelog`_ guide for full formatting details.

* **README.rst**: Should include an opening overview paragraph and any
  information users should know. Formatted in `reStructuredText`_.

* **.art**: Should include a logo and screenshots. Templates and
  guidelines are available in `TurnKey Artwork`_.

Step 10: Publishing
''''''''''''''''''''

* Register a new repository on GitHub and push your branch.
* Update the whiteboard on the `Wiki`_ you created earlier.
* Create a new issue on the `Issue Tracker`_ with a ``#new-appliance`` tag.

.. _existing apps: https://github.com/turnkeylinux-apps/
.. _reference material: ../reference/
.. _maintenance: ./maintaining-appliance.rst
.. _candidates: https://github.com/turnkeylinux/tracker/wiki/Candidates
.. _Wiki: https://github.com/turnkeylinux/tracker/wiki
.. _Debian Policy: http://www.debian.org/doc/debian-policy/
.. _Filesystem Hierarchy Standard (FHS): http://www.pathname.com/fhs/
.. _Installing Software: ./installing-software.rst
.. _Inithooks: ./inithooks.rst
.. _Changelog: ../contributing/changelog.rst
.. _Drupal6: https://github.com/turnkeylinux-apps/drupal6/
.. _Joomla25: https://github.com/turnkeylinux-apps/joomla25/
.. _Wordpress: https://github.com/turnkeylinux-apps/wordpress/
.. _Drupal7: https://github.com/turnkeylinux-apps/drupal7/
.. _GitLab inithook: https://github.com/turnkeylinux-apps/gitlab/blob/master/overlay/usr/lib/inithooks/bin/gitlab.py
.. _Vanilla inithook: https://github.com/turnkeylinux-apps/vanilla/blob/master/overlay/usr/lib/inithooks/bin/vanilla_pass.php
.. _MediaWiki conf: https://github.com/turnkeylinux-apps/mediawiki/blob/master/conf.d/main
.. _DomainController tklweb-cp: https://github.com/turnkeylinux-apps/domaincontroller/blob/master/overlay/var/www/index.shtml
.. _TurnKey Backup and Migration: https://www.turnkeylinux.org/tklbam/
.. _reStructuredText: http://docutils.sourceforge.net/docs/user/rst/quickref.html
.. _TurnKey Artwork: https://github.com/turnkeylinux/artwork/
.. _Issue Tracker: https://github.com/turnkeylinux/tracker/issues?labels=new-appliance
.. _Step 5: #step-5-clone-and-start-hacking
