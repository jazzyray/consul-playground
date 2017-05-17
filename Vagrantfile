# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "AntonioMeireles/coreos-stable"
  config.vm.network "private_network", type: "dhcp"

  number_of_instances = 3
  (1..number_of_instances).each do |instance_number|
      config.vm.define "host-#{instance_number}" do |host|
        host.vm.hostname = "host-#{instance_number}"
      end
  end
end
