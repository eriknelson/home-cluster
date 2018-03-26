# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure("2") do |config|
  pdir = File.dirname(__FILE__)
  config_file = File.join(pdir, "config.yml")
  config_present = File.file? config_file
  config_yaml = YAML.load(File.read(config_file))
  domain = config_yaml.fetch("cluster_domain", "example.net")

  #nodes = (1..(ENV["NUM_NODES"]||2).to_i).map { |i| "node#{i-1}.#{domain}"}
  nodes = [1].map { |i| "node#{i-1}.#{domain}"}
  #nodes = []
  verbosity = ENV["VERBOSITY"]||""

  config.hostmanager.enabled = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = false

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
    config.cache.synced_folder_opts = {
      type: 'rsync',
    }
  end

  config.vm.define "master.#{domain}", :primary => true do |master|
    master.vm.box = "centos/7"
    master.vm.hostname = "master.#{domain}"
    master.hostmanager.manage_host = true
    master.hostmanager.manage_guest = false
    master.vm.network :private_network,
      :ip => '192.168.13.2',
      :libvirt__network_name => "nsk-cluster",
      :libvirt__domain_name => domain,
      :libvirt__dhcp_enabled => true,
      :libvirt__dhcp_start=> "192.168.13.10",
      :libvirt__dhcp_stop=> "192.168.13.20",
      :libvirt__netmask => "255.255.255.0",
      :libvirt__dhcp_bootp_file => "undionly.kpxe",
      :libvirt__dhcp_bootp_server => "192.168.13.2"

    master.vm.synced_folder '.', '/vagrant', disabled: true

    master.vm.provision :file, :source => "~/.ssh/id_rsa.pub", :destination => "/tmp/key"
    master.vm.provision :shell, :inline => <<-EOF
      cat /tmp/key >> /home/vagrant/.ssh/authorized_keys
      cp -r /home/vagrant/.ssh /root
    EOF

    master.vm.provision :ansible do |ansible|
      ansible.verbose = verbosity
      ansible.playbook = 'playbooks/starter.yml'
      ansible.become = true
      ansible.limit = 'all,localhost'
      ansible.inventory_path = './inventory'
      ansible.extra_vars = 'config.yml' if File.file? 'config.yml'
    end

    if Vagrant.has_plugin?("vagrant-triggers") and nodes.length > 0
      config.trigger.after [:provision, :up] do
        nodes.each do |node|
          run "vagrant up #{node}"
          sleep 10
        end
        #extra_vars = {
          #:number_of_hosts => nodes.length,
          #:prompt_for_hosts => false,
          #:modify_etc_hosts => true,
          #:glusterfs_wipe => true
        #}
        ## TODO: Need a dynamic inventory script for this
        #run "ansible-playbook playbooks/nodes.yml -e '#{extra_vars.to_json}' #{ File.file?('config.yml') ? '-e @config.yml' : ''} -i inventory -l 'all,localhost' #{verbosity == '' ? '' : '-' + verbosity}"
      end
    end

  end


  nodes.each do |name|
    config.vm.define name, :autostart => false do |node|
      node.hostmanager.manage_guest = true
      node.hostmanager.manage_host = false
      node.vm.hostname = name

      node.vm.provider :libvirt do |domain|
        domain.storage :file, :size => '40G'
        domain.storage :file, :size => '20G'
        domain.mgmt_attach = 'false'
        domain.management_network_name = 'nsk-cluster'
        domain.management_network_address = "192.168.13.0/24"
        domain.management_network_mode = "nat"
        domain.boot 'hd'
        domain.boot 'network'
      end
    end
  end


  config.vm.provider :libvirt do |libvirt|
    libvirt.driver = "kvm"
    libvirt.memory = 4000
    libvirt.cpus = `grep -c ^processor /proc/cpuinfo`.to_i
  end
end
