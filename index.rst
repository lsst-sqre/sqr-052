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

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

Some of the provisioning steps in the current design require privileged
operations; for those, we should move those functions to privileged
initContainers so the main container runs only as the target user.  This
proposal outlines how we can do that.

.. Add content here.

The Problem
===========

We most definitely do not want to maintain a separate container build
per user.  Because of the way we are currently handling file permissions
and collaboration within the RSP, we create a Unix user with UIDs and
GIDs mapped from our authentication provider, and then we provision a
home directory for that user if it does not exist.

Currently, we use the ``provisionator`` (``769:769``) account to perform
those actions, which are privileged.  In order to do that, the container
build writes sudoers rules to allow the provisionator to act.  It also
allows the provisionator to write a rule (via a script) that allows the
provisionator to become the target user before starting Jupyterlab.

This requires ``sudo`` in order to work, and therefore the Lab
containers cannot be run (at least in environments that require
filesystem provisioning) with ``allowPrivilegeEscalation: False``.  This
violates the third item of section 4.1 of `SQR-048`_.

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

Recently, in order to allow loading the Multus CNI into containers in
Rubin-managed Kubernetes environments, necessary for control of
instruments on multicast networks with SAL, Josh Hoblitt introduced a
container that must be run as a privileged initContainer inside the
user's Lab pod before starting the Lab.  This change is already present
in the mainline Nublado environment.

A Fuller Solution
=================

Since we've worked out how to run privileged initContainers, all we
would need to do is to move the provisioning steps into a second
initContainer, and invoke it (or not) at user pod creation time.  We
could either restrict its privilege still further by doing the same set
of semi-privileged user plus sudoers rules we currently do, or just let
the initContainer run as root, which would simplify the logic within
considerably.

Then, in all cases, we could invoke the Lab pod directly with the
correct userid and set of groups, having provisioned those in ConfigMaps
to mount into ``/etc/passwd`` and ``/etc/group`` (``NO_SUDO`` already
does this), with the only piece that still requires privilege being the
creation of user filesystems.

This will be an initContainer that mounts our NFS share volume with
``no_root_squash`` and effectively runs as root while doing so in order
to create the user spaces.  That's a little scary, but no scarier than
what we are doing today, particularly if we restrict the provisionator's
actions with sudoers rules.

Future Steps
============

At some point we very likely will to want to replace this mechanism with
an actual JupyterHub-callable service that does the requested filesystem
provisioning.  The good thing about the proposed model is that, when we
do develop the service, then we just stop adding the privileged
provisioning container to ``initContainers`` at user pod startup, and
there's nothing else to do.  We cannot, with just the technique
suggested here, set a cluster-wide ``PodSecurityPolicy``, since every
user pod may have a privileged component.  However, the choice of
implementation of the Multus CNI may have tied our hands with respect to
this in any event.

Estimated Effort
================

Moving the privileged pieces to an initContainer model is at most a
week's worth of work, including testing.  The actual development work is
perhaps two days: it's basically just pulling the scripts out of the
``prov`` directory in the nublado JupyterLab build, and putting them
into an initContainer.

It will require changes to the spawning mechanism in the Hub, and will
allow us to remove the corresponding privileged scripts from the Lab
images.

Conclusion
==========

This is low-hanging fruit that will improve our overall security posture
and be very little effort.  The better option of providing a general
filesystem-provisioning service is perhaps somewhat more effort, but can
be tackled when resources allow, and migration between the two will be
trivial.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
