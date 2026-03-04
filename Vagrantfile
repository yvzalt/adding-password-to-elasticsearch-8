Vagrant.configure("2") do |config|
  

  (1..3).each do |i|
    config.vm.define "es-node-#{i}" do |node|
      node.vm.box = "almalinux/9"
      node.vm.hostname = "es-node-#{i}.local"
      node.vm.boot_timeout = 600
      

      node.vm.network "private_network", ip: "192.168.56.10#{i}"
      

      node.vm.provider "virtualbox" do |vb|
        vb.name = "es-node-#{i}"
        vb.memory = "2048"
        vb.cpus = 2
      end

      node.vm.provision "shell", inline: <<-SHELL
        echo "node #{i} starting..."
        sudo dnf update -y
        sudo dnf install -y epel-release nano wget curl bind-utils
        sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
        sudo tee /etc/yum.repos.d/elasticsearch.repo <<EOF
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
        sudo dnf install elasticsearch-8.5.3 -y

        sudo sysctl -w vm.max_map_count=262144
        echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf


      SHELL
    end
  end
end