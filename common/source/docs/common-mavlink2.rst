.. _common-mavlink2:

=============================
Using MAVLink2 with ArduPilot
=============================

MAVLink2 extends MAVLink1 in two important ways:

- increases the namespace available for messages
- allows MAVLink packets to be cryptographically signed to defend against Man-in-the-Middle attacks

This document explains show to enable mavlink2 and how to enable packet signing.

.. warning::

   Consider having multiple links to your available during setup.  You may lock yourself out of the vehicle accidentally!

Enabling MAVLink2
=================

MAVLink2 is enabled on a per-link basis.  Determine the link on which you wish to enable MAVLink - typically SERIAL1.

Set the relevant parameter using your ground control station:


MAVProxy
--------

.. code-block:: shell

   param set SERIAL1_PROTOCOL 2


You will need to restart ArduPilot for this parameter change to take effect.

You may also need to restart your GCS for changes to take effect.  MAVProxy may require ``--mav20``.

Enabling Packet Signing
=======================

Packet signing requires knowledge of a shared secret ("key") between
the GCS and the autopilot.  The protocol has facility for exchanging
this secret; this should be done by connecting over a "secure" link
such as USB.

Key Exchange
------------

Key exchange using MAVProxy:
............................

Remember to start MAVProxy in MAVLink2 mode using ``--mav20``

.. code-block:: shell

   signing setup SWORDFISH

Using your own passphrase is recommended.
