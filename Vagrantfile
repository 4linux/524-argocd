# -*- mode: ruby -*-
# vi: set ft=ruby :

BOX = "bento/ubuntu-24.04"

vms = {
  "ci-tools" => { "memory" => 3072, "cpus" => 2, "ip" => "192.168.88.10", "playbook" => "provision/ci-tools.yaml" },
  "nexus"    => { "memory" => 2560, "cpus" => 1, "ip" => "192.168.88.20", "playbook" => "provision/nexus.yaml" },
  "k3s"      => { "memory" => 7168, "cpus" => 2, "ip" => "192.168.88.30", "playbook" => "provision/k3s.yaml" },
}

Vagrant.configure("2") do |config|
  config.vm.box = BOX

  # A box ja traz o Guest Additions. Evita que o plugin vagrant-vbguest,
  # quando instalado na maquina fisica, tente atualiza-lo.
  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")

  vms.each do |name, conf|
    config.vm.define name do |vm|
      vm.vm.hostname = name
      vm.vm.network "private_network", ip: conf["ip"]

      vm.vm.provider "virtualbox" do |vb|
        vb.name   = "524-argocd-#{name}"
        vb.memory = conf["memory"]
        vb.cpus   = conf["cpus"]
      end

      # Ansible do repositorio do Ubuntu, ja com a colecao community.docker
      vm.vm.provision "shell", name: "ansible",
        inline: "command -v ansible-playbook >/dev/null || (apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -y -qq ansible >/dev/null)"

      vm.vm.provision "ansible_local" do |ansible|
        ansible.playbook           = conf["playbook"]
        ansible.install            = false
        ansible.compatibility_mode = "2.0"
      end
    end
  end
end
