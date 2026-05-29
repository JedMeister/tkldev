Reusing Appliance Build Code
============================

DRY Principle — Don't Repeat Yourself
--------------------------------------

The essence of the DRY principle is to reuse code as much as possible —
to avoid rewriting the same or similar code over and over again.

There are a number of ways to minimize the amount of code you need to
write for new appliances and major appliance updates. It's worth noting
that applying DRY principles involves trade-offs. Sometimes it is obvious
when code can be reused; other times it is a subjective decision.
Sometimes writing code that can be reused in general cases can be more
work than just having specific code in numerous places.

A good rule of thumb: if you find yourself spending time trying to
generalise code to account for numerous different scenarios, it's
probably time to back out and just write specific code for a known state.

The ``common`` Repository
--------------------------

Code that is useful for multiple — or all — appliances (e.g. LAMP-based
apps) should go in `common`_. By default on TKLDev, common is at
``/turnkey/fab/common``. It uses a very similar structure to an appliance
build code directory, but instead of ``overlay``, ``conf.d`` and ``plan``,
the common directories are ``overlays``, ``conf`` and ``plans``
respectively. Common also contains a ``mk`` directory for grouping bundles
of shared overlays and/or conf scripts.

For further context on the file tree structure including common, see the
`Buildcode Directories`_ reference page.

Shared plans — adding common packages
''''''''''''''''''''''''''''''''''''''

To add custom packages to multiple builds, the best place to put them is
in a common plan. In the ``plans/turnkey`` directory, you'll find the
``base`` plan — that is applied to all servers. To add your own:

1. Create ``common/plans/turnkey/custom`` with your package names.
2. In each appliance that wants this plan, add it near the top of
   ``plan/main``::

    #include <turnkey/custom>

Shared overlays and conf scripts
'''''''''''''''''''''''''''''''''

To add customisations that apply to **all** builds, add conf scripts
and/or overlays to ``common/conf/turnkey.d`` and/or
``common/overlays/turnkey.d``. Files in those locations are included in
all future builds.

To include customisations in **some** builds (but not all), place them
in ``common/conf/`` and/or ``common/overlays/`` and then reference them
from each appliance's ``Makefile``::

   COMMON_OVERLAYS = custom-overlay
   COMMON_CONF = custom-conf

These map to ``common/overlays/custom-overlay/`` and
``common/conf/custom-conf`` respectively. Additional items can be added
as a space-separated list.

Grouping via a common makefile
'''''''''''''''''''''''''''''''

To manage a group of conf scripts and overlays together, create a common
makefile in ``common/mk/turnkey/``. For example
``common/mk/turnkey/custom.mk``::

   COMMON_OVERLAYS += custom-overlay
   COMMON_CONF += custom-conf1 custom-conf2

   include $(FAB_PATH)/common/mk/turnkey/custom2.mk
   include $(FAB_PATH)/common/mk/turnkey/custom3.mk

Then to use the common makefile, add this to each appliance Makefile::

   include $(FAB_PATH)/common/mk/turnkey/custom.mk

Applying shared common components in an appliance
''''''''''''''''''''''''''''''''''''''''''''''''''

Common overlays/conf scripts are included in an appliance via its
``Makefile``. For example, to apply all common LAMP conf and overlays::

    include $(FAB_PATH)/common/mk/turnkey/lamp.mk

Shared plans are included via the appliance's ``plan/main`` file.

DRY in Appliance Conf Scripts
------------------------------

As appliance conf scripts are bash scripts, the usual bash methods of
applying DRY apply here too: variables, variable manipulation, loops,
flow control and functions can all minimise how much you need to write
and often make conf scripts easier to read and maintain.

Sharing variables between conf scripts
''''''''''''''''''''''''''''''''''''''

Scripts in ``conf.d/`` are executed in isolation (alpha-numeric order),
so a single script defining env vars and functions won't persist to the
next script. You can work around this by writing values to a temp file
and sourcing it in subsequent scripts.

For example, write a variable to ``/tmp/env`` in ``downloads``::

    echo "MY_VAR=$MY_VAR" >> /tmp/env

Then load it from ``main``::

    source /tmp/env

Use ``>>`` to append; ``>`` to overwrite. To quote a value::

    echo "MY_VAR=\"$MY_VAR\"" >> /tmp/env

Note that script execution order is alpha-numeric: ``downloads`` runs
before ``main``, and ``00-setup`` before ``zzz-final``.

Sharing functions with a heredoc env file
'''''''''''''''''''''''''''''''''''''''''

For more complex shared code including functions, writing a heredoc to
a temp file is often cleaner. For example, ``conf.d/00-env``::

    cat > /tmp/env << EOF
    MY_VAR_1="$MY_VAR"
    MY_VAR_2="345"

    echo_message() {
        local stuff_to_echo="\$@"
        echo "message: \$stuff_to_echo"
    }

    setup_dirs() {
        local default_dir="/some/dir"
        local dir1=\$1
        local dir2=\$2
        mkdir -p "\$dir1 \$dir2 \$default_dir"
        chmod 755 "\$dir1 \$dir2 \$default_dir"
    }
    EOF

Then use in other conf scripts::

    #!/bin/bash -e

    source /tmp/env

    echo_message "making dirs"
    setup_dirs "/some/dir /some/other/dir"

Scripts available to end users
'''''''''''''''''''''''''''''''

If you develop code that might be useful to end users, add the script as
a file in the overlay (rather than writing it via a heredoc). Scripts
intended for end users should go in ``/usr/local/bin`` — they will be
in the user's PATH automatically and are clearly not from apt packages.

By convention, prefix end-user scripts with ``turnkey-`` or ``tkl-``
(TKLDev scripts use a ``tkldev-`` prefix). Like conf scripts, overlay
scripts must include ``#!/bin/bash`` as the first line and must be
executable.

Where scripts are useful for multiple appliances, consider adding them
to a common overlay instead.

.. _common: https://github.com/turnkeylinux/common
.. _Buildcode Directories: ../reference/buildcode-directories.md
.. _base: https://github.com/turnkeylinux/common/blob/master/plans/turnkey/base
