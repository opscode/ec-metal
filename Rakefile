require 'json'
require 'fileutils'
require './cookbooks/ec-common/libraries/topo_helper.rb'
Dir["lib/tasks/*.rake"].each { |t| load t }

task :default => [:up]

# Environment variables to be consumed by ec-harness and friends
harness_dir = ENV['HARNESS_DIR'] ||= File.dirname(__FILE__)
repo_dir = ENV['REPO_PATH'] ||= File.join(harness_dir, 'chef-repo')

# just in cases user has a different default Vagrant provider
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'

def get_config
  JSON.parse(File.read(ENV['ECM_CONFIG'] || 'config.json'))
end

desc 'Install required Gems into the vendor/bundle directory'
task :bundle do
  sh('bundle install --path vendor/bundle --binstubs')
end

desc 'Bring the VMs online and install+configure Enterprise Chef HA'
task :up => [:print_enviornment, :keygen, :cachedir, :config_copy, :bundle, :berks_install] do
  create_users_directory
  sh("#{harness_dir}/bin/chef-client -z -o ec-harness::private_chef_ha")
end
task :start => :up

desc 'Bring the VMs online and then UPGRADE TORTURE'
task :upgrade_torture => [:keygen, :cachedir, :config_copy, :bundle, :berks_install] do
  create_users_directory
  sh("#{harness_dir}/bin/chef-client -z -o ec-harness::upgrade_torture")
end

desc 'Simple upgrade step, installs the package from default_package. Machines must be running'
task :upgrade => [:print_enviornment, :keygen, :cachedir, :config_copy, :bundle, :berks_install] do
  create_users_directory
  sh("#{harness_dir}/bin/chef-client -z -o ec-harness::upgrade")
end

desc "Copies pivotal.pem from chef server and generates knife.rb in the repo dir"
task :pivotal => [:keygen, :cachedir, :config_copy, :bundle, :berks_install] do
  sh("#{harness_dir}/bin/chef-client -z -o ec-harness::pivotal")
end

desc 'Destroy all VMs'
task :destroy do
  sh("#{harness_dir}/bin/chef-client -z -o ec-harness::cleanup")
end
task :cleanup => :destroy

desc 'SSH to a machine like so: rake ssh[backend1]'
task :ssh, [:machine] do |t,arg|
  Dir.chdir(File.join(harness_dir, 'vagrant_vms')) {
    sh("vagrant ssh #{arg.machine}")
  }
end

desc "Print all ec-metal enviornment variables"
task :print_enviornment do
  puts "================== ec-metal ENV ==========================="
  ENV.each { |k,v| puts "#{k} = #{v}" if k.include?("ECM_") }
  puts "==========================================================="
end

# Vagrant standard but useful commands
%w(status halt suspend resume).each do |command|
  desc "Equivalent to running: vagrant #{command}"
  task :"#{command}" do
    Dir.chdir(File.join(harness_dir, 'vagrant_vms')) {
      sh("vagrant #{command}")
    }
  end
end

task :config_copy do
  unless ENV['ECM_CONFIG'] && File.exists?(ENV['ECM_CONFIG'])
    config_file = File.join(harness_dir, 'config.json')
    config_ex_file = File.join(harness_dir, 'examples', 'config.json.example')
    unless File.exists?(config_file)
      FileUtils.cp(config_ex_file, config_file)
    end
  end
end

task :keygen do
  keydir = File.join(repo_dir, 'keys')
  FileUtils.mkdir_p keydir

  if Dir["#{keydir}/*"].empty? && !ENV['ECM_KEYPAIR_PATH'].nil?
    FileUtils.copy("#{ENV['ECM_KEYPAIR_PATH']}/id_rsa", "#{keydir}/id_rsa")
    FileUtils.copy("#{keydir}/id_rsa", "#{keydir}/#{ENV['ECM_KEYPAIR_NAME']}") unless ENV['ECM_KEYPAIR_NAME'].nil?
    FileUtils.copy("#{ENV['ECM_KEYPAIR_PATH']}/id_rsa.pub", "#{keydir}/id_rsa.pub")
  end

  if Dir["#{keydir}/*"].empty?
    comment = ENV['ECM_KEYPAIR_NAME'].nil? ? "" : "-C #{ENV['ECM_KEYPAIR_NAME']}"
    command = "ssh-keygen #{comment} -P '' -q -f #{keydir}/id_rsa"
    puts "Keygen: #{command}"
    sh(command)
  end
end

desc 'Add hosts entries to /etc/hosts'
task :add_hosts do
  config = get_config
  config = fog_populate_ips(config) if config['provider'] == 'ec2'
  create_hosts_entries(config['layout'])
  print_final_message(config, repo_dir)
end

desc 'Remove hosts entries to /etc/hosts'
task :remove_hosts do
  config = get_config
  config = fog_populate_ips(config) if config['provider'] == 'ec2'
  remove_hosts_entries(config['layout'])
end

task :cachedir do
  if ENV['ECM_CACHE_PATH'] && Dir.exists?(ENV['ECM_CACHE_PATH'])
    cachedir = ENV['ECM_CACHE_PATH']
  else
    cachedir = File.join(harness_dir, 'cache')
    FileUtils.mkdir_p cachedir
  end
  puts "Using package cache directory #{cachedir}"
end

task :berks_install do
  cookbooks_path = File.join(repo_dir, 'vendor/cookbooks')
  sh("rm -r #{cookbooks_path}") if Dir.exists?(cookbooks_path)
  sh("#{harness_dir}/bin/berks vendor #{cookbooks_path}")
end

# Fix to work with topohelper
# desc "Runs remote commands via ssh.  Usage remote[servername, 'command args string']"
# # "knife-opc user create rockawesome patrick wright patrick@getchef.com password"
# # "knife-opc org create myorg2 supercoolorg -a rockawesome"
# task :remote, [:machine, :command] do |t, arg|
#   configip = fog_populate_ips(get_config)
#   %w(backends frontends standalones).each do |whichend|
#     configip['layout'][whichend].each do |node,attrs|
#       if node == arg[:machine]
#         case configip['provider']
#           when 'ec2'
#             ssh_username = configip['ec2_options']['ssh_username'] || 'ec2-user'
#           when 'vagrant'
#             ssh_username = 'vagrant'
#           else
#             ssh_username = 'root'
#         end
#         cmd = "ssh #{ssh_username}@#{attrs['ipaddress']} -o StrictHostKeyChecking=no -i #{File.join(harness_dir, 'keys')}/id_rsa \"#{arg[:command]}\""
#         puts "Executing '#{arg[:command]}' on #{arg[:machine]}"
#         sh(cmd)
#       end
#     end
#   end
# end

desc "Open csshx to the nodes of the server."
task :csshx do
  config = get_config
  config = fog_populate_ips(config) if config['provider'] == 'ec2'
  csshx(config, repo_dir)
end

desc "Execute a command on a remote machine"
task :execute, [:machine, :command] do |t,arg|
  sh %Q{ssh -o StrictHostKeyChecking=no -i #{File.join(harness_dir, 'keys')}/id_rsa \
        #{ssh_user()}@#{machine(arg.machine)['hostname']} #{arg.command} }
end

desc "Copy a file/directory from local to the machine indicated"
task :scp, [:machine, :source_path, :remote_path] do |t,arg|
  sh %Q{scp -r -o StrictHostKeyChecking=no -i #{File.join(harness_dir, 'keys')}/id_rsa \
        #{arg.source_path} #{ssh_user()}@#{machine(arg.machine)['hostname']}:#{arg.remote_path} }
end

def machine(machine_name)
  topo = TopoHelper.new(:ec_config => get_config['layout'])
  merged_topo = topo.merged_topology
  machine = merged_topo[machine_name]
  abort("Machine #{machine_name} not found") if machine.nil?
  return machine
end

def ssh_user()
  config = get_config()
  case config['provider']
    when 'ec2'
      config['ec2_options']['ssh_username'] || 'ec2-user'
    when 'vagrant'
      'vagrant'
    else
      'root'
  end
end

# task :ec2_to_file do
#   file = File.open('ec2_ips', 'w')
#   file.truncate(file.size)
#   configip = fog_populate_ips(get_config)
#   %w(backends frontends standalones).each do |whichend|
#     configip['layout'][whichend].each do |node,attrs|
#       file.write("#{node}=#{attrs['ipaddress']}\n")
#     end
#   end
# end

