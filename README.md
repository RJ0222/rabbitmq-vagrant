# kubeadm-vagrant

Run rabbitMQ cluster with haproxy-keepalived on vagrant.

# Requirements

1. virtualbox: https://www.virtualbox.org/wiki/Downloads
2. vagrant: https://www.vagrantup.com/downloads.html
3. `vagrant plugin install vagrant-hostmanager`

# Usage

Run `vagrant up` and wait for the cluster to be set up

```bash
vagrant ssh rabbit-node-0

rabbitmq cluster-status
```
