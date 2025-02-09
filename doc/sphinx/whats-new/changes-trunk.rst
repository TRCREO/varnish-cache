**Note: This is a working document for a future release, with running
updates for changes in the development branch. For changes in the
released versions of Varnish, see:** :ref:`whats-new-index`

.. _whatsnew_changes_CURRENT:

%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
Changes in Varnish **$NEXT_RELEASE**
%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

For information about updating your current Varnish deployment to the
new version, see :ref:`whatsnew_upgrading_CURRENT`.

A more detailed and technical account of changes in Varnish, with
links to issues that have been fixed and pull requests that have been
merged, may be found in the `change log`_.

.. _change log: https://github.com/varnishcache/varnish-cache/blob/master/doc/changes.rst

varnishd
========

Extensions
~~~~~~~~~~

From the very first days of Varnish, we have been talking about having
an extension points for "more advanced stuff" and we did, by and large,
keep a place ready for it in the overall architecture.

Now a credible use-case finally appeared, and we have implemented
"Varnish Extensions" (VTLA: "VEXT"), which can both be used to load
ambient VMODs and to implement entirely new functionaly, for instance
stevedores.

See :ref:`ref-vext` in the reference manual for more information.

Parameters
~~~~~~~~~~

Duration values (with a unit in seconds) can optionally take a duration
unit with the same syntax as VCL. For example, the default values of
``default_ttl``, ``default_grace`` and ``default_keep`` were changed
respectively from ``120.000``, ``10.000`` and ``0.000`` to ``2m``, ``10s``
and ``0s``.

The platform-dependent ``tcp_keepalive_time`` parameter is supported on
MacOS.

The new ``vcc_feature`` bits parameter replaces previous ``vcc_*`` boolean
parameters. The latter still exist as deprecated aliases.

Other changes in varnishd
~~~~~~~~~~~~~~~~~~~~~~~~~

The metadata VMODs exposes to Varnishd has changed to a non-binary
format, and it is incompatible with all previous releases.
That makes it possible for the VCC (compilation) process to avoid
opening the VMODs with ``dlopen(3)``, which is both faster and
safer.

Background fetch tasks are no longer queued as this could result in slow
grace hits subject to indefinite delays when thread pools are saturated.

Changes to VCL
==============

VCL variables
~~~~~~~~~~~~~

ESI sub-requests can no longer inherit a ``req.http.transfer-encoding``
header since the request body is strictly handled by the top request.

The ``resp.http.via`` header generated by Varnish uses ``server.identity``
which defaults to the host name. A ``req.http.via`` header is generated
also before entering ``vcl_recv``. If a client request or backend response
already had a Via header, it is now appended to instead of overwritten.

The ``server.identity`` variable is guaranteed to be a single token as
defined in the HTTP grammar, to safely be used as either a host name or
pseudonym in Via headers.

The ``now`` variable remains constant in a VCL subroutine. This was already
the case, but is now (pun intended) formally defined behavior. It keeps the
same value even if the execution blocks for a significant time, for example
while calling a VMOD function.

Bundled VMODs
=============

For a real time timestamp, the function ``std.now()`` can be used instead.
There is also a new ``std.timed_call()`` to measure the execution time of a
subroutine.

Cookie headers generated by vmod_cookie no longer have a spurious trailing
semi-colon (``';'``) at the end of the string.

varnishlog
==========

The ``Begin`` log records may contain a 4th field with the sub-level of
sub-tasks. The ``Begin[4]`` field is used by the ``-E`` option (or lack
thereof) in log utilities to include sub-tasks or not. Internally, only ESI
tasks are subject to this filtering, but it can apply to tasks spawned by
VMODs too.

Similarly, the ``Link`` record has the same optional 4th field.

.. XXX: any reason against ``varnish{hist,top} -k``?

The ``-k`` option from ``varnishlog`` is now available in ``varnishncsa``.

varnishstat
===========

The unused counter ``MAIN.fetch_no_thread`` was repurposed and renamed to
``MAIN.bgfetch_no_thread`` to signal when background fetch tasks fail to
be scheduled because thread pools are saturated.

To help estimate the rate of ``vsl_space`` consumption, the new counter
``MAIN.shm_bytes`` was added. It offers a finer-grained metric than the
existing ``MAIN.shm_cycles`` that depends on the ``vsl_space`` setting.

A new contribution script called ``varnishstatdiff`` can be used to compare
the output of two ``varnishstat -1`` executions with a friendly diff format
for ``varnishstat``'s specific output.

varnishtest
===========

New macros ``${pkg_version}`` and ``${pkg_branch}`` expanding respectively
to ``7.2.0`` and ``7.2`` for the current release.

It is possible to match the text on screen against a regular expression
with the new ``process -match`` command.

The new ``filewrite [-a]`` command can put or append text into a file.

A Varnish instance name in a VTC is used by default as the server identity
for predictable Via headers.

For example::

    varnish v1 -vcl+backend { ... }

The expected Via header is::

    Via: 1.1 v1 (Varnish/7.2)

The instance name can still be set to a different value using the ``-arg``
command to change the ``varnishd -i`` option.

Changes for developers and VMOD authors
=======================================

The ``varnishtest -i`` option only works from a Varnish source tree, in
which case the new macro ``${topsrc}`` is available in addition to the
old ``${topbuild}`` macro.

The functions ``VRT_AddVDP()``, ``VRT_AddVFP()``, ``VRT_RemoveVDP()`` and
``VRT_RemoveVFP()`` are deprecated.

The ``VCS_String()`` function can take the string ``"B"`` for the package
branch.

The ``vnum.h`` functions are exposed to VMOD and VEXT authors.

The termination rules for ``WRK_BgThread()`` were relaxed to allow VMODs to
use it.

*eof*
