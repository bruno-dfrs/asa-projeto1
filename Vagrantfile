# -*- mode: ruby -*-
# vi: set ft=ruby :

NOME1 = "bruno"
NOME2 = "icaro"
XX = "28"

Vagrant.configure("2") do |config|
  # Configurações genéricas
  config.vm.box = "debian/bookworm64"
  config.ssh.insert_key = false

  config.trigger.before :"Vagrant::Action::Builtin::WaitForCommunicator", type: :action do |t|
    t.warn = "Interrompendo o servidor DHCP do VirtualBox"
    t.run = {inline: "vboxmanage dhcpserver stop --interface vboxnet0"}
  end

  config.vm.provider :virtualbox do |vb|
    vb.gui = false
    vb.memory = "512"
    vb.linked_clone = true
    vb.check_guest_additions = false
  end

  # Servidor de arquivos
  config.vm.define "arq" do |arq|
    (1..3).each do |i|
      arq.vm.disk :disk, size: "10GB", name: "disk-#{i}"
    end

    arq.vm.network :private_network, ip: "192.168.56.#{XX}" 
    arq.vm.hostname = "arq.#{NOME1}.#{NOME2}.devops"
  end

  # Servidor de banco de dados
  config.vm.define "db" do |db|
    db.vm.network :private_network, mac: "080027ABCDEF", type: :dhcp
    db.vm.hostname = "db.#{NOME1}.#{NOME2}.devops"
  end

  # Serivor de aplicação
  config.vm.define "app" do |app|
    app.vm.network :private_network, mac: "080027FEDCBA", type: :dhcp
    app.vm.hostname = "app.#{NOME1}.#{NOME2}.devops"
  end

  # Host Cliente
  config.vm.define "cli" do |cli|
    cli.vm.provider :virtualbox do |vb|
      vb.gui = true
      vb.memory = "1024"
    end

    cli.vm.network :private_network, type: :dhcp
    cli.vm.hostname = "cli.#{NOME1}.#{NOME2}.devops"
  end
end
