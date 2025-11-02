PRoot
=====

*chroot, mount --bind, and binfmt_misc without privilege/setup for Linux*

Build status
============

**Please take the PRoot Usage Survey for 2023!**

.. image:: https://img.shields.io/badge/survey-2023-green?style=flat-square
   :target: https://www.surveymonkey.com/r/7GVXS7W
   
--

.. image:: https://img.shields.io/gitlab/pipeline/proot/proot.svg?label=gitlab-ci&style=flat-square
   :target: https://gitlab.com/proot/proot/pipelines

.. image:: https://img.shields.io/badge/scan--build-latest-yellow.svg?style=flat-square
   :target: https://proot.gitlab.io/proot/reports/scan-build

.. image:: https://img.shields.io/badge/lcov-latest-6688D4.svg?style=flat-square
   :target: https://proot.gitlab.io/proot/reports/lcov

.. image:: https://img.shields.io/cii/summary/2444.svg?label=cii-best-practices&style=flat-square
   :target: https://bestpractices.coreinfrastructure.org/projects/2444

.. image:: https://img.shields.io/badge/DOI-10.5281%2Fzenodo.5371409-blue?style=flat-square
   :target: https://doi.org/10.5281/zenodo.5371409

Compiling
=========

The project now ships with a CMake build system; a typical build looks like::

    cmake -S . -B build
    cmake --build build

The binaries ``build/src/proot`` and ``build/src/care`` will be produced in the
configured build directory.  By default the CMake build downloads the required
``libtalloc`` and ``libarchive`` dependencies automatically; pass
``-DPROOT_FETCH_DEPENDENCIES=OFF`` if you prefer to link against system
packages.  You can still invoke the legacy ``make -C test`` target to run the
historical test suite if required.

|asciicast|

.. |asciicast| image:: https://asciinema.org/a/315367.svg
   :target: https://asciinema.org/a/315367

Dependencies
============

- `libarchive <https://libarchive.org>`_
- `libtalloc <https://talloc.samba.org>`_
- `uthash <https://troydhanson.github.io/uthash>`_ (only required for building CARE)

Manuals
=======

- `PRoot <https://github.com/proot-me/proot/blob/master/doc/proot/manual.rst#proot>`_

- `CARE <https://github.com/proot-me/proot/blob/master/doc/care/manual.rst#care>`_

Support
=======

- `Mailing List <mailto:proot_me@googlegroups.com>`_
- `Forum <https://groups.google.com/forum/?fromgroups#!forum/proot_me>`_
- `Chat <https://gitter.im/proot-me/devs>`_

License
=======

SPDX-License-Identifier: `GPL-2.0-or-later <COPYING>`_
