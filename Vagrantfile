# -*- mode: ruby -*-
# vi: set ft=ruby :

# Before execute check path of virtualbox virtualmachines and fix it for your localsystem paths
# TODO
#   automatically fetch virtualbox virtualmachines working folder

# vagrant box update --box 'centos/7'
# vagrant validate
# vagrant up
# vagrant provision
# vagrant reload
# vagrant destroy -f

nodes = {
  :firewall => {
        :box_name => "centos/7",
        :net => [
                    {ip: '172.20.200.1', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "dmz-net"},
                    {ip: '172.21.200.1', adapter: 3, netmask: "255.255.255.0", virtualbox__intnet: "lan-net"},
                    {ip: '192.168.11.11', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: false},
                ]
  },
  :web1 => {
        :box_name => "centos/7",
        :net => [
                    {ip: '172.20.200.10', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "dmz-net"},
                ]
  },
  :web2 => {
        :box_name => "centos/7",
        :net => [
                    {ip: '172.20.200.20', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "dmz-net"},
                ]
  },
  :stuff => {
        :box_name => "centos/7",
        :net => [
                    {ip: '172.20.200.250', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "dmz-net"},
                ]
  },
  :db1 => {
        :box_name => "centos/7",
        :net => [
                    {ip: '172.21.200.30', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "lan-net"},
                ]
  },
  :db2 => {
        :box_name => "centos/7",
        :net => [
                    {ip: '172.21.200.40', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "lan-net"},
                ]
  },
}

Vagrant.configure(2) do |config|
    nodes.each do |boxname, boxconfig|
        config.vm.define boxname do |box|
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
          
            boxconfig[:net].each do |ipconf|
                box.vm.network "private_network", ipconf
            end
          
            box.vm.provider :virtualbox do |vb|
                vb.memory = "1024"
                vb.name = boxname.to_s
            end
            
            box.vm.provision "shell", run: "always", inline: <<-SHELL
                sysctl net.ipv6.conf.all.disable_ipv6=1
                sysctl net.ipv6.conf.default.disable_ipv6=1
                chronyc makestep
            SHELL
            
            box.vm.provision "shell", inline: <<-SHELL
                systemctl stop NetworkManager
                systemctl disable NetworkManager
                echo "ip_resolve=4" >> /etc/yum.conf
                echo "timeout=2" >> /etc/yum.conf
                echo "retries=1" >> /etc/yum.conf
                ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
                yum -y install epel-release
                yum -y update
                yum install -y vim tcpdump traceroute telnet nc
            SHELL

            if boxname.to_s == "firewall"
                box.vm.provision "shell", inline: <<-SHELL
                    # disabled cause added a lot of preconfigured rules
                    #systemctl start firewalld
                    #systemctl enable firewalld
                    echo "net.ipv4.conf.all.forwarding = 1" >> /etc/sysctl.conf
                    #echo "GATEWAY=192.168.11.1" >> /etc/sysconfig/network-scripts/ifcfg-eth3
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                    #iptables -t nat -A POSTROUTING ! -d 172.16.0.0/12 -o eth3 -j MASQUERADE
                    #iptables -t nat -A POSTROUTING ! -d 172.16.0.0/12 -o eth0 -j MASQUERADE
                    #iptables -t nat -A POSTROUTING -s 172.16.0.0/12 -o bond0 -j SNAT --to-source 192.168.255.18
                    #iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -o bond0 -j SNAT --to-source 192.168.255.18
                    #iptables -L -t nat
                    #iptables -L
                    echo "127.0.0.1       firewall        firewall" > /etc/hosts
                    echo "127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4" >> /etc/hosts
                    echo "::1         localhost localhost.localdomain localhost6 localhost6.localdomain6" >> /etc/hosts
                    echo "172.21.200.10 db" >> /etc/hosts
                    echo "172.21.200.30 db1" >> /etc/hosts
                    echo "172.21.200.40 db2" >> /etc/hosts
                    echo "172.20.200.5 web" >> /etc/hosts
                    echo "172.20.200.10 web1" >> /etc/hosts
                    echo "172.20.200.20 web2" >> /etc/hosts
                    echo "172.20.200.250 stuff" >> /etc/hosts
                SHELL
                box.vm.provision "shell", run: "always", inline: <<-SHELL
                    #echo "GATEWAY=192.168.11.1" >> /etc/sysconfig/network-scripts/ifcfg-eth3
                    #systemctl restart network
                    iptables -F -t nat
                    iptables -t nat -A POSTROUTING ! -d 172.16.0.0/12 -o eth0 -j MASQUERADE
                    #
                    iptables -F
                    # 
                    iptables -t nat -A PREROUTING -p tcp -i eth3 --dport 80 -j DNAT --to-destination 172.20.200.5:80
                    iptables -t nat -A POSTROUTING -p tcp -d 172.20.200.5 --dport 80 -j SNAT --to-source 172.20.200.1
                    #
                    iptables -t nat -A PREROUTING -p tcp -i eth3 --dport 443 -j DNAT --to-destination 172.20.200.5:443
                    iptables -t nat -A POSTROUTING -p tcp -d 172.20.200.5 --dport 443 -j SNAT --to-source 172.20.200.1
                    #
                    iptables -t nat -A PREROUTING -p tcp -i eth3 --dport 8181 -j DNAT --to-destination 172.20.200.10:80
                    iptables -t nat -A POSTROUTING -p tcp -d 172.20.200.10 --dport 8181 -j SNAT --to-source 172.20.200.1
                    #
                    iptables -t nat -A PREROUTING -p tcp -i eth3 --dport 8282 -j DNAT --to-destination 172.20.200.20:80
                    iptables -t nat -A POSTROUTING -p tcp -d 172.20.200.20 --dport 8282 -j SNAT --to-source 172.20.200.1
                    #
                    sysctl net.ipv4.conf.all.forwarding=1
                    #cat /proc/sys/net/ipv4/ip_forward
                SHELL
            elsif boxname.to_s ==  "web1"
                # adding disk for drbd
                box.vm.provider :virtualbox do |vb|
                    disk = '/mnt/d/VirtualBoxVms/web1/sata1.vdi'
                    #C:\mnt\d\VirtualBoxVms\web1
                    disk = '/mnt/c/mnt/d/VirtualBoxVms/web1/sata1.vdi'
                    disksize = 1024
                    diskport = 1
                    if (!(File.exist?(disk)))
                        vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                        vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', disksize]
                        vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', diskport, '--device', 0, '--type', 'hdd', '--medium', disk]
                    end
                end
            # httpd.conf; Listen 80 -> Listen 0.0.0.0:80
            # sed -i -e 's/Listen 80/Listen 0.0.0.0:80/' /etc/httpd/conf/httpd.conf
            # systemctl restart httpd
                box.vm.provision "shell", inline: <<-SHELL
                    echo "127.0.0.1       web1.local        web1.local" > /etc/hosts
                    echo "127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4" >> /etc/hosts
                    echo "::1         localhost localhost.localdomain localhost6 localhost6.localdomain6" >> /etc/hosts
                    echo "172.20.200.1 firewall" >> /etc/hosts
                    echo "172.21.200.10 db" >> /etc/hosts
                    echo "172.21.200.30 db1" >> /etc/hosts
                    echo "172.21.200.40 db2" >> /etc/hosts
                    echo "172.20.200.5 web" >> /etc/hosts
                    echo "172.20.200.10 web1" >> /etc/hosts
                    echo "172.20.200.20 web2" >> /etc/hosts
                    echo "172.20.200.250 stuff" >> /etc/hosts
                SHELL
                box.vm.provision "shell", run: "always", inline: <<-SHELL
                    echo "GATEWAY=172.20.200.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                SHELL
            elsif boxname.to_s ==  "web2"
                # adding disk for drbd
                box.vm.provider :virtualbox do |vb|
                    #disk = '/mnt/d/VirtualBoxVms/web2/sata1.vdi'
                    disk = '/mnt/c/mnt/d/VirtualBoxVms/web2/sata1.vdi'
                    disksize = 1024
                    diskport = 1
                    if (!(File.exist?(disk)))
                        vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                        vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', disksize]
                        vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', diskport, '--device', 0, '--type', 'hdd', '--medium', disk]
                    end
                end
                box.vm.provision "shell", inline: <<-SHELL
                    echo "127.0.0.1       web2.local        web2.local" > /etc/hosts
                    echo "127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4" >> /etc/hosts
                    echo "::1         localhost localhost.localdomain localhost6 localhost6.localdomain6" >> /etc/hosts
                    echo "172.20.200.1 firewall" >> /etc/hosts
                    echo "172.21.200.10 db" >> /etc/hosts
                    echo "172.21.200.30 db1" >> /etc/hosts
                    echo "172.21.200.40 db2" >> /etc/hosts
                    echo "172.20.200.5 web" >> /etc/hosts
                    echo "172.20.200.10 web1" >> /etc/hosts
                    echo "172.20.200.20 web2" >> /etc/hosts
                    echo "172.20.200.250 stuff" >> /etc/hosts
                SHELL
                box.vm.provision "shell", run: "always", inline: <<-SHELL
                    echo "GATEWAY=172.20.200.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                SHELL
            elsif boxname.to_s ==  "stuff"
                box.vm.provision "shell", inline: <<-SHELL
                    echo "GATEWAY=172.20.200.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                    echo "127.0.0.1       stuff.local        stuff.local" > /etc/hosts
                    echo "127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4" >> /etc/hosts
                    echo "::1         localhost localhost.localdomain localhost6 localhost6.localdomain6" >> /etc/hosts
                    echo "172.20.200.1 firewall" >> /etc/hosts
                    echo "172.21.200.10 db" >> /etc/hosts
                    echo "172.21.200.30 db1" >> /etc/hosts
                    echo "172.21.200.40 db2" >> /etc/hosts
                    echo "172.20.200.5 web" >> /etc/hosts
                    echo "172.20.200.10 web1" >> /etc/hosts
                    echo "172.20.200.20 web2" >> /etc/hosts
                    echo "172.20.200.250 stuff" >> /etc/hosts
                SHELL
                box.vm.provision "shell", run: "always", inline: <<-SHELL
                    echo "GATEWAY=172.20.200.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                SHELL
            elsif boxname.to_s ==  "db1"
                # adding disk for drbd
                box.vm.provider :virtualbox do |vb|
                    #disk = '/mnt/d/VirtualBoxVms/db1/sata1.vdi'
                    #disk = '/mnt/c/mnt/d/VirtualBoxVms/db1/sata1.vdi'
                    disk = '/mnt/c/mnt/c/mnt/d/VirtualBoxVms/db1/sata1.vdi'
                    disksize = 1024
                    diskport = 1
                    if (!(File.exist?(disk)))
                        vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                        vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', disksize]
                        vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', diskport, '--device', 0, '--type', 'hdd', '--medium', disk]
                    end
                end
                box.vm.provision "shell", inline: <<-SHELL
                    echo "127.0.0.1       db1.local        db1.local" > /etc/hosts
                    echo "127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4" >> /etc/hosts
                    echo "::1         localhost localhost.localdomain localhost6 localhost6.localdomain6" >> /etc/hosts
                    echo "172.21.200.1 firewall" >> /etc/hosts
                    echo "172.21.200.10 db" >> /etc/hosts
                    echo "172.21.200.30 db1" >> /etc/hosts
                    echo "172.21.200.40 db2" >> /etc/hosts
                    echo "172.20.200.5 web" >> /etc/hosts
                    echo "172.20.200.10 web1" >> /etc/hosts
                    echo "172.20.200.20 web2" >> /etc/hosts
                    echo "172.20.200.250 stuff" >> /etc/hosts
                SHELL
                box.vm.provision "shell", run: "always", inline: <<-SHELL
                    echo "GATEWAY=172.21.200.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                SHELL
            elsif boxname.to_s ==  "db2"
                # adding disk for drbd
                box.vm.provider :virtualbox do |vb|
                    # after reload or supspend/resume path changes some time
                    # C:\mnt\d\VirtualBoxVms\db2\sata1.vdi ->
                    # /mnt/c/mnt/d/VirtualBoxVms/db2/sata1.vdi
                    #disk = '/mnt/d/VirtualBoxVms/db2/sata1.vdi'
                    disk = '/mnt/c/mnt/d/VirtualBoxVms/db2/sata1.vdi'
                    #C:\mnt\c\mnt\d\VirtualBoxVms\db2/sata1.vdi#
                    disk = '/mnt/c/mnt/c/mnt/d/VirtualBoxVms/db2/sata1.vdi'
                    disksize = 1024
                    diskport = 1
                    if (!(File.exist?(disk)))
                        vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                        vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', disksize]
                        vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', diskport, '--device', 0, '--type', 'hdd', '--medium', disk]
                    end
                end
                box.vm.provision "shell", inline: <<-SHELL
                    echo "127.0.0.1       db2.local        db2.local" > /etc/hosts
                    echo "127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4" >> /etc/hosts
                    echo "::1         localhost localhost.localdomain localhost6 localhost6.localdomain6" >> /etc/hosts
                    echo "172.21.200.1 firewall" >> /etc/hosts
                    echo "172.21.200.10 db" >> /etc/hosts
                    echo "172.21.200.30 db1" >> /etc/hosts
                    echo "172.21.200.40 db2" >> /etc/hosts
                    echo "172.20.200.5 web" >> /etc/hosts
                    echo "172.20.200.10 web1" >> /etc/hosts
                    echo "172.20.200.20 web2" >> /etc/hosts
                    echo "172.20.200.250 stuff" >> /etc/hosts
                SHELL
                box.vm.provision "shell", run: "always", inline: <<-SHELL
                    echo "GATEWAY=172.21.200.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
                    # https://serverfault.com/questions/660210/cant-start-centos-7-network-service
                    kill $(pgrep dhclient)
                    systemctl restart network
                SHELL
            end
            
        end
    end
end
