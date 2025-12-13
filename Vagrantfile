# -*- mode: ruby -*-
# vi: set ft=ruby :

# Define constant variables for team member names to use throughout the configuration
NAME1 = "bruno"
NAME2 = "icaro"

# Configure Vagrant version 2
Vagrant.configure("2") do |config|
  # ──────────────────────────────────────────────────────────────────
  # Generic configurations (arq, db, app, cli)
  # ──────────────────────────────────────────────────────────────────

  # Base box image used for all VMs (Debian 12 "Bookworm" 64-bit)
  config.vm.box = "debian/bookworm64"
  # Prevent Vagrant from generating new SSH keys (uses existing ones)
  config.ssh.insert_key = false

  # Configuration for vagrant-vbguest plugin (adds VirtualBox Guest Additions support)
  if Vagrant.has_plugin?("vagrant-vbguest")
    # Disable automatic guest additions updates
    config.vbguest.auto_update = false
  end

  # Trigger to stop VirtualBox's internal DHCP server before provisioning
  config.trigger.before :"Vagrant::Action::Builtin::WaitForCommunicator", type: :action do |t|
    # Warning message displayed when trigger runs
    t.warn = "Stopping VirtualBox DHCP server"
    # Run shell command to stop VirtualBox DHCP server on vboxnet0 interface
    t.run = {inline: "vboxmanage dhcpserver stop --interface vboxnet0"}
  end

  # Disable the default synced folder (shared folder between host and guest)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # VirtualBox-specific provider settings
  config.vm.provider :virtualbox do |vb|
    # Run VirtualBox in headless mode (no GUI)
    vb.gui = false
    # Allocate 512MB of RAM to the VM
    vb.memory = "512"
    # Use linked clones for faster VM creation (shares base disk image)
    vb.linked_clone = true
    # Disable checking for guest additions compatibility
    vb.check_guest_additions = false
  end

  # Apply generic Ansible playbook configuration to all machines
  config.vm.provision "ansible" do |ansible|
    # Use Ansible compatibility mode version 2.0
    ansible.compatibility_mode = "2.0"
    # Path to the playbook with common configuration for all VMs
    ansible.playbook = "playbooks/generic.yml"
    # Define Ansible inventory groups - all VMs belong to "all" group
    ansible.groups = {
      "all" => ["arq", "db", "app", "cli"]  # Gets IP addresses of all VMs
    }
  end

  # ──────────────────────────────────────────────────────────────────
  # File Server (arq)
  # ──────────────────────────────────────────────────────────────────

  # Define the "arq" virtual machine
  config.vm.define "arq" do |arq|
    # Create 3 additional virtual disks of 10GB each for the file server
    (1..3).each do |i|
      # Add disk with specified size and name
      arq.vm.disk :disk, size: "10GB", name: "disk-#{i}"
    end

    # Configure private network with static IP address
    arq.vm.network :private_network, ip: "192.168.56.128"
    # Set hostname using the defined name variables
    arq.vm.hostname = "arq.#{NAME1}.#{NAME2}.devops"

    # Apply file server specific Ansible playbook
    arq.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      # Playbook specific to file server configuration
      ansible.playbook = "playbooks/arq.yml"
    end
  end

  # ──────────────────────────────────────────────────────────────────
  # Database Server (db)
  # ──────────────────────────────────────────────────────────────────

  # Define the "db" virtual machine
  config.vm.define "db" do |db|
    # Configure private network with specific MAC address and DHCP for IP assignment
    db.vm.network :private_network, mac: "080027ABCDEF", type: :dhcp
    # Set hostname for database server
    db.vm.hostname = "db.#{NAME1}.#{NAME2}.devops"

    # Apply database server specific Ansible playbook
    db.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      # Playbook specific to database server configuration
      ansible.playbook = "playbooks/db.yml"
    end
  end

  # ──────────────────────────────────────────────────────────────────
  # Application Server (app)
  # ──────────────────────────────────────────────────────────────────

  # Define the "app" virtual machine
  config.vm.define "app" do |app|
    # Configure private network with specific MAC address and DHCP for IP assignment
    app.vm.network :private_network, mac: "080027FEDCBA", type: :dhcp
    # Set hostname for application server
    app.vm.hostname = "app.#{NAME1}.#{NAME2}.devops"

    # Apply application server specific Ansible playbook
    app.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      # Playbook specific to application server configuration
      ansible.playbook = "playbooks/app.yml"
    end
  end

  # ──────────────────────────────────────────────────────────────────
  # Client Host (cli)
  # ──────────────────────────────────────────────────────────────────

  # Define the "cli" virtual machine
  config.vm.define "cli" do |cli|
    # Override VirtualBox settings specifically for the client VM
    cli.vm.provider :virtualbox do |vb|
      # Allocate more RAM (1024MB) to the client VM
      vb.memory = "1024"
    end

    # Configure private network with DHCP for IP assignment
    cli.vm.network :private_network, type: :dhcp
    # Set hostname for client machine
    cli.vm.hostname = "cli.#{NAME1}.#{NAME2}.devops"

    # Apply client-specific Ansible playbook
    cli.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      # Playbook specific to client machine configuration
      ansible.playbook = "playbooks/cli.yml"
    end
  end
end
