ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'
Vagrant.configure("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.provider :virtualbox do |v|
    v.memory = 2048
    v.cpus = 2
  end

  # Define two VMs with static private IP addresses.
  boxes = [
    { :name => "web",
      :ip => "192.168.0.234",
    },
    { :name => "log",
      :ip => "192.168.0.235",
    } 
  ]
  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "public_network", ip: opts[:ip]

    end
  end
end
