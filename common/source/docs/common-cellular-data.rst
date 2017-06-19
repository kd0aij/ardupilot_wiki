.. _common-cellular-data:

===========================================
Using Cellular Data for MAVLink Connections
===========================================

Using cellular data (e.g. 3G or 4G networks) is becoming a popular
method of communication with unmanned autonomous vehicles.

Methodology
===========

The chief disadvantage of using a cellular link for MAVLink is that
most providers will not easily provide a static IP address for a
cellular data link.  This leads to an addressability issue - how does
the GCS contact the UAV?

There are several ways this can be solved.

- dynamic DNS
- manually determining addresses and hoping they don't change during flight
- using a UDP proxy at a known location to bounce UDP packets off.  The UDP Proxy listens on two UDP ports and relays traffic between the two ports.  Your GCS connects to one of the ports, your vehicle to the other.

This document currently only describes the third option.

Setting up the UDP Proxy
========================

.. warning::

   The UDP Proxy setup described in this document is completely insecure.  We *know* it suffers from denial-of-service vulnerabilities, we *know* it suffers from a complete lack of privacy, and we *know* it suffers from a Man-in-the-Middle vulnerability - and those are just the problems we know of!  Should you choose to run this UDP proxy, you may wish to mitigate some or all of these; in particular, you should absolutely enable :ref:`mavlink2 with signing <common-mavlink2>`  before connecting your vehicle to the proxy.

.. note::

   It is assumed you will be using an AWS instance.  If you are not, please adjust as appropriate for your chosen system image.

- Work out which UDP ports you want to use for your UDP Proxy

  - in the examples below, ``9435`` and ``9436`` are used

- Log into AWS
- Create an AWS instance

  - T2 Nano is sufficient
  - 8GB storage
  - add your UDP ports to the ``Inbound`` firewall rules in the Security Group used by your VM.  You will need two rules, one for each of your ports.

    - Type: ``Custom UDP``
    - Protocol: ``UDP``
    - Port Range: *one of your port numbers*
    - Source: ``0.0.0.0/0``

  - You may also need ``Outbound`` Security Group rules if you have changed the AWS defaults.

Log in to your Virtual Machine and execute the following commands:

.. code-block:: shell

    time git clone https://github.com/canberrauav/cuav
    sudo apt-get update
    sudo apt-get install -y gcc
    pushd cuav/tools
    time gcc udpproxy.c -o ~/udpproxy

    NORMAL_USER=ubuntu
    mkdir "/home/$NORMAL_USER/udpproxy"
    SCRIPT="/home/$NORMAL_USER/udpproxy/autostart_udpproxy"
    LINE="sudo -H -u $NORMAL_USER /bin/bash -c '$SCRIPT'"
    sudo perl -pe "s%^exit 0%$LINE\\n\\nexit 0%" -i /etc/rc.local

    cat >$SCRIPT <<EOF
    #!/bin/bash

    set -e
    set -x

    TITLE=UDPProxy
    UDPPROXY_HOME="\$HOME/udpproxy"
    SCRIPT=\$UDPPROXY_HOME/start_udpproxy
    LOG=\$UDPPROXY_HOME/autostart_udpproxy.log

    # autostart for udpproxy
    (
    set -e
    set -x

    date
    set

    cd \$UDPPROXY_HOME
    screen -L -d -m -S "\$TITLE" -s /bin/bash \$SCRIPT
    ) >\$LOG 2>&1
    exit 0
    EOF

    chmod +x $SCRIPT

    SCRIPT2="/home/$NORMAL_USER/udpproxy/start_udpproxy"
    cat >$SCRIPT2 <<EOF
    #!/bin/bash

    set -e
    set -x

    BINARY="./udpproxy"

    PORT1=9435
    PORT2=9436

    \$BINARY \$PORT1 \$PORT2
    EOF
    chmod +x "$SCRIPT2"

After rebooting, you should find a screen session running (``screen -list``).  The UDP Proxy will be looping repeating ``opening sockets``

You should now be able to test your UDP proxy.


Testing the UDP Proxy using SITL
================================

This is convenient for testing your UDP Proxy before connecting your actual vehicle.  We will send the output from SITL to one of the ports and connect mavproxy to the other.

.. note::

   This section assumes you are running Linux.  Adjust as appropriate for other operating systems.

In a shell:

.. code-block:: shell

   IP=NNN.NNN.NNN.NNN
   PORT1=9435
   PORT2=9436

   ./Tools/autotest/sim_vehicle.py -v ArduCopter -m "--out udpout:$IP:$PORT1"
   # in mavproxy, remove the other output links:
   output remove 0
   output remove 0

.. code-block:: shell

   IP=NNN.NNN.NNN.NNN
   PORT1=9435
   PORT2=9436

   mavproxy.py --master udpout:$IP:$PORT2
   set source_system 254

If all is working, the second mavproxy should start to receive telemetry from the vehicle, passed through the MAVProxy instance used by SITL.


Using the link on a Companion Computer running APSync
=====================================================

Please read again the warning at the top of this page regarding enabling :ref:`mavlink2 with signing <common-mavlink2>` before using the UDP Proxy.

The first step is to plug in a cellular modem and get a network interface.  I was lucky enough to have a dongle which simply gives me a simple, working, internet-able ``usb0`` network interface, and Ubuntu simply reset my default route and sent all traffic via that interface.  Obtaining such a network interface is thus beyond the scope of this document.

On the companion computer, modify ``cmavnode.conf`` to send traffic via your UDP Proxy:

.. code-block:: shell

   cp start_cmavnode/cmavnode.conf{,-$(date '+%Y%m%d%H%M%S')}
   IP=NNN.NNN.NNN.NNN
   PORT1=9435
   PORT2=9436
   cat >>start_cmavnode/cmavnode.conf <<EOF

   [aws]
       type=udp
       targetip=$IP
       targetport=$PORT1
       localport=9009
       sim_enable=false
   EOF

   You may now connect your GCS to the other UDP port; e.g. in MAVProxy:

.. code-block:: shell

   IP=NNN.NNN.NNN.NNN
   PORT1=9435
   PORT2=9436

   mavproxy.py --master udpout:$IP:$PORT2
   set source_system 254
