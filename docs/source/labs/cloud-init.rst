##########
Cloud-init
##########


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

