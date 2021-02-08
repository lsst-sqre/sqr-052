..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

The current design for automated user file storage provisioning requires
that the user Lab pod be run without ``allowPrivilegeEscalation =
False``.  This technote describes how to improve that design to block
that avenue of attack.

.. Add content here.

The Problem
===========

In an RSP environment, we need some way to manage resources for users.
The most obvious of these resources is per user file storage space
(which in our current design is presented as a home directory).

We most definitely do not want to maintain a separate container build
per user.  Because of the way we are currently handling file permissions
and collaboration within the RSP, we create a Unix user with UIDs and
GIDs mapped from our authentication provider, and then we provision a
home directory for that user if it does not exist.

Currently, we use the ``provisionator`` (``769:769``) account to perform
those actions, which are privileged, within the Lab container.  In order
to do that, the container build writes sudoers rules to allow the
provisionator to act.  It also allows the provisionator to write a rule
(via a script) that allows the provisionator to become the target user
before starting Jupyterlab.

This requires ``sudo`` in order to work, and therefore the Lab
containers cannot currently be run (at least in environments that
require filesystem provisioning) with ``allowPrivilegeEscalation:
False``.  This violates the third item of section 4.1 of `SQR-048`_.

.. _SQR-048: https://sqr-048.lsst.io/

The Partial Solution
====================

A year or more ago we introduced a ``NO_SUDO`` flag environment variable
that can be passed into the container, which bypasses all of the sudo
steps to escalate privilege.

This comes with one major caveat: it only works if there is a
pre-provisioned user home directory.  This is fine at NCSA (which is
where the rule was intended for use) since user home space is
provisioned as part of account creation and NCSA forces all RSP users to
have NCSA accounts; it is not fine in a GKE environment using GitHub as
its authentication back-end.  We have recently proven that everything
still works as expected (assuming a pre-provisioned filesystem) if the
container is run with ``allowPrivilegeEscalation: False`` in its
Kubernetes ``securityContext``.

A Better Solution
=================

What we really want is an administrative service, callable from within
the RSP Kubernetes cluster, which can perform privileged actions on
behalf of users.  The first use-case of this service, to which we have
given the moniker "Moneypenny," will be to provision home directories
for users.

Design
------

In order for this to work, we need a service that can receive web
requests, authenticate them via some model, and spawn containers with
sufficient privilege to perform the requested work.

For the initial case of provisioning the home directories, the scripts
to create those directories with the right user and group IDs already
exist.  We simply need to move them into a minimal container of their
own, and ensure that that container is created with sufficient
privilege.

Authentication can (and should) be handled by ingress annotations, just
like everything else in the RSP platform.  It should require the
``exec:admin`` mapping.  That will require that nublado/nublado2 gain
the ability to request or mint an admin token.  The code to create that
token already exists within mobu.

However, we may initially want to do request checking within the
application itself: until we have restrictive intra-cluster network
policies in place, there is nothing preventing any nublado user from
talking to the Moneypenny service or endpoint directly, and creating
arbitrary user directories.  While this isn't a data corruption problem
yet, when Moneypenny becomes able to deprovision user directories (which
will surely be a future enhancement), it will be, and in any event a
user could create a home directory with an incorrect UID, thus
preventing future creation of the directory built correctly.  The
workflow dispatcher and the RubinSpawner implementation in the Hub
already do this sort of token verification in their operation, so
borrowing the authorization implementation from one or the other will be
easy.

That leaves the web service itself.  We should probably use a model like
cachemachine, which already provides a RESTful interface via aiohttp,
and which also creates containers on the back end.  Thus, I propose to
start with cachemachine and use a very similar set of objects and
classes, but replace the handlers, since the work performed will be
different.

Diagram
-------

.. figure:: /_static/Moneypenny.png
    :name: Moneypenny-diagram

Moneypenny Diagram

Scope of Work
-------------

I think the whole project is less than two weeks worth of work for me.
The creation of the container to provision user home directories is
trivial, as are the required annotations.

Adapting Cachemachine and gluing in an authentication/authorization
mechanism should be relatively easy, albeit nontrivial.  Modifications
to the Hub's spawning mechanism to acquire a token and then call
Moneypenny will also be easy although nontrivial.

Once Moneypenny is functional and deployed in all non-NCSA RSP
environments, that will allow us to remove the need for privilege
escalation from the Lab for future image builds; that is nearly
negligible effort.

Conclusion
==========

This is a task that will improve our overall security posture
and be fairly little effort.  We should undertake it with alacrity.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
