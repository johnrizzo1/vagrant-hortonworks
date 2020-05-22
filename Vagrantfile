# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'getoptlong'

ansible_tags = ''
ansible_skip_tags = 'sensors'

#
# Process Ansible Arguments
begin
  opts = GetoptLong.new(
    [ '--ansible-tags', GetoptLong::OPTIONAL_ARGUMENT ],
    [ '--ansible-skip-tags', GetoptLong::OPTIONAL_ARGUMENT ]
  )

  opts.quiet = true

  opts.each do |opt, arg|
    case opt
      when '--ansible-tags'
        ansible_tags=arg
      when '--ansible-skip-tags'
        ansible_skip_tags=arg
    end
  end
  rescue Exception => ignored
  #Ignore to allow other opts to be passed to Vagrant
end

puts "Running with ansible-tags: #{ansible_tags.split(',').to_s}" if ansible_tags != ''
puts "Running with ansible-skip-tags: #{ansible_skip_tags.split(',').to_s}" if ansible_skip_tags != ''

#
# Host Configuration
hosts = [
  {
    hostname: 'master01.example.com',
    ip: '192.168.0.2',
    # memory: '16384',
    memory: '24576',
    cpus: 4,
    promisc: 2,  # enables promisc on the 'Nth' network interface
    disksize: 100
  # }, {
  #   hostname: 'slave01',
  #   ip: '192.168.0.3',
  #   memory: '16384',
  #   cpus: 4,
  #   promisc: 2,  # enables promisc on the 'Nth' network interface
  #   disksize: 100
  }
]

#
# Vagrant Configuration
Vagrant.configure('2') do |config|
  config.vm.box = 'centos/7'
  # config.vm.box = 'ubuntu/bionic64' # 18.04 LTS
  # config.vm.box = 'ubuntu/xenial64' # 16.04 LTS
  config.vm.box_check_update = true
  # config.ssh.insert_key = true
  config.ssh.insert_key = false

  # enable the hostmanager plugin
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true

  # enable vagrant cachier if present
  # if Vagrant.has_plugin?('vagrant-cachier')
    # config.cache.enable :apt
    # config.cache.enable :yum, :gem
    # config.cache.scope = :box
    # config.cache.scope = :machine

    # config.cache.synced_folder_opts = {
      # type: :nfs,
      # mount_options: ['rw', 'vers=3', 'tcp', 'nolock']
    # }
  # end

  # host definition
  hosts.each_with_index do |host, index|
    config.vm.define host[:hostname] do |node|
      # host settings
      node.vm.hostname = host[:hostname]
      node.vm.network :private_network, ip: host[:ip]
      # The disk will be resized to the following value by the
      # vagrant-disksize plugin
      node.disksize.size = "#{host[:disksize]}GB"

      # vm settings
      node.vm.provider 'virtualbox' do |vb|
        vb.memory = host[:memory]
        vb.cpus   = host[:cpus]

        # disable audio, so that the vm doesn't capture the sound / mic
        vb.customize ['modifyvm', :id, '--audio', 'none']
        # vb.customize ['modifyvm', :id, '--cpuexecutioncap', '50']
        vb.linked_clone = true if Gem::Version.new(Vagrant::VERSION) >= Gem::Version.new('1.8.0')
      end

      # node.vm.provision :shell, inline: <<-SCRIPT
# apt-get update -y && apt-get upgrade -y && apt-get install -y libsnappy1v5
# SCRIPT

      if index == hosts.length - 1
        # provision the host with ansible
        node.vm.provision :ansible do |ansible|
          ansible.config_file        = 'ansible/ansible.cfg'
          ansible.playbook           = 'ansible/playbooks/install_cluster.yml'
          ansible.limit              = 'all,localhost'

          ansible.become             = true
          ansible.tags               = ansible_tags.split(',') if ansible_tags != ''
          ansible.skip_tags          = ansible_skip_tags.split(',') if ansible_skip_tags != ''
          ansible.compatibility_mode = '2.0'
          # ansible.verbose            = '-vvvv'
          ansible.raw_arguments      = Shellwords.shellsplit(ENV['ANSIBLE_ARGS']) if ENV['ANSIBLE_ARGS']

          # ansible.inventory_path     = 'ansible/inventory/static'
          ansible.groups = {
            'hdp-master' => ['master01.example.com'],
            # 'hdp-slave' => ['slave01.example.com'],
            # 'hadoop-cluster' => ['master01.example.com', 'slave01.example.com'],
            'hdp-singlenode' => ['master01.example.com'],
            # 'mytestcluster:children' => ['hdp-master', 'hdp-slave'],
            'mytestcluster:children' => ['hdp-singlenode'],
            'all:vars' => {
              # 'ansible_user' => 'vagrant',
              # 'ansible_ssh_private_key_file' => '~/.vagrant.d/insecure_private_key',
              # 'ansible_python_interpreter' => '/usr/bin/python3',
              # 'ansible_python_interpreter' => '/usr/bin/python',
              'rack' => '/default-rack'
            }
          }

          ansible.extra_vars = {
            cloud_name: 'static',
            cloud_to_use: 'static',
            inventory_to_use: 'static'
          }
        end
      end
    end
  end
end
