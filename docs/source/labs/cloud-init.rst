**********
Cloud-init
**********


.. contents::
   :local:
   :depth: 2


Introduction
============

Cloud-init is a widely used tool for customizing cloud instances during the boot process. It enables automatic configuration of virtual machines by applying user-defined settings such as:

- Setting hostnames
- Creating users and groups
- Installing packages
- Running scripts

Cloud-init supports multiple data sources and is commonly used in all major cloud platforms like AWS, Azure, and OpenStack. It reads configuration from metadata services or configuration files (e.g., ``#cloud-config``) and applies them during instance initialization.

For more information, visit the `cloud-init documentation <https://cloudinit.readthedocs.io/en/latest/>`_.


.. _cloud_init_config:

Cloud-Init Configuration
========================

``cloud-init`` is a powerful tool used to automate the initialization of cloud instances. It reads configuration data from various sources and applies settings during the first boot of a virtual machine.

The configuration is usually provided in a file starting with the header ``#cloud-config``, written in **YAML** format. It supports multiple modules and directives, such as:

- ``users``: Create and configure users
- ``packages``: Install software packages
- ``runcmd``: Run shell commands
- ``write_files``: Create files with specified content

Cloud-init also supports **autoinstall** for unattended OS installations, where configuration is nested under the ``autoinstall`` key.

Example structure:

.. code-block:: yaml
    :linenos:
    :caption: Example ``cloud-init`` configuration with autoinstall and user-data

    #cloud-config
    autoinstall:
      version: 1
      packages:
        - cowsay
      user-data:
        users:
          - name: ciuser
            sudo: ALL=(ALL) NOPASSWD:ALL
            shell: /bin/bash
      runcmd:
        - echo "Hello from cloud-init!"


Cloud-Init vs Autoinstall
=========================

Cloud-init and autoinstall are both tools used in Ubuntu systems to automate setup, but they serve different purposes and operate at different stages of the provisioning lifecycle.

- **Cloud-init** is used to configure a system *after* it has been installed, typically during the first boot.
- **Autoinstall** is used to automate the *installation process itself*, including disk partitioning, user creation, and package selection.

Subiquity
---------

Subiquity is the modern installer used in Ubuntu Server editions. It replaces the older Debian-based installer and supports **autoinstall** for fully automated, unattended installations. Subiquity reads configuration from a YAML file embedded in a ``#cloud-config`` document and executes the installation accordingly.

Cloud-Init and Autoinstall Interaction
--------------------------------------

Autoinstall is implemented as a module within cloud-init. During installation, cloud-init processes the ``autoinstall`` section of the configuration file to guide Subiquity through the installation steps. After installation, cloud-init continues to configure the system using the ``user-data`` section on first boot.


.. list-table:: Comparison of Autoinstall and Cloud-init
   :header-rows: 1
   :widths: 25 25 50

   * - Feature
     - Autoinstall
     - Cloud-init
   * - Purpose
     - Automates OS installation
     - Configures system post-install
   * - Trigger
     - During installation
     - On first boot
   * - Configuration Format
     - YAML under ``autoinstall`` key
     - YAML with ``#cloud-config`` header
   * - Common Use
     - Ubuntu Server, cloud images
     - Cloud VMs, custom boot setups
   * - Supported Installer
     - Subiquity
     - Cloud-init engine
   * - Desktop Support
     - No (Ubiquity used)
     - Yes (limited)

Configuration Hierarchy 
-----------------------

The configuration hierarchy in ``cloud-init`` can be visualized as follows:

.. graphviz::
    :align: center
    :caption: Cloud-init Configuration Structure (autoinstall and user-data sections)

    digraph G {
        rankdir=TB;
        compound=true;
        node [shape=box, style=filled, fillcolor=lightgray, fontname="Helvetica"];
        edge [dir=none,style=invis]

        subgraph cluster_cloud_init{

            subgraph cluster_autoinstall{
                rankdir=TB;

                subgraph cluster_autoinstall_directives{
                    rankdir=TB;
                    autoinstalldirectives [label="version:\linteractive-sections:\learly-commands:\l", style=filled, fillcolor=lightblue];
                    label="autoinstall directives:";
                    style = rounded;
                    color = blue;
                }
                subgraph cluster_userdata{
                    rankdir=TB;
                    userdata [label="user-data:\l    users:\l", style=filled, fillcolor=lightpink];
                    label="user-data directives:";
                    style = rounded;
                    color = red;
                }

                label = "autoinstall:";
                style = rounded;
                color = gray;
            }

            label = "cloud-init";
            style = rounded;
            color = black;
        }
        autoinstalldirectives -> userdata;
        

    }


.. _nocloud_datasource:

NoCloud Data Source
===================

The ``NoCloud`` data source is a generic method for providing ``meta-data`` and ``user-data`` to ``cloud-init``. It is ideal for environments without native cloud metadata services, such as bare-metal servers, virtual machines, or custom provisioning systems.

Overview
--------

The NoCloud data source supports two modes:

- **NoCloud (local disk)**: Uses a filesystem (e.g., ISO9660 or VFAT) with a volume label `CIDATA` containing configuration files.
- **NoCloud (local image)**: Uses a mounted filesystem (e.g., ISO, disk image).
- **NoCloud-Net**: Fetches data from a remote HTTP server.

Required Files
--------------

The following files must be present in the data source:

- ``meta-data``: Contains instance metadata (hostname, instance-id, etc.).
- ``user-data``: Contains cloud-config or shell scripts for provisioning.

Optional files:

- ``vendor-data``: Additional configuration from vendor.
- ``network-config``: Network configuration in YAML format.

Example: ``meta-data``
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    instance-id: nocloud-instance-001
    local-hostname: myserver

Example: ``user-data``
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    #cloud-config
    users:
      - name: testuser
        sudo: ALL=(ALL) NOPASSWD:ALL
        groups: users
        shell: /bin/bash
    runcmd:
      - echo "Provisioning complete" > /var/log/provision.log


.. important:: The above example is missing the ``autoinstall`` section. For unattended installations, See the :ref:`cloud_init_config`  section.


Provisioning Workflow
---------------------


NoCloud (local disk): Creating a USB Drive labeled CIDATA
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To use a USB drive as the NoCloud data source:

1. **Create configuration files**:

   .. code-block:: bash

      mkdir -p /tmp/nocloud
      echo "instance-id: nocloud-001" > /tmp/nocloud/meta-data
      echo -e "#cloud-config\nruncmd:\n  - echo Hello > /tmp/hello.txt" > /tmp/nocloud/user-data

2. **Create a VFAT filesystem image**:

   .. code-block:: bash

      truncate --size 2M seed.img
      mkfs.vfat -n CIDATA seed.img

3. **Copy configuration files to the image**:

   .. code-block:: bash

      mcopy -oi seed.img /tmp/nocloud/meta-data ::meta-data
      mcopy -oi seed.img /tmp/nocloud/user-data ::user-data

4. **Write image to USB drive**:

   Identify your USB device (e.g., ``/dev/sdX``) and write the image:

   .. code-block:: bash

      sudo dd if=seed.img of=/dev/sdX bs=4M status=progress && sync

.. warning:: Ensure ``/dev/sdX`` is the correct USB device to avoid data loss.

1. **Boot the target system with the USB drive inserted**:

   Cloud-init will detect the ``CIDATA`` volume and apply the configuration.

Alternative: NoCloud (local image): ISO Image
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can also create an ISO image:

1. **Create ISO or directory with required files**:

   .. code-block:: bash

      mkdir -p /tmp/nocloud
      echo "instance-id: nocloud-001" > /tmp/nocloud/meta-data
      echo -e "#cloud-config\nruncmd:\n  - echo Hello > /tmp/hello.txt" > /tmp/nocloud/user-data

2. **Create ISO image (optional)**:

   .. code-block:: bash

      genisoimage -output seed.iso -volid cidata -joliet -rock /tmp/nocloud/user-data /tmp/nocloud/meta-data

3. **Attach ISO to VM or mount directory**:

   - For KVM/QEMU:

     .. code-block:: bash

        qemu-system-x86_64 -cdrom nocloud.iso ...

   - For cloud-init testing:

     .. code-block:: bash

        sudo cloud-init single --file /tmp/nocloud/user-data --name runcmd --frequency always

4. **Boot the system**:

   Cloud-init will detect the NoCloud data source and apply the configuration.

NoCloud-Net: Kernel Command Line
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To use NoCloud-Net via HTTP:

.. code-block:: bash

   ds=nocloud-net;s=http://<your-server>/cloud-init/

Ensure the HTTP server serves ``meta-data`` and ``user-data`` files at the root of the specified path.


As an example, to serve the configuration files using a Python HTTP server on port 8080:

1. **Create a directory with configuration files**:

   .. code-block:: bash

      mkdir -p ~/cloud-init-data
      echo "instance-id: nocloud-net-001" > ~/cloud-init-data/meta-data
      echo "#cloud-config\nruncmd:\n - echo Hello from NoCloud-Net > /tmp/hello.txt" > ~/cloud-init-data/user-data

2. **Start Python HTTP server**:

   .. code-block:: bash

      cd ~/cloud-init-data
      python3 -m http.server 8080

   This will serve files at `http://<your-ip>:8080/`.

3. **Configure kernel command line on target system**:

   Add the following to the boot parameters:

   .. code-block:: bash

      ds=nocloud-net;s=http://<your-ip>:8080/

   Replace ``<your-ip>`` with the IP address of the server running the Python web server.

4. **Boot the target system**:

   Cloud-init will fetch ``meta-data` and ``user-data`` from the specified URL and apply the configuration.



.. _grub_autoinstall:

Using GRUB to Enable Autoinstall with Cloud-Init
================================================

To automate OS installation using cloud-init and avoid manual confirmation prompts, you can modify the GRUB boot parameters to include the ``autoinstall`` directive.

This is especially useful when using the **NoCloud** or **NoCloud-Net** data sources for unattended installations.

Editing GRUB Kernel Line
------------------------

1. **Boot into the installer ISO or PXE environment**.

2. **At the GRUB menu**, press ``e`` to edit the boot entry.

3. **Locate the line starting with** ``linux`` or ``linuxefi``. It typically looks like:

   .. code-block:: bash

      linux /casper/vmlinuz ... quiet --

4. **Append one of the following to the end of the line**:

   .. code-block:: bash

      # For NoCloud with USB
      autoinstall
      
      # For NoCloud-Net with HTTP server
      autoinstall ds=nocloud-net;s=http://<your-server>:<port>/

   Replace ``<your-server>`` and ``<port>`` with the IP address or hostname and the port of the server hosting your ``meta-data`` and ``user-data`` files.

5. **Edited GRUB kernel line example**:

   .. code-block:: bash

      linux /casper/vmlinuz ... quiet autoinstall ds=nocloud-net;s=http://192.168.1.100:8080/ --

6. **Press `Ctrl + X` or `F10`** to boot with the modified parameters.

This will trigger the autoinstall process using the provided cloud-init configuration without any user interaction.

.. important:: 
      * The ``autoinstall`` keyword is required for Ubuntu Server 20.04+ and other cloud-init enabled installers to bypass confirmation.
      * Ensure your HTTP server is running and accessible before booting the target system.
      * Optional: You can also use ``ds=nocloud;s=/media/usb/`` if using a USB drive with a ``CIDATA`` label.


.. seealso::

   1. `Ubuntu Autoinstall Docs <https://ubuntu.com/server/docs/install/autoinstall>`_
   2.  `Cloud-Init NoCloud Data Source <https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html>`_





Troubleshooting
===============

- Validate cloud-config:

  .. code-block:: bash

     cloud-init devel schema --config-file user-data

- View logs:

  .. code-block:: bash

     cat /var/log/cloud-init.log
     cat /var/log/cloud-init-output.log


.. seealso::

    1. `NoCloud Data Source Documentation <https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html>`_
    2. `Cloud-Init Official Docs <https://cloudinit.readthedocs.io/en/latest/>`_
    3. `Autoinstall configuration reference manual <https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html>`_
    4. `Introduction to autoinstall <https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html>`_
    5. `Cloud-config examples <https://cloudinit.readthedocs.io/en/latest/reference/examples.html>`_



