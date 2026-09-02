.. _lab1_cloud_init:

#################################################
cloud-init — Declarative First-Boot Configuration
#################################################

.. contents::
   :local:
   :depth: 2

.. note::

   **Estimated Time:** 3–4 hours.
   
   **Environment Versions:** Ubuntu Server 26.04 LTS, cloud-init 24.x, LXD 5.x
   (or Incus 6.x). Verify the exact versions before
   you begin.
   
   **Note:** cloud-init's datasource names and schema evolve between releases.


Overview
========

In cloud-native infrastructure, we do not manually configure machines. We *describe the desired state* of a machine in a declarative document, and the platform realizes that state automatically when the machine boots. Machines become disposable (“cattle, not pets”): if one misbehaves, we destroy it and let the description rebuild an identical replacement.

``cloud-init`` <https://cloudinit.readthedocs.io/en/latest/>_ is the industry-standard tool for first-boot configuration. It executes during the initial boot of a virtual machine or container, reads a configuration document ``user-data`` provided by the platform (its **datasource**), and applies settings including user accounts, packages, files, and commands.

In this exercise, you will:

* **Create** a declarative specification,
* Allow the platform to **realize** it automatically, and then
* **Verify** convergence by demonstrating that the machine has reached the specified state you defined.

The final assessment (see :ref:`lab1_submission`) evaluates the resulting system state using an automated CI/CD pipeline, rather than the content of your configuration file. This approach embodies the core cloud-native discipline: *Describe*, *Realize*, and *Verify*.


Learning Objectives
====================

After completing this exercise you will be able to:

1. Explain, in your own words, the difference between *install-time* configuration (autoinstall) and *first-boot* configuration (cloud-init ``user-data``), and give an example directive from each.
2. Author a valid ``#cloud-config`` document that creates users with SSH access and correctly managed passwords, installs packages, writes files, and runs
   commands.
3. Deliver ``user-data`` to a machine through the **NoCloud** datasource (both a local seed and NoCloud-Net over HTTP) and explain how managed clouds provide the same
   data automatically.
4. Verify that a machine converged to its declared state using ``cloud-init status``, ``cloud-init query``, and ``cloud-init analyze``.
5. Reason about **idempotency** and re-run a configuration cleanly with ``cloud-init clean``.
6. Diagnose a failed configuration from ``/var/log/cloud-init*.log`` and the schema validator.


Prerequisites and Environment
==============================

.. important::

   Complete the exercise using a **single** environment, as it is intentionally single-track. All core walkthrough steps execute locally within an **LXD system container**. This approach eliminates the need for ISO mastering, USB drives, cloud accounts, or cloud-related expenses. LXD delivers ``user-data`` directly to ``cloud-init`` in the same manner as a real cloud environment. The assessment grader utilizes the same engine, ensuring that the local environment aligns with the grading system and maintains development and production parity, which is a key cloud-native principle.

You need a Linux host with:

- LXD 5.x (``sudo snap install lxd``) **or** Incus 6.x
  (``sudo apt install incus``). The commands below use ``lxc``; for Incus,
  substitute ``incus`` everywhere.
- ``cloud-init`` installed on the host for the schema validator
  (``sudo apt install cloud-init``). You do **not** need cloud-init to run on the
  host itself — you only use its ``schema`` subcommand there.
- ``git`` and a text editor.

Initialize LXD once:

.. code-block:: bash

   sudo lxd init --auto
   # Confirm the Ubuntu image server is reachable:
   lxc image list ubuntu:26.04

.. admonition:: Alternative for macOS / Windows
   :class: tip

   If you cannot run LXD natively, `Multipass <https://canonical.com/multipass>`_ accepts
   ``user-data`` with ``multipass launch --cloud-init user-data.yaml`` and behaves
   almost identically. The core walkthrough notes the Multipass equivalent where
   it differs. The **autograder still uses LXD**, so prefer LXD if you can.


Concepts 
========

Read this section before writing any YAML. These four ideas explain *why* the rest of the exercise behaves the way it does.

Boot Stages
-----------

`cloud-init` does not run all at once. It runs across several **stages** as the
system boots, so that configuration which needs the network runs after the network
is up, and so on:

#. **Local** (``init --local``) — before networking; detects the datasource, sets
   the hostname.
#. **Network** (``init``) — after networking; fetches remote ``user-data`` if needed.
#. **Config** (``modules --mode config``) — runs "config" modules (e.g. package
   installation).
#. **Final** (``modules --mode final``) — runs late modules such as ``runcmd``
   and ``write_files`` finalization, then prints the "done" status.

When you run the ``cloud-init status --wait`` command, you are waiting for the **Final**
stage to complete.

Datasource Detection
--------------------

A **datasource** is *where the* ``user-data`` *comes from*. On boot, ``cloud-init`` probes a
list of candidate datasources in order and uses the first that responds. On a
managed cloud (AWS, Azure, GCP, OpenStack) the matching datasource is detected
automatically and reads from the platform's metadata service. On bare metal, VMs,
and containers you provide the data yourself through the **NoCloud** datasource.
LXD presents ``user-data`` to ``cloud-init`` through the NoCloud datasource for you — which
is why launching a container with ``-c cloud-init.user-data=...`` "just works".

Modules and Run Frequency
-------------------------

Each unit of work is a **module** with a **frequency**:

- ``per-instance`` — runs once per unique ``instance-id`` (most modules; e.g. creating users). Re-running ``cloud-init`` will **not** repeat these unless the ``instance-id`` changes or you clean the cached state.
- ``per-boot`` — runs on every boot (e.g. ``bootcmd``).
- ``per-once`` — runs exactly once, ever.

This is the single most common source of "why didn't my change apply?" confusion. If you edit user-data and reboot, ``per-instance`` modules will **not** re-run, because ``cloud-init`` remembers it already configured this instance.

Idempotency and ``clean``
-------------------------

Because most modules are ``per-instance``, ``cloud-init`` is effectively **idempotent** by default: applying the same configuration twice does not create two users. When *you* write imperative steps (``runcmd``), **you** are responsible for keeping them idempotent — a naive ``echo x >> file`` in ``runcmd`` will append a second line if it ever runs twice. 

To force a genuine re-run during development (e.g. to test a fixed config on the same container), reset cloud-init's cached state:

.. code-block:: bash

   # inside the container
   sudo cloud-init clean --logs   # wipe state + logs; next boot re-runs everything
   sudo reboot

Configuration Document Structure
--------------------------------

The relationship between the two nesting contexts you will meet — the installer's ``autoinstall`` block and cloud-init's own ``user-data`` — looks like this:

.. graphviz::
   :align: center
   :caption: Where directives live. Installer-time directives sit directly under
             ``autoinstall``; first-boot directives sit under
             ``autoinstall.user-data`` (or stand alone in a plain ``#cloud-config``).

   digraph G {
       rankdir=TB;
       node [shape=box, style=filled, fillcolor=lightgray, fontname="Arial"];

       subgraph cluster_cloud_init {
           label = "#cloud-config document";
           style = rounded; color = black;

           subgraph cluster_autoinstall {
               label = "autoinstall:  (install-time only — see the Optional appendix)";
               style = rounded; color = gray;

               ai [label="version:\lidentity:\lstorage:\llate-commands:\l",
                   fillcolor=lightblue];

               subgraph cluster_userdata {
                   label = "user-data:  (first boot — the focus of this exercise)";
                   style = rounded; color = red;
                   ud [label="users:\lpackages:\lwrite_files:\lruncmd:\l",
                       fillcolor=lightpink];
               }
           }
       }
       ai -> ud [style=invis];
   }

.. tip::

   For the **core** of this exercise you write a *plain* ``#cloud-config`` — just the pink ``user-data`` directives, with no ``autoinstall`` wrapper. The ``autoinstall`` wrapper only matters when you are automating a full OS *installation*, which is the optional appendix.


Guided Walkthrough
==================

This is the spine of the exercise. Do the steps in order. After each step there is a **Checkpoint** — an explicit command whose output tells you the step succeeded. 

.. note::
   
   In cloud-native environments, success is always *observed*, never assumed.

Step 0 — Launch a Clean Instance
--------------------------------

Start with an empty configuration so you can see cloud-init's baseline before you change anything.

.. code-block:: bash

   printf '#cloud-config\n' > user-data.yaml
   lxc launch ubuntu:26.04 lab1 -c cloud-init.user-data="$(cat user-data.yaml)"
   lxc exec lab1 -- cloud-init status --wait

**Checkpoint:** ``cloud-init status --wait`` ends with ``status: done``. Inspect
what cloud-init knows about this instance:

.. code-block:: bash

   lxc exec lab1 -- cloud-init query --all | head -n 20
   lxc exec lab1 -- cloud-init analyze show | tail -n 15   # per-stage timing

.. note::

   **Multipass equivalent:**
   ``multipass launch 26.04 --name lab1 --cloud-init user-data.yaml`` then
   ``multipass exec lab1 -- cloud-init status --wait``.

Step 1 — Create a user with SSH access and a *managed* password
---------------------------------------------------------------

This is the step where most first-time users get locked out, so read the callout.

.. code-block:: yaml
   :caption: user-data.yaml
   :linenos:

   #cloud-config
   users:
     - name: ciuser
       groups: [sudo]
       shell: /bin/bash
       sudo: "ALL=(ALL) NOPASSWD:ALL"     # deliberate: passwordless sudo for the exercise
       lock_passwd: true                   # no usable password — SSH-key login only
       ssh_authorized_keys:
         - ssh-ed25519 AAAA...replace-with-your-public-key... you@example

.. warning::

   By default cloud-init **locks the password** of every user it creates
   (``lock_passwd: true``). That is correct and secure, but it means you cannot log
   in at the console with a password — you must use the SSH key. If you *need*
   console password login for testing, set ``lock_passwd: false`` **and** supply a
   ``hashed_passwd`` (never a plaintext password in a committed file). Generate one
   with ``openssl passwd -6``. For this exercise, keep ``lock_passwd: true`` and use
   your SSH key.

Generate a key first if you do not have one
(``ssh-keygen -t ed25519 -f ~/.ssh/lab1_key``) and paste the ``.pub`` contents.

.. warning::

   Re-apply the configuration on a fresh container (editing ``user-data`` does not re-run ``per-instance`` modules — see :ref:`Idempotency and clean <lab1_cloud_init>`):

.. code-block:: bash

   lxc delete -f lab1
   lxc launch ubuntu:26.04 lab1 -c cloud-init.user-data="$(cat user-data.yaml)"
   lxc exec lab1 -- cloud-init status --wait

**Checkpoint:**

.. code-block:: bash

   lxc exec lab1 -- id ciuser
   lxc exec lab1 -- sudo -l -U ciuser
   lxc exec lab1 -- getent shadow ciuser   # second field is '!' or '*' when locked

Step 2 — Install a Package and Manage a Service
-----------------------------------------------

Add ``nginx`` and confirm cloud-init both installs it and leaves the service
running.

.. code-block:: yaml
   :caption: add to user-data.yaml

   packages:
     - nginx

**Checkpoint** (after relaunching as in Step 1):

.. code-block:: bash

   lxc exec lab1 -- systemctl is-enabled nginx
   lxc exec lab1 -- systemctl is-active nginx

Step 3 — Write a File and Run a Command
---------------------------------------

Use ``write_files`` for static content and ``runcmd`` for a one-off action. Note
how the marker command is written to be **idempotent** (it overwrites, it does not
append):

.. code-block:: yaml
   :caption: add to user-data.yaml

   write_files:
     - path: /etc/lab1/motd
       owner: root:root
       permissions: "0644"
       content: |
         Provisioned by cloud-init for Cloud-Native Infrastructure Lab 1.

   runcmd:
     - [ mkdir, -p, /var/log/lab1 ]
     - [ sh, -c, "echo 'Provisioning complete' > /var/log/lab1/provision.log" ]

.. warning::

   ``runcmd`` runs late, as ``root``, once per instance. Prefer the list-of-lists
   form (each argument a list item) over a single shell string — it avoids quoting
   surprises. If you must append, guard it so a second run cannot duplicate the
   line (e.g. ``grep -q MARKER file || echo MARKER >> file``).

**Checkpoint:**

.. code-block:: bash

   lxc exec lab1 -- cat /etc/lab1/motd
   lxc exec lab1 -- cat /var/log/lab1/provision.log

Step 4 — Prove Idempotency
--------------------------

Re-run ``cloud-init`` on the *same* container and confirm nothing breaks and nothing
duplicates:

.. code-block:: bash

   lxc exec lab1 -- cloud-init clean --logs
   lxc restart lab1
   lxc exec lab1 -- cloud-init status --wait
   lxc exec lab1 -- sh -c 'grep -c "Provisioning complete" /var/log/lab1/provision.log'

**Checkpoint:** status is ``done`` and the ``grep -c`` count is exactly ``1``. If it
is ``2``, your ``runcmd`` is not idempotent — fix it before moving on. This is
exactly the property the autograder checks.


The NoCloud Datasource, Explained
=================================

In Step 0 LXD quietly acted as your datasource. To understand what real
provisioning systems do, deliver the *same* ``user-data`` yourself through the
**NoCloud** datasource — the transport that generalizes to bare-metal installs and
custom provisioning systems. NoCloud can take its seed either 

1. **Over the network** (from an HTTP server), or 
2. **From a local labeled volume** such as a USB stick or a seed ISO. 

You will meet both below; the ``#cloud-config`` document is identical in
each case — only the delivery changes.

NoCloud needs two files, whichever transport you use:

- ``meta-data`` — instance metadata (at minimum an ``instance-id``).
- ``user-data`` — your ``#cloud-config`` (the same document from the walkthrough).

Over the network (HTTP)
-----------------------

Serve the two files with Python's built-in server:

.. code-block:: bash
   :linenos:

   mkdir -p ~/cloud-init-data
   printf 'instance-id: nocloud-001\nlocal-hostname: lab1\n' \
       > ~/cloud-init-data/meta-data
   # IMPORTANT: use printf (or echo -e) so the newlines are real and
   # '#cloud-config' is genuinely the first line — a literal '\n' produces an
   # invalid seed that cloud-init silently rejects.
   cp user-data.yaml ~/cloud-init-data/user-data
   ( cd ~/cloud-init-data && python3 -m http.server 8080 )

.. warning::

   The classic beginner bug is ``echo "#cloud-config\nruncmd: ..."`` **without**
   ``-e``. That writes the literal characters ``\n`` into the file, so the whole
   document collapses onto one line, ``#cloud-config`` is no longer the first line,
   and ``cloud-init`` discards it with no obvious error. Always verify with
   ``head -n1 user-data`` that the first line is exactly ``#cloud-config``.

A machine is then pointed at this server through its kernel command line:

.. code-block:: text

   ds=nocloud;s=http://<your-ip>:8080/

.. note::

   Recent ``cloud-init`` accepts ``ds=nocloud`` for both local and HTTP seeds and
   infers the transport from the URL scheme; older releases used a separate
   ``nocloud-net`` name. Use whichever your pinned version documents — check with
   ``cloud-init --version`` and the datasource reference.

On Bare Metal, from a USB drive
-------------------------------

A bare-metal machine has no cloud metadata service, and often no network at first
boot. The classic answer is to hand ``cloud-init`` its seed on **removable media**: a USB
stick (or a small ISO) whose filesystem is **labelled** ``CIDATA``. During the local
boot stage ``cloud-init`` probes attached volumes, finds the ``CIDATA`` label, reads
``user-data`` and ``meta-data`` from it, and provisions the node — no kernel arguments
and no network required. This is the same NoCloud datasource as above, just delivered
by disk instead of HTTP.

The target node must boot an image that already has ``cloud-init`` installed and that has
**not yet been initialized** — a fresh Ubuntu cloud or server image, or any system on
which you have just run ``sudo cloud-init clean --logs``. Give each node a **unique**
``instance-id`` so ``cloud-init`` treats it as a new instance and runs the
``per-instance`` modules.

**Step 1 — Prepare the two seed files** (reuse the document from the walkthrough):

.. code-block:: bash

   mkdir -p ~/seed
   printf 'instance-id: node-0001\nlocal-hostname: node01\n' > ~/seed/meta-data
   cp user-data.yaml ~/seed/user-data
   head -n1 ~/seed/user-data          # must print exactly: #cloud-config

**Step 2 — Write the seed onto the USB stick.** Identify the device with great care:
writing to the wrong disk destroys data.

.. code-block:: bash

   lsblk -o NAME,SIZE,TRAN,MOUNTPOINT      # find your USB, e.g. /dev/sdX

   # Create ONE FAT filesystem labelled CIDATA on the stick — this ERASES it:
   sudo mkfs.vfat -n CIDATA /dev/sdX1

   sudo mkdir -p /mnt/cidata
   sudo mount /dev/sdX1 /mnt/cidata
   sudo cp ~/seed/user-data ~/seed/meta-data /mnt/cidata/
   sudo umount /mnt/cidata
   sudo blkid /dev/sdX1                     # confirm: LABEL="CIDATA" TYPE="vfat"

.. admonition:: Alternative — build a CIDATA ISO instead of formatting a stick
   :class: tip

   ``cloud-localds`` packages the two files into a correctly labelled seed image in
   one step: ``cloud-localds seed.iso ~/seed/user-data ~/seed/meta-data``. Write that
   image to a USB with
   ``sudo dd if=seed.iso of=/dev/sdX bs=4M status=progress oflag=sync`` (again,
   double-check the device), or attach it directly to a VM as a virtual CD-ROM.

**Step 3 — Provision the node.** Insert the USB into the bare-metal machine and boot
(or reboot) it. ``cloud-init`` detects the ``CIDATA`` volume during early boot and applies
your configuration automatically.

**Checkpoint** (run on the node once it has booted):

.. code-block:: bash

   cloud-init status --wait                 # ends with: status: done
   cloud-id                                 # prints the detected datasource: nocloud
   sudo cloud-init query --all | grep -iE '"(sub)?platform"'   # shows the seed source

.. warning::

   Treat the stick as a secret. ``user-data`` frequently carries SSH keys, hashed
   passwords, or tokens, and anyone holding the USB can read them. Remove the media
   after provisioning, and never put a plaintext password on it — use
   ``ssh_authorized_keys`` or a ``hashed_passwd`` instead.

.. seealso::

   `NoCloud datasource reference
   <https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html>`_
   and `cloud-localds
   <https://manpages.debian.org/testing/cloud-image-utils/cloud-localds.1.en.html>`_,
   a helper that packages ``meta-data`` and ``user-data`` into a seed image for VMs.


How this Maps to Clouds Provider Environments
=============================================

You will not run these in the lab — they cost money and need accounts. Read them as
*reference*: every managed cloud provides your ``user-data`` through its own datasource,
which ``cloud-init`` detects automatically. The document you wrote in the walkthrough
is portable across all of them; only the delivery mechanism changes.

.. list-table:: The same ``user-data``, delivered by each platform's datasource
   :header-rows: 1
   :widths: 20 20 60

   * - Platform
     - Datasource
     - How user-data is supplied (reference only — **do not run; incurs cost**)
   * - Local VM / container
     - NoCloud
     - Seed image, or ``-c cloud-init.``user-data``=`` (LXD), or ``--cloud-init`` (Multipass)
   * - AWS EC2
     - Ec2
     - ``aws ec2 run-instances --user-data file://user-data.yaml``
   * - Azure
     - Azure (IMDS)
     - ``az vm create --custom-data user-data.yaml`` (use a current image URN, e.g. ``Canonical:ubuntu-24_04-lts:server:latest``)
   * - Google Cloud
     - GCE
     - ``gcloud compute instances create ... --metadata-from-file user-data=user-data.yaml``
   * - OpenStack
     - ConfigDrive / Metadata
     - ``openstack server create --user-data user-data.yaml ...``

The point to internalize: **Write the desired state once. Run it anywhere.** That portability is the payoff of declarative, datasource-agnostic configuration.


.. _lab1_autoinstall_appendix:

Optional Appendix — Automating a full OS install (autoinstall)
==============================================================

.. admonition:: (Optional) Advanced
   :class: caution

   Everything above configures a machine that is *already installed*. **Autoinstall**
   automates the *installation itself* (partitioning, base packages, first user) via
   Ubuntu's **Subiquity** installer. It is heavier, slower, and hardware-dependent.

Autoinstall is implemented as a ``cloud-init`` module. Its directives live under an
``autoinstall:`` key; anything you want applied on the *first boot of the installed
system* goes under ``autoinstall.user-data`` (the ordinary ``#cloud-config`` schema
you already know).

.. code-block:: yaml
   :caption: Corrected autoinstall example — note where ``runcmd`` belongs
   :linenos:

   #cloud-config
   autoinstall:
     version: 1
     packages:
       - cowsay
     late-commands:
       # runs in the installer environment, after install, before first reboot
       - curtin in-target --target=/target -- systemctl enable cowsay.service || true
     user-data:            # <-- ordinary cloud-init, runs on the installed system's first boot
       users:
         - name: ciuser
           groups: [sudo]
           shell: /bin/bash
           sudo: "ALL=(ALL) NOPASSWD:ALL"
           lock_passwd: true
           ssh_authorized_keys:
             - ssh-ed25519 AAAA...your-key...
       runcmd:
         - echo "Hello from first boot" > /var/log/firstboot.log

.. warning::

   **Common mistake this corrects:** ``runcmd`` is *not* an ``autoinstall`` directive.
   Placing it directly under ``autoinstall:`` (as a sibling of ``version:`` and
   ``packages:``) is invalid. For installer-time actions use ``late-commands``; for
   first-boot actions put ``runcmd`` **inside** ``autoinstall.user-data``. Mixing
   these up is the single most common ``autoinstall`` error and is exactly the timing
   distinction Objective 1 asks you to explain.

To trigger ``autoinstall`` unattended, append ``autoinstall`` (and, for a remote seed,
the ``ds=`` parameter) to the installer's GRUB kernel line — press ``e`` at the GRUB
menu, edit the ``linux /casper/vmlinuz ...`` line, and boot with ``Ctrl+X``:

.. code-block:: text

   linux /casper/vmlinuz ... quiet autoinstall ds=nocloud;s=http://192.168.1.100:8080/ --

.. seealso::

   `Introduction to autoinstall
   <https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html>`_
   and the `autoinstall reference
   <https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html>`_.


Troubleshooting and verification reference
==========================================

Keep this handy — most of these commands are also how you *demonstrate* success,
not just how you debug failure.

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Command
     - What it tells you
   * - ``cloud-init status --wait --long``
     - Blocks until the Final stage finishes; prints ``done``, ``degraded``, or ``error``.
   * - ``cloud-init query --all``
     - The instance metadata and merged config cloud-init actually used.
   * - ``cloud-init query userdata``
     - The exact user-data cloud-init received (confirms your document arrived intact).
   * - ``cloud-init analyze show`` / ``analyze blame``
     - Per-stage and per-module timing; ``blame`` ranks the slowest modules.
   * - ``cloud-init schema --config-file user-data.yaml``
     - Validates your document **before** you boot anything. Run this every time.
   * - ``cloud-init schema -c user-data.yaml --annotate``
     - Same, but points at the exact offending lines.
   * - ``cloud-init clean --logs``
     - Resets cached state so the next boot re-runs everything (development only).
   * - ``sudo cat /var/log/cloud-init.log``
     - The detailed engine log — start here when status is ``error``.
   * - ``sudo cat /var/log/cloud-init-output.log``
     - stdout/stderr of ``runcmd`` and package installs — start here when a *command* failed.

A good debugging loop: validate the schema → launch → ``status --wait`` → if not
``done``, read ``cloud-init.log``; if a command failed, read
``cloud-init-output.log`` → fix → ``clean --logs`` and relaunch.


.. _lab1_submission:

Submission and Assessment
=========================

Clone the lab template repository available at `cloud-lab-templates <https://github.com/purduecyan/cloud-lab-templates>`_. See the instruction in ``README.md`` in the ``lab1-template`` folder for how to submit your work.


.. tip::

   Two requirements are deliberately under-specified in this rubric — the locked
   password and idempotency. They are not tricks: they follow directly from the
   *concepts* section. If you understood run frequency and cloud-init's password
   defaults, you already know what to do.


.. seealso::

   #. `cloud-init official documentation <https://cloudinit.readthedocs.io/en/latest/>`_
   #. `cloud-config examples <https://cloudinit.readthedocs.io/en/latest/reference/examples.html>`_
   #. `NoCloud datasource <https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html>`_
   #. `Ubuntu autoinstall <https://canonical-subiquity.readthedocs-hosted.com/en/latest/>`_
   #. `LXD instance configuration (cloud-init keys) <https://documentation.ubuntu.com/lxd/en/latest/cloud-init/>`_
