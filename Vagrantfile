# -*- mode: ruby -*-
# vi: set ft=ruby :

ANSIBLE_PATH = '.' # path targeting Ansible directory (relative to Vagrantfile)

config_file = File.join(ANSIBLE_PATH, 'group_vars/development')

if File.exists?(config_file)
  wordpress_sites = YAML.load_file(config_file)['wordpress_sites']
else
  raise 'group_vars/development file not found. Please set `ANSIBLE_PATH` in Vagrantfile'
end

Vagrant.require_version '>= 1.5.1'

Vagrant.configure('2') do |config|
  config.vm.box = 'roots/bedrock'

  # Required for NFS to work, pick any local IP
  config.vm.network :private_network, ip: '192.168.50.5'
  config.vm.hostname = wordpress_sites.first['site_hosts'].first

  if Vagrant.has_plugin? 'vagrant-hostsupdater'
    config.hostsupdater.aliases = wordpress_sites.flat_map { |site| site['site_hosts'] }
  else
    puts 'vagrant-hostsupdater missing, please install the plugin:'
    puts 'vagrant plugin install vagrant-hostsupdater'
  end

  if Vagrant::Util::Platform.windows?
    wordpress_sites.each do |site|
      config.vm.synced_folder site['local_path'], remote_site_path(site), owner: 'vagrant', group: 'www-data', mount_options: ['dmode=776', 'fmode=775']
    end
  else
    if !Vagrant.has_plugin? 'vagrant-bindfs'
      raise Vagrant::Errors::VagrantError.new,
        "vagrant-bindfs missing, please install the plugin:\nvagrant plugin install vagrant-bindfs"
    else
      wordpress_sites.each do |site|
        config.vm.synced_folder site['local_path'], nfs_path(site), type: 'nfs'
        config.bindfs.bind_folder nfs_path(site), remote_site_path(site), u: 'vagrant', g: 'www-data'
      end
    end
  end

  config.vm.provision :ansible do |ansible|
    ansible.playbook = File.join(ANSIBLE_PATH, 'site.yml')
    ansible.groups = {
      'web' => ['default'],
      'development' => ['default']
    }
    ansible.extra_vars = {
      ansible_ssh_user: 'vagrant',
      user: 'vagrant'
    }
  end

  config.vm.provider 'virtualbox' do |vb|
    # Give VM access to all cpu cores on the host
    cpus = case RbConfig::CONFIG['host_os']
      when /darwin/ then `sysctl -n hw.ncpu`.to_i
      when /linux/ then `nproc`.to_i
      else 2
    end

    # Customize memory in MB
    vb.customize ['modifyvm', :id, '--memory', 1024]
    vb.customize ['modifyvm', :id, '--cpus', cpus]

    # Fix for slow external network connections
    vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    vb.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
  end
end

def nfs_path(site)
  "/vagrant-nfs-#{site['site_name']}"
end

def remote_site_path(site)
  File.join('/srv/www/', site['site_name'], 'current')
end
