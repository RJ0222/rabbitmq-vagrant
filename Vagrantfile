BOX_IMAGE = "ubuntu/xenial64"
RABBIT_NODE_COUNT = 2
RABBIT_NODE_0_IP = "192.168.100.100"
RABBIT_NODE_IP_NW = "192.168.100."
MASTER_IP = "192.168.100.201"
BAKCUP_IP = "192.168.100.202"
VIP = "192.168.100.203"
LISTEN_PORT = "5672"
ADMIN_UESRNAME = "admin"
ADMIN_PASSWORD = "admin"

def gen_haproxy_backend(rabbit_node_count)
  server=""
  (1..rabbit_node_count).each do |i|
    ip = RABBIT_NODE_IP_NW + "#{10 + i}"
    server << "    server rabbitmq-server-#{i} #{ip}:5672 check\n"
  end
  server
end

init_script = <<SCRIPT
#!/bin/bash

sed -i 's/\\(archive\\|security\\)\\.ubuntu\\.com/mirrors\\.aliyun\\.com/g' /etc/apt/sources.list
apt-get update
apt-get install -y apt-transport-https
SCRIPT

rabbit_script = <<SCRIPT
curl -fsSL https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc | sudo apt-key add -
sudo apt-key adv --keyserver "hkps.pool.sks-keyservers.net" --recv-keys "0x6B73A36E6026DFCA"
echo "deb https://dl.bintray.com/rabbitmq/debian xenial main" | sudo tee /etc/apt/sources.list.d/bintray.rabbitmq.list
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install -y esl-erlang=1:20.3
sudo apt-get install -y rabbitmq-server
mkdir -p /var/lib/rabbitmq/
echo "test123" > /var/lib/rabbitmq/.erlang.cookie
systemctl restart rabbitmq-server
rabbitmq-plugins enable rabbitmq_management
rabbitmqctl add_user #{ADMIN_UESRNAME} #{ADMIN_PASSWORD}
rabbitmqctl set_user_tags #{ADMIN_UESRNAME} administrator
rabbitmqctl set_permissions -p / #{ADMIN_UESRNAME} ".*" ".*" ".*"
SCRIPT

node_script = <<SCRIPT
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbit-node-0
rabbitmqctl start_app
rabbitmqctl cluster_status
SCRIPT

ha_master_script = <<SCRIPT
#!/bin/bash

set -eo pipefail

status() {
    echo -e "\033[35m >>>   $*\033[0;39m"
}

status "configuring haproxy and keepalived.."
apt-get install -y keepalived haproxy

systemctl stop keepalived || true

vrrp_if=$(ip a | grep 192.168.100 | awk '{print $7}')
vrrp_ip=$(ip a | grep 192.168.100 | awk '{split($2, a, "/"); print a[1]}')

cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
    router_id LVS_DEVEL
}
vrrp_instance VI_1 {
    state master
    interface ${vrrp_if}
    mcast_src_ip ${vrrp_ip}
    virtual_router_id 51
    priority 101
    advert_int 2
    unicast_peer {
        #{BAKCUP_IP}
    }
    virtual_ipaddress {
        #{VIP}
    }
    nopreempt
}
EOF
systemctl start keepalived

cat > /etc/haproxy/haproxy.cfg <<EOF
global
  log /dev/log  local0
  log /dev/log  local1 notice
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client 50s
    timeout client-fin 50s
    timeout server 50s
    timeout tunnel 1h

listen stats
    bind *:1080
    stats refresh 30s
    stats uri /stats

listen rabbitmq-server
    bind *:#{LISTEN_PORT}
    mode tcp
    option tcplog
    balance roundrobin

#{gen_haproxy_backend(RABBIT_NODE_COUNT)}
EOF

systemctl restart haproxy
SCRIPT

ha_backup_script = <<SCRIPT
#!/bin/bash

set -eo pipefail

status() {
    echo -e "\033[35m >>>   $*\033[0;39m"
}

status "configuring haproxy and keepalived.."
apt-get install -y keepalived haproxy

systemctl stop keepalived || true

vrrp_if=$(ip a | grep 192.168.100 | awk '{print $7}')
vrrp_ip=$(ip a | grep 192.168.100 | awk '{split($2, a, "/"); print a[1]}')

cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
    router_id LVS_DEVEL
}
vrrp_instance VI_1 {
    state backup
    interface ${vrrp_if}
    mcast_src_ip ${vrrp_ip}
    virtual_router_id 51
    priority 100
    advert_int 2
    unicast_peer {
        #{MASTER_IP}
    }
    virtual_ipaddress {
        #{VIP}
    }
    nopreempt
}
EOF
systemctl start keepalived

cat > /etc/haproxy/haproxy.cfg <<EOF
global
  log /dev/log  local0
  log /dev/log  local1 notice
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client 50s
    timeout client-fin 50s
    timeout server 50s
    timeout tunnel 1h

listen stats
    bind *:1080
    stats refresh 30s
    stats uri /stats

listen rabbitmq-server
    bind *:#{LISTEN_PORT}
    mode tcp
    option tcplog
    balance roundrobin

#{gen_haproxy_backend(RABBIT_NODE_COUNT)}
EOF

systemctl restart haproxy
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = BOX_IMAGE
  config.vm.box_check_update = false

  config.vm.provider "virtualbox" do |l|
    l.cpus = 1
    l.memory = "1024"
  end

  config.vm.provision :shell, inline: init_script

  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true

  config.vm.define "rabbit-node-0" do |subconfig|
    subconfig.vm.hostname = "rabbit-node-0"
    subconfig.vm.network :private_network, nic_type: "virtio", ip: RABBIT_NODE_0_IP
    subconfig.vm.provision :shell, inline: rabbit_script
    # subconfig.vm.provider :virtualbox do |vb|
    #   vb.customize ["modifyvm", :id, "--cpus", "2"]
    #   vb.customize ["modifyvm", :id, "--memory", "2048"]
    # end
  end

  (1..RABBIT_NODE_COUNT).each do |i|
    config.vm.define "rabbit-node-#{i}" do |subconfig|
      subconfig.vm.hostname = "rabbit-node-#{i}"
      subconfig.vm.network :private_network, nic_type: "virtio", ip: RABBIT_NODE_IP_NW + "#{10 + i}"
      subconfig.vm.provision :shell, inline: rabbit_script
      subconfig.vm.provision :shell, inline: node_script
    end
  end

  config.vm.define "ha-node-master" do |subconfig|
    subconfig.vm.hostname = "ha-node-master"
    subconfig.vm.network :private_network, nic_type: "virtio", ip: MASTER_IP
    subconfig.vm.provision :shell, inline: ha_master_script
  end
  config.vm.define "ha-node-backup" do |subconfig|
    subconfig.vm.hostname = "ha-node-backup"
    subconfig.vm.network :private_network, nic_type: "virtio", ip: BAKCUP_IP
    subconfig.vm.provision :shell, inline: ha_backup_script
  end
end
