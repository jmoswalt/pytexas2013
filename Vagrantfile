# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    config.vm.box = "precise64"
    config.vm.network :forwarded_port, guest: 80, host: 8060  # nginx
    config.vm.hostname = "pytexas2013"

    config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", 512]
    end

end