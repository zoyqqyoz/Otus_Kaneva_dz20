# -*- mode: ruby -*-
# vim: set ft=ruby :
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :inetRouter => {
        :box_name => "centos/7",
        #:public => {:ip => '10.10.10.1', :adapter => 1},
        :net => [
                   {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"},
                ]
  },
  
  :inetRouter2 => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"}
               ]
  },
  
  :centralRouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.3', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"},
                   {ip: '192.168.0.1', adapter: 3, netmask: "255.255.255.240", virtualbox__intnet: "dir-net"},
                ]
  },

  :centralServer => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.0.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "dir-net"}
                ]
  },
}

Vagrant.configure("2") do |config|

config.vm.synced_folder ".", "/vagrant", disabled: true

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ip: ipconf[:ip]
        end

        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL

        case boxname.to_s
        when "inetRouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p
            sudo yum install -y iptables-services; sudo systemctl enable iptables && sudo systemctl start iptables;
            sudo sed -i 's/NETMASK=255.255.255.0/NETMASK=255.255.255.240/g' /etc/sysconfig/network-scripts/ifcfg-eth1
            sudo bash -c 'echo "192.168.0.0/16 via 192.168.255.2 dev eth1" > /etc/sysconfig/network-scripts/route-eth1'; sudo systemctl restart network
			# The rules are setup to open the standard SSH port 22 after a series of single knocks to the ports 8881, 7777 and 9991 in that order
			sudo iptables -P INPUT DROP
                        sudo iptables -P FORWARD ACCEPT
                        sudo iptables -P OUTPUT ACCEPT
                        sudo iptables -t nat -F
                        sudo iptables -t mangle -F
                        sudo iptables -F
                        sudo iptables -X
			sudo iptables -N TRAFFIC
			sudo iptables -N SSH-INPUTTWO
			sudo iptables -N SSH-INPUT
		#	sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
                        sudo iptables -A INPUT -j TRAFFIC
			sudo iptables -A TRAFFIC -m state --state ESTABLISHED,RELATED -j ACCEPT
                        sudo iptables -A TRAFFIC -p icmp -m icmp --icmp-type any -j ACCEPT
                        sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT
			sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
			sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
			sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
			sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
			sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
			sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP
			sudo iptables -A SSH-INPUT -m recent --name SSH1 --set -j DROP
			sudo iptables -A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP
			sudo iptables -A TRAFFIC -j DROP
                        sudo iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
			sudo service iptables save
                        echo "vagrant:vagrant" | sudo chpasswd
                        sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
                        systemctl restart sshd			
                        sudo reboot
             SHELL
		when "inetRouter2"
			#Forward port from host to guest
			box.vm.network "forwarded_port", guest: 8080, host: 1212, host_ip: "127.0.0.1", id: "nginx"
			box.vm.provision "shell", run: "always", inline: <<-SHELL
			sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p
                        sudo sed -i 's/NETMASK=255.255.255.0/NETMASK=255.255.255.240/g' /etc/sysconfig/network-scripts/ifcfg-eth1
			sudo yum install -y iptables-services; sudo systemctl enable iptables && sudo systemctl start iptables;
                        sudo iptables -P INPUT ACCEPT
                        sudo iptables -P FORWARD ACCEPT
                        sudo iptables -P OUTPUT ACCEPT
                        sudo iptables -t nat -F
                        sudo iptables -t mangle -F
                        sudo iptables -F
                        sudo iptables -X
			#Forward 8080 to nginx 80
			sudo iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
			#Back to inetRouter2
			sudo iptables -t nat -A POSTROUTING -d 192.168.0.2/32 -p tcp -m tcp --dport 80 -j SNAT --to-source 192.168.255.2
			sudo service iptables save
			echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0
			echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
			sudo bash -c 'echo "192.168.0.0/16 via 192.168.255.3 dev eth1" > /etc/sysconfig/network-scripts/route-eth1'
			sudo reboot
            SHELL
        when "centralRouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p
            sudo sed -i '$a DEFROUTE="no"' /etc/sysconfig/network-scripts/ifcfg-eth0
            sudo sed -i 's/NETMASK=255.255.255.0/NETMASK=255.255.255.240/g' /etc/sysconfig/network-scripts/ifcfg-eth1
            sudo sed -i 's/NETMASK=255.255.255.0/NETMASK=255.255.255.240/g' /etc/sysconfig/network-scripts/ifcfg-eth2
            echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
	    #for port knocking
	    sudo yum install -y epel-release; sudo yum install -y nmap
            sudo yum install -y traceroute
            sudo systemctl restart network
            sudo reboot
            SHELL
        when "centralServer"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            sudo sed -i '$a DEFROUTE="no"' /etc/sysconfig/network-scripts/ifcfg-eth0
            sudo sed -i 's/NETMASK=255.255.255.0/NETMASK=255.255.255.240/g' /etc/sysconfig/network-scripts/ifcfg-eth1
            echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
	    sudo yum install -y epel-release; sudo yum install -y nginx; sudo systemctl enable nginx; sudo systemctl start nginx
            sudo yum install -y traceroute
            sudo systemctl restart network
            sudo reboot
            SHELL
        
         end
      end
  end
end
