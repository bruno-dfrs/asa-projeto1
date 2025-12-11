# -*- mode: ruby -*-
# vi: set ft=ruby :

NOME1 = "bruno"
NOME2 = "icaro"
XX = "28"

Vagrant.configure("2") do |config|
  # ──────────────────────────────────────────────────────────────────
  # Configurações genéricas (arq, db, app e cli)
  # ──────────────────────────────────────────────────────────────────

  config.vm.box = "debian/bookworm64" # Box base utilizada para todas as VMs
  config.ssh.insert_key = false # Impede que o Vagrant gere novas chaves SSH

  # Configuração do plugin vagrant-vbguest (adiciona suporte a guest additions)
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  # Trigger para parar o servidor DHCP interno do VirtualBox antes de provisionar
  config.trigger.before :"Vagrant::Action::Builtin::WaitForCommunicator", type: :action do |t|
    t.warn = "Interrompendo o servidor DHCP do VirtualBox"
    t.run = {inline: "vboxmanage dhcpserver stop --interface vboxnet0"}
  end

  # Desativa a pasta sincronizada padrão
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Configurações do VirtualBox
  config.vm.provider :virtualbox do |vb|
    vb.gui = false
    vb.memory = "512"
    vb.linked_clone = true
    vb.check_guest_additions = false
  end

  # Aplicando configurações do playbook genérica em todas as máquinas
  config.vm.provision "ansible" do |ansible|
    ansible.compatibility_mode = "2.0"
    ansible.playbook = "playbooks/generic.yml" # Playbook de configuração comum
    ansible.groups = {
      "all" => ["arq", "db", "app", "cli"] # Pega o IP de todas as VMs
    }
  end

  # ──────────────────────────────────────────────────────────────────
  # Servidor de Arquivos (arq)
  # ──────────────────────────────────────────────────────────────────

  config.vm.define "arq" do |arq|
    (1..3).each do |i|
      arq.vm.disk :disk, size: "10GB", name: "disk-#{i}"
    end

    arq.vm.network :private_network, ip: "192.168.56.#{XX}" 
    arq.vm.hostname = "arq.#{NOME1}.#{NOME2}.devops"

    arq.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/arq.yml"
    end
  end

  # ──────────────────────────────────────────────────────────────────
  # Servidor de Banco de Dados (db)
  # ──────────────────────────────────────────────────────────────────

  config.vm.define "db" do |db|
    db.vm.network :private_network, mac: "080027ABCDEF", type: :dhcp
    db.vm.hostname = "db.#{NOME1}.#{NOME2}.devops"

    arq.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/db.yml"
    end
  end

  # ──────────────────────────────────────────────────────────────────
  # Servidor de Aplicação (app)
  # ──────────────────────────────────────────────────────────────────

  config.vm.define "app" do |app|
    app.vm.network :private_network, mac: "080027FEDCBA", type: :dhcp
    app.vm.hostname = "app.#{NOME1}.#{NOME2}.devops"

    arq.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/app.yml"
    end
  end

  # ──────────────────────────────────────────────────────────────────
  # Host Cliente (cli)
  # ──────────────────────────────────────────────────────────────────

  config.vm.define "cli" do |cli|
    cli.vm.provider :virtualbox do |vb|
      vb.gui = true
      vb.memory = "1024"
    end

    cli.vm.network :private_network, type: :dhcp
    cli.vm.hostname = "cli.#{NOME1}.#{NOME2}.devops"

    arq.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/cli.yml"
    end
  end
end
