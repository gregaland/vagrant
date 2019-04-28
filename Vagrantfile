#
# Vagrant Config
# 
# Default: /path/to/Vagrantfile/config.json
# Override: VAGRANT_CONFIG environment variable
#
require 'yaml'
require 'json'

full_config_path = nil
load_yaml = false
load_json = false
config_file = ENV['VAGRANT_CONFIG'].nil? ? 'config' : ENV['VAGRANT_CONFIG']

# If default, traverse to where the Vagrantfile is located and load the config
# from there.
if config_file == 'config'
  dir = '.'
  while File.expand_path(dir) != '/' do
    if File.exists?(File.join dir, 'Vagrantfile')
      full_config_path = File.join dir, config_file
      if File.exists?("#{full_config_path}.json") 
        full_config_path << '.json'
        load_json=true
      elsif File.exists?("#{full_config_path}.yml") 
        full_config_path << '.yml'
        load_yaml=true
      end
      break
    else
      dir = File.join dir, '..'
    end
  end
else
  # Using fully qualified (hopefully) VAGRANT_CONFIG env var
  if File.exists?(config_file)
    full_config_path = config_file
  end
end

if full_config_path.nil?
  puts "Failed to find the config file: #{config_file}"
  exit 1
end

if full_config_path.end_with?("json")
  config = JSON.parse(File.read(full_config_path))
elsif full_config_path.end_with?("yml")
  config = YAML.load(File.read(full_config_path))
end

sync_folders = config['sync_folders']
nodes_config = config['nodes']

Vagrant.configure("2") do |config|

  #
  # Iterate over the array of nodes
  nodes_config.each do |node|

    hostname = node[0] # name of node
    node_values = node[1] # conent of node
    name = node_values['name'].nil? ? hostname : node_values['name']
    environment = node_values['environment']

    #
    # Node Configuration
    #
    config.vm.define name do |config|

      config.ssh.insert_key = false if node_values['base']

      # Basic configurations start here
      
      # If box_url and box are not provided - skip the node
      if node_values['box_url'].nil? and node_values['box'].nil?
        puts "box_url|box is not specified for the node...skipping node #{name}"
        next
      end
     
      # box/box_url processing.
      config.vm.box = node_values['box'].nil? ? node_values['box_url'] : node_values['box']
      config.vm.box_url = node_values['box_url']
      if node_values['box_url'].nil?
        config.vm.box = node_values['box']
      elsif node_values['box_url'].match(/^http:.*$/)
        config.vm.box = node_values['box'].nil? ? File.basename(node_values['box_url']) : node_values['box']
        config.vm.box_url = node_values['box_url']
      end
      config.vm.box_version = node_values['box_version'] unless node_values['box_version'].nil?
        
      config.vm.hostname = hostname
      if node_values['ip'] == 'dhcp'
        config.vm.network 'private_network', type: "dhcp"
      else
        config.vm.network 'private_network', ip: node_values['ip']
      end

      #
      # Global (all nodes) Sync Folders
      #
      sync_folders.each do |sf|
        config_sync_folder(config, sf)
      end unless sync_folders.nil?

      #
      # Node Specific Sync Folders
      #
      node_values['sync_folders'].each do |sf|
        config_sync_folder(config, sf)
      end unless node_values['sync_folders'].nil?

      #
      # Memory, cpu, etc
      #
      config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", node_values['memory']] unless node_values['memory'].nil?
        vb.customize ["modifyvm", :id, "--cpus", node_values['cpus']] unless node_values['cpus'].nil?
        vb.customize ["modifyvm", :id, "--name", name]
      end

      #
      # Forwarded Ports
      #
      ports = node_values['ports']
      ports.each do |port|
        config.vm.network :forwarded_port, host: port['host'], guest: port['guest']
      end unless ports.nil?

      #---------------------------------------------------------------------------------
      # Provisioning - All lines below this line only get executed when the box is 
      # coming up for the first time or when vagrant is explicitly told to do 
      # provisioning, i.e. vagrant provision.  All provisioning occurs in order as 
      # defined below.
      #---------------------------------------------------------------------------------
      #
      #
      config.vm.provision "ansible" do |ansible|
        ansible.compatibility_mode = "2.0"
        ansible.playbook = "provisioning/playbook.yml"
      end
    end
  end
end

# 
# Load the config.json or whatever VAGRANT_CONFIG is pointing at
#
def get_config
  full_config_path = nil
  config_file = ENV['VAGRANT_CONFIG'].nil? ? 'config.json' : ENV['VAGRANT_CONFIG']

  dir = '.'
  while File.expand_path(dir) != '/' do
    if File.exists?(Vagrantfile)
      full_config_path = File.join dir, config_file
      break
    else
      dir = File.join dir, '..'
    end
  end

  if full_config_path.nil?
    raise StandardError("Failed to find the config file")
  end

  return JSON.parse(File.read(full_config_path))

end

#
# Setup a sync folder
#
def config_sync_folder(config, sf)
  host = sf['host']
  guest = sf['guest']
  owner = sf['owner'].nil? ? 'vagrant' : sf['owner']
  group = sf['group'].nil? ? 'users' : sf['group']
  #mo = sf['mount_options'].nil? ? nil : sf['mount_options']
  unless host.start_with?('/') and Dir.exists?(host)
    puts "Attempt to mount: #{host} failed.  Does Directory exist? Is path fully qualified?...skipping"
    return
  end
  unless guest.start_with?('/')
    puts "Guest mount: #{guest} must be fully qualified...skipping"
    retrun
  end 
  #config.vm.synced_folder host, guest, create: true, owner: owner, group: group, mount_options: mo
  config.vm.synced_folder host, guest, create: true, owner: owner, group: group
end
