Appliance Release Checklist
============================

Use this checklist when preparing an appliance for release. For detailed
guidance on any item, refer to the linked documentation.

Makefile
--------

See `New Appliance`_ and `Buildcode Directories`_.

- [ ] Ensure any ``/common/mk/turnkey/*`` makefiles are loaded BEFORE ``/common/mk/turnkey.mk``
- [ ] Shared code is included from common rather than re-implemented

Upstream Software
-----------------

Third Party Apt Repos
~~~~~~~~~~~~~~~~~~~~~
- [ ] GPG keys are set up correctly:

  - [ ] GPG keys downloaded to ``/usr/share/keyrings``
  - [ ] ASCII armored GPG keys are either dearmored or have a ``.asc`` suffix
  - [ ] ``/etc/apt/sources.list`` line includes ``[signed-by=/usr/share/keyrings/<key>.gpg]``

- [ ] Apt pinning:

  - [ ] Packages **meant** for install are pinned to 500
  - [ ] Packages **not** meant for install are pinned to 10

Third Party Package Management
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

    Document your third party package manager usage (pip, npm, composer, etc.)
    and ensure the package manager itself is included in the plan if needed.

Install from Upstream
~~~~~~~~~~~~~~~~~~~~~

- [ ] ``conf.d/downloads`` setup correctly:

  - [ ] Includes proxy-compliant ``dl`` function
  - [ ] ``URL`` variable points to the download URL
  - [ ] ``VERSION`` variable interpolates into ``URL`` if applicable
  - [ ] Both source and binary packages downloaded to ``/usr/local/src``

- [ ] ``conf.d/main`` setup correctly:

  - [ ] Compilation occurs in ``conf.d/main``

Inithooks vs conf.d/main
~~~~~~~~~~~~~~~~~~~~~~~~~

- [ ] Any one-time configuration that **doesn't** include secrets is in ``conf.d/main``
- [ ] Any idempotent configuration not required for basic functionality is in inithooks
- [ ] Any one-time configuration that **does** include secrets is handled case-by-case,
      with secret regeneration in firstboot scripts

Inithooks
~~~~~~~~~

See the `Inithooks`_ guide for full details.

- [ ] Ensure the following preseed values that are relevant to this appliance are
      passed from firstboot scripts to the inithooks:

  - ``ROOT_PASS``
  - ``DB_PASS``
  - ``APP_PASS``
  - ``APP_EMAIL``
  - ``APP_DOMAIN``

- [ ] All non-standard preseed values have sane defaults (required for Hub compatibility)
- [ ] Passwords are all set
- [ ] Admin email is set and added to inithooks cache

Other
-----

- [ ] TKLBAM profile is up to date and working
- [ ] Changelog has been updated — see `Changelog`_
- [ ] README has been updated (if applicable)
- [ ] Artwork has been updated (if applicable)

.. _New Appliance: ./new-appliance.rst
.. _Buildcode Directories: ../reference/buildcode-directories.md
.. _Inithooks: ./inithooks.rst
.. _Changelog: ../contributing/changelog.rst
