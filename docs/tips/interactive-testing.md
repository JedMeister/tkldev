Development and Testing in a Build Chroot
=========================================

When working on appliance development and/or updates, you can tighten the
feedback loop by only rebuilding the build step/s you need to. That will save
you from having to build from scratch to ISO every time.

For an overview of the build steps and what each layer contains, see the
[build steps reference](../reference/build-steps.md).

Interactively exploring a product's filesystem
----------------------------------------------

You can chroot into the product's filesystem to explore, hack or test
stuff interactively without having to make changes to source code:

```
root@tkldev products/core# make root.sandbox
root@tkldev products/core# fab-investigate build/root.sandbox
(core chroot)root@tkldev /#
```

Then when you're done:

```
(core chroot)root@tkldev /# exit
root@tkldev products/core#
```

For a more detailed example see [Hello world](../getting-started/helloworld.rst).

Remember: if you do a `make clean` (or rebuild root.patched as noted below)
then any changes within root.sandbox will be lost. Be sure to:

- copy out overlay files you want to keep; and/or
- copy/paste commands you want to rerun at build time into a conf.d script.

Adding new packages
-------------------

If you want to add or test a new package without rebuilding from scratch,
you can install it directly into the sandbox chroot:

```
root@tkldev products/core# make root.sandbox
root@tkldev products/core# fab-chroot build/root.sandbox
(core chroot)root@tkldev /# apt-get install <package-name>
(core chroot)root@tkldev /# exit
```

This lets you verify a package installs and behaves correctly before
committing to a full rebuild. Once you're satisfied, add the package name
to ``plan/main`` and rebuild from root.build (see below) to incorporate
it properly.

Note that changes to ``plan/main`` require a full ``make clean`` and rebuild
from scratch, because package installation happens at the root.build step
which root.patched depends on.

Quick re-patch: how to reapply root.patched configurations
----------------------------------------------------------

You'll save a lot of time if you realise you don't have to start the build
from scratch with a `make clean` every time you make a change to a conf
script or an overlay file.

Removing a build step can be done with `fab-rewind`. To clean root.sandbox
and play some more without completely rebuilding from scratch:

```
root@tkldev products/core# fab-rewind root.patched
root@tkldev products/core# make root.sandbox
root@tkldev products/core# fab-investigate build/root.sandbox
```

To test your updated overlays/conf-scripts, rebuild root.patched:

```
root@tkldev products/core# fab-rewind root.build
root@tkldev products/core# make root.sandbox
```

This is very useful after you tweak configuration scripts in `conf.d/`
or files in `overlay/` without changing the list of packages in `plan/`.

If you change the plan, you need to rebuild from scratch:

```
root@tkldev products/core# make clean
root@tkldev products/core# make root.sandbox
```
