# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'net/http'
require 'open-uri'
require 'json'

class Module
  def redefine_const(name, value)
    __send__(:remove_const, name) if const_defined?(name)
    const_set(name, value)
  end
end

def getK8Sreleases()
  url = "https://api.github.com/repos/GoogleCloudPlatform/kubernetes/releases"
  json =  JSON.load(open(url))
  return json.collect { |item|
    if KUBERNETES_ALLOW_PRERELEASES
      item['tag_name'].gsub('v', '')
    else
      item['tag_name'].gsub('v', '') if item['prerelease'] == false
    end
  }.compact
end

required_plugins = %w(vagrant-triggers)
required_plugins.each do |plugin|
  need_restart = false
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
  exec "vagrant #{ARGV.join(' ')}" if need_restart
end

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">= 1.6.0"

MASTER_YAML = File.join(File.dirname(__FILE__), "master.yaml")
NODE_YAML = File.join(File.dirname(__FILE__), "node.yaml")

DOCKERCFG = File.expand_path(ENV['DOCKERCFG'] || "~/.dockercfg")
KUBERNETES_ALLOW_PRERELEASES = (ENV['KUBERNETES_ALLOW_PRERELEASES'] =~ (/(true|t|yes|y|1)$/i) && true) || false
KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || 'latest'
if KUBERNETES_VERSION == "latest"
  Object.redefine_const(:KUBERNETES_VERSION, getK8Sreleases[0])
end

CHANNEL = ENV['CHANNEL'] || 'alpha'
if CHANNEL != 'alpha'
  puts "============================================================================="
  puts "As this is a fastly evolving technology CoreOS' alpha channel is the only one"
  puts "expected to behave reliably. While one can invoke the beta or stable channels"
  puts "please be aware that your mileage may vary a whole lot."
  puts "So, before submitting a bug, in this project, or upstreams (either kubernetes"
  puts "or CoreOS) please make sure it (also) happens in the (default) alpha channel."
  puts "============================================================================="
end

COREOS_VERSION = ENV['COREOS_VERSION'] || 'latest'
upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_VERSION}"
if COREOS_VERSION == "latest"
  upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/current"
  url = "#{upstream}/version.txt"
  Object.redefine_const(:COREOS_VERSION,
    open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', ''))
end

NUM_INSTANCES = ENV['NUM_INSTANCES'] || 2

MASTER_MEM = ENV['MASTER_MEM'] || 512
MASTER_CPUS = ENV['MASTER_CPUS'] || 1

NODE_MEM= ENV['NODE_MEM'] || 1024
NODE_CPUS = ENV['NODE_CPUS'] || 1

ETCD_CLUSTER_SIZE = ENV['ETCD_CLUSTER_SIZE'] || 3

SERIAL_LOGGING = (ENV['SERIAL_LOGGING'].to_s.downcase == 'true')
GUI = (ENV['GUI'].to_s.downcase == 'true')

BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "172.17.8"

DNS_REPLICAS = ENV['DNS_REPLICAS'] || 2
DNS_DOMAIN = ENV['DNS_DOMAIN'] || "k8s.local"
DNS_UPSTREAM_SERVERS = ENV['DNS_UPSTREAM_SERVERS'] || "8.8.8.8:53,8.8.4.4:53"

tempCloudProvider = (ENV['CLOUD_PROVIDER'].to_s.downcase)
case tempCloudProvider
when "gce", "gke", "aws", "azure", "vagrant", "sphere", "libvirt-coreos", "juju"
  CLOUD_PROVIDER = tempCloudProvider
else
    CLOUD_PROVIDER = 'vagrant'
end
puts "Cloud provider: #{CLOUD_PROVIDER}"

(1..(NUM_INSTANCES.to_i + 1)).each do |i|
  if i == 1
    ETCD_SEED_CLUSTER = ""
    hostname = "master"
  else
    hostname = ",node-%02d" % (i - 1)
  end
  ETCD_SEED_CLUSTER.concat("#{hostname}=http://#{BASE_IP_ADDR}.#{i+100}:2380") if i <= ETCD_CLUSTER_SIZE
end

# Read YAML file with mountpoint details
MOUNT_POINTS = YAML::load_file('synced_folders.yaml')

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # always use Vagrants' insecure key
  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  config.vm.box = "coreos-#{CHANNEL}"
  config.vm.box_version = "= #{COREOS_VERSION}"
  config.vm.box_url = "#{upstream}/coreos_production_vagrant.json"

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "#{upstream}/coreos_production_vagrant_vmware_fusion.json"
    end
  end

  config.vm.provider :parallels do |vb, override|
    override.vm.box = "AntonioMeireles/coreos-#{CHANNEL}"
    override.vm.box_url = "https://vagrantcloud.com/AntonioMeireles/coreos-#{CHANNEL}"
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end
  config.vm.provider :parallels do |p|
    p.update_guest_tools = false
    p.check_guest_tools = false
  end

  # plugin conflict
  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")

  (1..(NUM_INSTANCES.to_i + 1)).each do |i|
    if i == 1
      hostname = "master"
      cfg = MASTER_YAML
      memory = MASTER_MEM
      cpus = MASTER_CPUS
      MASTER_IP="#{BASE_IP_ADDR}.#{i+100}"
    else
      hostname = "node-%02d" % (i - 1)
      cfg = NODE_YAML
      memory = NODE_MEM
      cpus = NODE_CPUS
    end

    config.vm.define vmName = hostname do |kHost|
      kHost.vm.hostname = vmName

      # vagrant-triggers has no concept of global triggers so to avoid having
      # then to run as many times as the total number of VMs we only call them
      # in the master (re: emyl/vagrant-triggers#13)...
      if vmName == "master"
        kHost.trigger.before [:up, :provision] do
          info "regenerating kubLocalSetup"
          system <<-EOT.prepend("\n\n") + "\n"
            cat kubLocalSetup.tmpl | \
             sed -e "s|__KUBERNETES_VERSION__|#{KUBERNETES_VERSION}|g" \
                 -e "s|__MASTER_IP__|#{MASTER_IP}|g" > kubLocalSetup
             chmod +x kubLocalSetup
          EOT
          info "making sure localhosts' 'kubectl' matches what we just booted..."
          system "./kubLocalSetup install"
        end
        # GoogleCloudPlatform/kubernetes#6380
        # blob bellow coming from stock + https://github.com/kelseyhightower/kube-register#15
        kHost.vm.provision :file, :source => "kube-register", :destination => "/tmp/kube-register"
        kHost.vm.provision :shell, :privileged => true,
          inline: <<-EOF
            echo "using customized kube-register to get post 0.15.x"
            chmod +x /tmp/kube-register
          EOF

        kHost.trigger.before [:up, :provision] do
          info "checking host platform..."
          system <<-EOT.prepend("\n\n") + "\n"
            which uname &>/dev/null || \
              ( echo "'uname' not found.";
              echo "looks like this host is not of unix type (linux or MacOS X)...";
              echo "...some features may not fully work, or just not work at all." )
          EOT
        end

        kHost.trigger.after [:up, :resume] do
          info "making sure ssh agent has the default vagrant key..."
          system "ssh-add ~/.vagrant.d/insecure_private_key"
          info "making sure local fleetctl won't misbehave with old staled data..."
          info "(wiping old ~/.fleetctl/known_hosts)"
          system "rm -rf ~/.fleetctl/known_hosts"
        end
        kHost.trigger.after [:up] do
          info "waiting for the cluster to be fully up..."
          system <<-EOT.prepend("\n\n") + "\n"
            until curl -o /dev/null -sIf http://#{MASTER_IP}:8080; do \
              sleep 1;
            done;
          EOT
          info "configuring k8s internal dns service"
          system <<-EOT.prepend("\n\n") + "\n"
            cd defaultServices/dns
            sed -e "s|__MASTER_IP__|#{MASTER_IP}|g" \
                -e "s|__DNS_REPLICAS__|#{DNS_REPLICAS}|g" \
                -e "s|__DNS_DOMAIN__|#{DNS_DOMAIN}|g" \
                -e "s|__DNS_UPSTREAM_SERVERS__|#{DNS_UPSTREAM_SERVERS}|g" \
              dns-controller.yaml.tmpl > dns-controller.yaml
            cd ../..
            $(./kubLocalSetup shellinit)
            kubectl create -f defaultServices/dns/dns-controller.yaml
            kubectl create -f defaultServices/dns/dns-service.yaml
          EOT
          info "configuring k8s internal monitoring tools"
          system <<-EOT.prepend("\n\n") + "\n"
            $(./kubLocalSetup shellinit)
            kubectl create -f defaultServices/cluster-monitoring/grafana-service.yaml
            kubectl create -f defaultServices/cluster-monitoring/heapster-service.yaml
            kubectl create -f defaultServices/cluster-monitoring/influxdb-service.yaml
            kubectl create -f defaultServices/cluster-monitoring/heapster-controller.yaml
            kubectl create -f defaultServices/cluster-monitoring/influxdb-grafana-controller.yaml
          EOT
        end
        kHost.trigger.after [:up, :resume] do
          info "============================================================================="
          info ""
          info "  please don't forget to run '$(./kubLocalSetup shellinit)' "
          info "                                                                             "
          info "  see the documentation at"
          info "    https://github.com/AntonioMeireles/kubernetes-vagrant-coreos-cluster/    "
          info ""
          info "============================================================================="
        end
      end

      if SERIAL_LOGGING
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "#{vmName}-serial.txt")
        FileUtils.touch(serialFile)

        ["vmware_fusion", "vmware_workstation"].each do |vmware|
          kHost.vm.provider vmware do |v, override|
            v.vmx["serial0.present"] = "TRUE"
            v.vmx["serial0.fileType"] = "file"
            v.vmx["serial0.fileName"] = serialFile
            v.vmx["serial0.tryNoRxLoss"] = "FALSE"
          end
        end
        kHost.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
        # supported since vagrant-parallels 1.3.7
        # https://github.com/Parallels/vagrant-parallels/issues/164
        kHost.vm.provider :parallels do |v|
          v.customize("post-import",
            ["set", :id, "--device-add", "serial", "--output", serialFile])
          v.customize("pre-boot",
            ["set", :id, "--device-set", "serial0", "--output", serialFile])
        end
      end

      ["vmware_fusion", "vmware_workstation", "virtualbox"].each do |h|
        kHost.vm.provider h do |vb|
          vb.gui = GUI
        end
      end
      ["parallels", "virtualbox"].each do |h|
        kHost.vm.provider h do |n|
          n.memory = memory
          n.cpus = cpus
        end
      end

      kHost.vm.network :private_network, ip: "#{BASE_IP_ADDR}.#{i+100}"
      # you can override this in synced_folders.yaml
      kHost.vm.synced_folder ".", "/vagrant", disabled: true

      begin
        MOUNT_POINTS.each do |mount|
          mount_options = ""
          disabled = false
          nfs =  true
          mount_options = mount['mount_options'] if mount['mount_options']
          disabled = mount['disabled'] if mount['disabled']
          nfs = mount['nfs'] if mount['nfs']

          if File.exist?(File.expand_path("#{mount['source']}"))
            if mount['destination']
              kHost.vm.synced_folder "#{mount['source']}", "#{mount['destination']}",
                id: "#{mount['name']}",
                disabled: disabled,
                mount_options: ["#{mount_options}"],
                nfs: nfs
            end
          end
        end
      rescue
      end

      if File.exist?(DOCKERCFG)
        kHost.vm.provision :file, run: "always",
         :source => "#{DOCKERCFG}", :destination => "/home/core/.dockercfg"

        kHost.vm.provision :shell, run: "always" do |s|
          s.inline = "cp /home/core/.dockercfg /.dockercfg"
          s.privileged = true
        end
      end

      if File.exist?(cfg)
        kHost.vm.provision :file, :source => "#{cfg}", :destination => "/tmp/vagrantfile-user-data"
        kHost.vm.provision :shell, :privileged => true,
        inline: <<-EOF
          sed -i "s,__RELEASE__,v#{KUBERNETES_VERSION},g" /tmp/vagrantfile-user-data
          sed -i "s,__CHANNEL__,v#{CHANNEL},g" /tmp/vagrantfile-user-data
          sed -i "s,__NAME__,#{hostname},g" /tmp/vagrantfile-user-data
          sed -i "s|__ETCD_SEED_CLUSTER__|#{ETCD_SEED_CLUSTER}|g" /tmp/vagrantfile-user-data
          sed -i "s|__MASTER_IP__|#{MASTER_IP}|g" /tmp/vagrantfile-user-data
          sed -i "s,__CLOUDPROVIDER__,#{CLOUD_PROVIDER},g" /tmp/vagrantfile-user-data
          mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/
        EOF
      end
    end
  end
end
