*******
Vagrant
*******

Vagrant is an open-source tool for building and managing virtual machines
in a single workflow. It provides a simple and easy-to-use command-line interface
to create and configure lightweight, reproducible, and portable development
environments. Vagrant works with various virtualization providers, including
VirtualBox, VMware, Hyper-V, and Docker. It is widely used by developers
to create consistent development environments across different machines and
teams.

.. contents::
   :local:
   :depth: 2


Getting Started
===============

Follow the official documentation for the most up-to-date instructions:

- VirtualBox: `https://www.virtualbox.org/wiki/Linux_Downloads <https://www.virtualbox.org/wiki/Linux_Downloads>`_
- Vagrant: `https://developer.hashicorp.com/vagrant/downloads <https://developer.hashicorp.com/vagrant/downloads>`_


Verify the installation:

   .. code-block:: bash

      vboxmanage --version
      vagrant --version

Initialize a Vagrant Project
----------------------------

Create a new directory for your Vagrant project:

   .. code-block:: bash

      mkdir vagrant-demo
      cd vagrant-demo


Initialize a Vagrantfile with Ubuntu 24.04 (Noble Numbat):

   .. code-block:: bash

      vagrant init cloud-image/ubuntu-24.04

This creates a ``Vagrantfile`` in your directory.


Configure the Vagrantfile
-------------------------

Open the ``Vagrantfile`` and update it as follows:

.. code-block:: ruby
    :linenos:

    Vagrant.configure("2") do |config|
        config.vm.box = "cloud-image/ubuntu-24.04"
        config.vm.hostname = "demo-vm"

        config.vm.provider "virtualbox" do |vb|
        vb.name = "DemoVM"
        vb.memory = "4096"
        vb.cpus = 2
        end
    end

Working with Vagrant
--------------------

The basic Vagrant commands to manage your VM are:

1. Start the VM:

   .. code-block:: bash

      vagrant up


2. SSH into the VM:

   .. code-block:: bash

      vagrant ssh


3. Halt the VM:

   .. code-block:: bash

      vagrant halt


4. Destroy the VM:

   .. code-block:: bash

      vagrant destroy


Other Common Commands
^^^^^^^^^^^^^^^^^^^^^

- ``vagrant status``: Check VM status
- ``vagrant reload``: Restart VM with updated configuration
- ``vagrant box list``: List installed boxes


Provisioning with Vagrant
=========================

Available Provisioners
----------------------

Provisioners in Vagrant are tools that allow you to automatically configure
your virtual machines after they are created. They enable you to install
software, update packages, and set up services without manual intervention.
Vagrant supports multiple types of provisioners, including:

- **Shell**: Executes shell scripts or inline commands.
- **Docker**: Uses Docker containers for provisioning.
- **Ansible**: Uses Ansible playbooks for configuration management.
- **Chef**: Integrates with Chef for advanced provisioning.
- **Puppet**: Applies Puppet manifests for system configuration.

For a complete list of supported provisioners, refer to the official documentation 
`https://developer.hashicorp.com/vagrant/docs/provisioning <https://developer.hashicorp.com/vagrant/docs/provisioning>`_.

Provisioners are defined in the ``Vagrantfile`` using the
``config.vm.provision`` directive. For example:

Inline Provisioning
^^^^^^^^^^^^^^^^^^^
You can use inline shell scripts to provision your VM. For example: 

.. code-block:: ruby
    :linenos:

    config.vm.provision "shell", inline: <<-SHELL
        apt update
        apt install -y nginx
    SHELL


Script Provisioning
^^^^^^^^^^^^^^^^^^^

You can also use external shell scripts for provisioning. For example:

.. code-block:: ruby

   config.vm.provision "shell", path: "setup.sh"

Where ``setup.sh`` is a shell script in the same directory as your
``Vagrantfile``.

Provisioning Execution
^^^^^^^^^^^^^^^^^^^^^^

Provisioning can be triggered during ``vagrant up`` or later using
``vagrant provision``. This feature is essential for creating reproducible
and automated development environments.


Example: Install Nginx automatically:

.. code-block:: ruby
    :linenos:

    Vagrant.configure("2") do |config|
        config.vm.box = "cloud-image/ubuntu-24.04"

        config.vm.provision "shell", inline: <<-SHELL
        apt update
        apt install -y nginx
        SHELL
    end

Apply provisioning:

.. code-block:: bash
    :linenos:

    # to create and provision the VM
    vagrant up 
    
    # to force provisioning at startup for an already created VM 
    vagrant up --provision 

    # to provision an already running VM
    vagrant provision


Combining Provisioners
----------------------

Vagrant allows you to use multiple provisioners in a single ``Vagrantfile``.
This is useful when you want to mix simple shell commands with more advanced
configuration management tools. Provisioners run in the order they are defined.

Example:

.. code-block:: ruby
    :linenos:

    Vagrant.configure("2") do |config|
        config.vm.box = "cloud-image/ubuntu-24.04"

        # First, run a shell script to update packages
        config.vm.provision "shell", inline: <<-SHELL
        apt update
        SHELL

        # Then, use an Ansible playbook
        config.vm.provision "ansible" do |ansible|
        ansible.playbook = "playbook.yml"
        end
    end

You can also specify when a provisioner should run using the ``run`` option:

.. code-block:: ruby
    :linenos:

    config.vm.provision "shell", inline: "echo 'Hello!'", run: "always"
    config.vm.provision "shell", inline: "echo 'This runs only once'", run: "once"





Multi-Machine Environments
==========================

This section provides examples for defining and controlling **multi-machine Vagrant environments**
and enabling **machine-to-machine communication** using **private networks**
(Host‑only networking with VirtualBox). All examples assume Ubuntu 24.04
hosts and guests with the ``virtualbox`` provider.


Overview
--------

A **multi-machine** Vagrant environment is a single project (i.e., one ``Vagrantfile``)
that defines multiple VMs, often representing several roles (like **web**, **app**, and
**db**). Each machine can have its own CPU/memory, network interfaces, synced
folders, and **machine-specific provisioners**. A **private network** (i.e., a host‑only
adapter in VirtualBox) lets VMs reach each other **directly by IP** without
exposing services publicly.


Multi‑Machine Vagrantfile
-------------------------

The following minimal example defines **two machines** (``web`` and ``db``) on a
shared private network. Replace the IPs with any free addresses in your
VirtualBox host‑only network (commonly ``192.168.56.0/24``).

.. code-block:: ruby
    :linenos:

    # Vagrantfile
    Vagrant.configure("2") do |config|
        # Base box (Ubuntu 24.04: "noble")
        config.vm.box = "cloud-image/ubuntu-24.04"

        # --- DB machine ---
        config.vm.define "db" do |db|
        db.vm.hostname = "db.local"
        db.vm.network "private_network", ip: "192.168.56.11"

        db.vm.provider "virtualbox" do |vb|
            vb.name = "demo-db"
            vb.cpus = 1
            vb.memory = 1024
        end

        # Machine-specific provisioning (optional)
        db.vm.provision "shell", inline: <<-SHELL
            sudo apt update
            sudo apt install -y postgresql
        SHELL
        end

        # --- WEB machine ---
        config.vm.define "web" do |web|
        web.vm.hostname = "web.local"
        web.vm.network "private_network", ip: "192.168.56.10"

        web.vm.provider "virtualbox" do |vb|
            vb.name = "demo-web"
            vb.cpus = 2
            vb.memory = 2048
        end

        web.vm.provision "shell", inline: <<-SHELL
            sudo apt update
            sudo apt install -y nginx
        SHELL
        end
    end

Control the Machines
--------------------

You can start and manage both machines with:

.. code-block:: bash

   vagrant up               # starts all machines in the Vagrantfile
   vagrant status           # shows each machine status
   vagrant global-status    # lists all Vagrant machines on the host

You can SSH into each machine individually with:

.. code-block:: bash

   vagrant ssh web
   vagrant ssh db

Bring up **several** in sequence:

.. code-block:: bash

    vagrant up db web


Start sequentially (avoid parallel startup):

.. code-block:: bash

    vagrant up --no-parallel
    # or
    vagrant up db && vagrant up web

Use **Vagrant triggers** to block until a dependency is reachable:

.. code-block:: ruby
    :linenos:

    config.trigger.after :up do |t|
    t.only_on = ["web"]
    t.run = {
        inline: "until nc -z 192.168.56.11 5432; do echo 'Waiting for DB...'; sleep 2; done"
    }
    end





Patterns & Tips
---------------

Per machine ``config.vm.define`` blocks 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- Each machine gets a **logical name** and its own block for settings.

.. code-block:: ruby
    :linenos:

    config.vm.define "app" do |app|
        app.vm.box = "ubuntu/noble64"
        app.vm.hostname = "app.local"
    end

Separate role blocks keep configs clear
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Group **provider**, **network**, **provisioners**, and **synced folders**
under each machine definition to avoid cross‑contamination.

Globals and Per Machine Override 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Keep global settings (like base box) at the top level, and override
per machine as needed.

- Global default:

  .. code-block:: ruby

     config.vm.box = "cloud-image/ubuntu-24.04"

- Per machine override (e.g., a different base image for db):

  .. code-block:: ruby

     db.vm.box = "bento/ubuntu-24.04"


CPU/Memory Per Machine
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: ruby

   web.vm.provider "virtualbox" do |vb|
     vb.cpus = 2
     vb.memory = 2048
   end

   db.vm.provider "virtualbox" do |vb|
     vb.cpus = 1
     vb.memory = 1024
   end




Private Networks for Inter‑VM Communication
===========================================

Private networks create a **host-only** LAN that is **not** reachable from outside your 
host. All VMs on the same private network can **talk to each other using IP addresses**.

Static IPs (most common)
------------------------

- Assign **unique** IPs in the same subnet:

  .. code-block:: ruby

     web.vm.network "private_network", ip: "192.168.56.10"
     db.vm.network  "private_network", ip: "192.168.56.11"


- Ensure the subnet (e.g., ``192.168.56.0/24``) exists in VirtualBox
  Host‑Only Networks. If unsure, bring machines up; Vagrant/VirtualBox will
  create a host‑only adapter as needed.


DHCP (alternative)
------------------

- Let the host-only DHCP server assign addresses:

  .. code-block:: ruby

     web.vm.network "private_network", type: "dhcp"
     db.vm.network  "private_network", type: "dhcp"


Multiple Private Networks
-------------------------

- You can attach multiple host‑only networks (e.g., a **backend** and a **monitoring** LAN):

  .. code-block:: ruby

     app.vm.network "private_network", ip: "192.168.56.20"  # backend
     app.vm.network "private_network", ip: "192.168.57.20"  # monitoring

- Machines can share one or more of these to control communication patterns.


Name Resolution Between Machines
--------------------------------

By default, VMs **do not** resolve each other by hostname. Use one of:

IP Addresses
^^^^^^^^^^^^

.. code-block:: bash

   # From web -> db
   ping -c1 192.168.56.11
   psql -h 192.168.56.11 -U postgres


``/etc/hosts`` Entries via Provisioning
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Add hosts entries on **each** VM so names resolve locally:

.. code-block:: ruby
    :linenos:

    hosts = <<-HOSTS
    192.168.56.10 web.local web
    192.168.56.11 db.local db
    HOSTS

    ["web", "db"].each do |m|
        config.vm.define m do |node|
        node.vm.provision "shell", inline: <<-SHELL
            cat <<'EOF' | sudo tee -a /etc/hosts
            #{hosts}
            EOF
        SHELL
        end
    end

Hostmanager Plugin
^^^^^^^^^^^^^^^^^^

The community **vagrant-hostmanager** plugin can manage host entries across the
host and guests. If you choose this route:

.. code-block:: bash

   vagrant plugin install vagrant-hostmanager

Then add:

.. code-block:: ruby
    :linenos:

    Vagrant.configure("2") do |config|
        config.hostmanager.enabled = true
        config.hostmanager.manage_host = true
        config.hostmanager.manage_guest = true
        config.hostmanager.ignore_private_ip = false
        config.hostmanager.include_offline = true
        config.vm.define 'example-box' do |node|
            node.vm.hostname = 'example-box-hostname'
            node.vm.network :private_network, ip: '192.168.42.42'
            node.hostmanager.aliases = %w(example-box.localdomain example-box-alias)
        end
    end


Syncing Code/Data across Machines
=================================

A shared project folder can be mounted on multiple VMs for consistent builds. For example, the folder "./src" on the host is mounted to "/srv/src" on both VMs:

.. code-block:: ruby
    :linenos:

    web.vm.synced_folder "./src", "/srv/src", create: true
    app.vm.synced_folder "./src", "/srv/src", create: true

More details on synced folder options, see the official docs.


Troubleshooting
===============

- **IP conflicts**: Ensure each VM gets a **unique** IP; verify the host‑only
  network range in VirtualBox. Adjust to another subnet (e.g., ``192.168.57.0/24``)
  if your host uses ``192.168.56.0/24`` elsewhere.

- **Service not reachable**: Confirm the service binds to the **private IP** or
  to **0.0.0.0** (not only ``127.0.0.1``). Restart the service and check its
  port with ``ss -tulpn | grep LISTEN``.

- **Provisioning order**: Use ``vagrant up --no-parallel`` or ``vagrant up db && vagrant up web``.
  Add **triggers** to wait until dependent ports are open.

- **UFW/Firewall**: If enabled, allow the private subnet:
  ``sudo ufw allow from 192.168.56.0/24``. For ICMP, also allow on the interface:
  ``sudo ufw allow in on enp0s8``.

- **Name resolution**: If hostnames don’t resolve, either use IPs or ensure
  ``/etc/hosts`` entries are provisioned on **every** VM, or use the
  **vagrant-hostmanager** plugin.



