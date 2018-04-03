# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'tempfile'

Vagrant.require_version ">= 1.6.0"

$update_channel = "stable"
$etcd_count = 1
$etcd_vm_memory = 512
$etcd_version = 2

ETCD_CLOUD_CONFIG_PATH = File.expand_path("etcd%d-cloud-config.yaml" % $etcd_version)

def etcdIP(num)
  return "172.17.2.#{num+100}"
end

etcdIPs = [*1..$etcd_count].map{ |i| etcdIP(i) }
initial_etcd_cluster = etcdIPs.map.with_index{ |ip, i| "etcd#{i+1}.coreos.local=https://#{ip}:2380" }.join(",")
etcd_endpoints = etcdIPs.map{ |ip| "https://#{ip}:2379" }.join(",")

puts "etcd version %d. IPs: %s" % [$etcd_version, etcdIPs.join(", ")]

def createCert(ip, name)
  if not File.directory?("ssl")
    FileUtils.mkdir("ssl")
  end

  if not File.exists?('ssl/ca-key.pem')
    system("cd ssl && echo '{\"CN\":\"CA\",\"key\":{\"algo\":\"rsa\",\"size\":2048}}' | cfssl gencert -initca - | cfssljson -bare ca -") or abort ("failed generating SSL CA")
  end

  if not File.exists?('ssl/ca-config.json')
    system("echo '{\"signing\":{\"default\":{\"expiry\":\"43800h\",\"usages\":[\"signing\",\"key encipherment\",\"server auth\",\"client auth\"]}}}' > ssl/ca-config.json") or abort ("failed create CA config")
  end

  if not File.exists?("ssl/#{ip}.tgz")
    crt_data = '{"CN":"%{name}","hosts":[""],"key":{"algo":"rsa","size":2048}}'
    address = "#{ip},#{name}"
    name = "server"

    system("echo '#{crt_data}' | cfssl gencert -config=ssl/ca-config.json -ca=ssl/ca.pem -ca-key=ssl/ca-key.pem -hostname=\"%{address}\" - | cfssljson -bare ssl/#{ip}-%{name}" % { :name => name, :address => address }) or abort ("failed create server crt")

    address = ""
    name = "client"

    system("echo '#{crt_data}' | cfssl gencert -config=ssl/ca-config.json -ca=ssl/ca.pem -ca-key=ssl/ca-key.pem -hostname=\"%{address}\" - | cfssljson -bare ssl/#{ip}-%{name}" % { :name => name, :address => address }) or abort ("failed create client crt")

    system("cd ssl && tar czf #{ip}.tgz #{ip}-server.pem #{ip}-server-key.pem ca.pem")
  end
end
# Generate root CA
# system("mkdir -p ssl && lib/init-ssl-ca ssl") or abort ("failed generating SSL artifacts")

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.ssh.insert_key = false

  config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= 1151.0.0"
  config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  config.vm.provider :virtualbox do |vb|
    vb.cpus = 1
    vb.gui = false
  end

  (1..$etcd_count).each do |i|
    config.vm.define vm_name = "etcd%d.coreos.local" % i do |etcd|

      IP = etcdIP(i)

      data = File.read(ETCD_CLOUD_CONFIG_PATH)
      data = data.gsub("{{ETCD_NODE_NAME}}", vm_name)
      data = data.gsub("{{ETCD_INITIAL_CLUSTER}}", initial_etcd_cluster)
      data = data.gsub("{{IP}}", IP)
      etcd_config_file = Tempfile.new('etcd_config')
      etcd_config_file.write(data)
      etcd_config_file.close

      etcd.vm.hostname = vm_name

      etcd.vm.provider :virtualbox do |vb|
        vb.memory = $etcd_vm_memory
      end

      createCert(IP, vm_name)
#      etcd.vm.network "forwarded_port", guest: 2379, host: i+12378, id: vm_name

      etcd.vm.network :private_network, ip: IP

      etcd.vm.provision :file, :source => "ssl/#{IP}.tgz", :destination => "/tmp/etcd-ssl.tgz"
      etcd.vm.provision :shell, :inline => "mkdir -p /etc/etcd/tls && tar -C /etc/etcd/tls/ -xf /tmp/etcd-ssl.tgz", :privileged => true

      etcd.vm.provision :shell, :inline => "echo \"%s %s\" >> /etc/hosts" % [etcdIP(i), vm_name], :privileged => true
      etcd.vm.provision :shell, :inline => "chown etcd:etcd /etc/etcd/tls/*", :privileged => true

      etcd.vm.provision :file, :source => etcd_config_file.path, :destination => "/tmp/vagrantfile-user-data"
      etcd.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
    end
  end

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end
