# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "user-data")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$num_instances = 1
$instance_name_prefix = "core"
$update_channel = "alpha"
$image_version = "current"
$enable_serial_logging = false
$share_home = false
$vm_gui = false
$vm_memory = 1024
$vm_cpus = 1
$shared_folders = {}
$forwarded_ports = {}


# if the config.rb file exists, require it
if File.exist?(CONFIG)
  require CONFIG
end

Vagrant.configure("2") do |config|
    # use vagrant insecure key
    config.ssh.insert_key=false

    # sets the box that the machine will be brought up against
    # this value needs to match the name of a downloaded box or
    # the shorthand name of a box in Atlas
    config.vm.box = "coreos-%s" % $update_channel

    # set version of box to use
    if $image_version != "current"
        config.vm.box_version = $image_version
    end

    # This points to where the configured box can be found at
    # NOTE: This needs to be set to the correct url of where the mutated libvirt box is hosted
    #config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/%s/coreos_production_vagrant.json" % [$update_channel, $image_version]


    # Set configuration for libvirt provider
    config.vm.provider :libvirt do |domain|

        # Libvirt host connection details
        domain.host = ""
        domain.connect_via_ssh = true
        domain.username = "jharshman"
        domain.storage_pool_name = "default"
        
        # Console graphics settings
        domain.graphics_type = "vnc"
        domain.graphics_port = "-1"
        domain.graphics_ip = "0.0.0.0"

    end

    # setup settings for each VM started
    (1..$num_instances).each do |i|
        config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i] do |config|
            
            # NOTE: Work on enabling serial logging for libvirt
            if $enable_serial_logging
                logdir = File.join(File.dirname(__FILE__), "log")
                FileUtils.mkdir_p(logdir)

                serialFile = File.join(logdir, "%s-serial.txt" %vm_name)
                FileUtils.touch(serialFile)

                config.vm.provider :libvirt do |domain, override|
                    #NOTE: implement serial logging 
                end
            end

            # expose docker ports
            if $expose_docker_tcp
                config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
            end

            # forward ports
            $forwarded_ports.each do |guest, host|
                config.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
            end

            # configure memory for the instance
            config.vm.provider :libvirt do |domain|
                domain.gui = $vm_gui
                domain.memory = $vm_memory
                domain.cpus = $vm_cpus
            end

            # set network ip
            ip = "172.17.8.#{i+100}"
            config.vm.network :private_network, ip: ip

            
            # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
            #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']
            
            # setup synced folders between the host and guest machines
            $shared_folders.each_with_index do |(host_folder, guest_folder), index|
                config.vm.synced_folder host_folder.to_s, guest_folder.to_s, id: "core-share%02d" % index, nfs: true, mount_options: ['nolock,vers=3,udp']
            end

            # share out the home directory
            if $share_home
                config.vm.synced_folder ENV['HOME'], ENV['HOME'], id: "home", :nfs => true, :mount_options => ['nolock,vers=3,udp']
            end

            
            # setup provisioners ie: cloud config
            if File.exist?(CLOUD_CONFIG_PATH)
                config.vm.provision :file, :source => "#{CLOUD_CONFIG_PATH}", :destination => "/tmp/vagrantfile-user-data"
                config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
            end
        end
    end
end
