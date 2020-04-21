# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  
  config.vm.define "master", primary: true do |s|
    s.vm.box = "centos/7"
    s.vm.provider :virtualbox do |sb|
        sb.customize ["modifyvm", :id, "--memory", "1024"]
        sb.customize ["modifyvm", :id, "--cpus", "1"]
    end
    s.vm.hostname = 'master'
    s.vm.network "private_network", ip: "192.168.112.60"
    s.vm.provision "shell", inline: <<-SHELL
    	sudo yum update
	sudo yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
    sudo yum install http://repo.percona.com/centos/7/RPMS/x86_64/Percona-Server-selinux-56-5.6.42-rel84.2.el7.noarch.rpm -y
        sudo yum install Percona-Server-server-57.x86_64 -y
	sudo cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
	sudo rm /etc/my.cnf.d/01-base_slave.cnf
	sudo rm /etc/my.cnf.d/05-binlog_slave.cnf
	sudo systemctl start mysql
    	SHELL
  end

  config.vm.define "slave" do |c|
    c.vm.box = "centos/7"
    c.vm.provider :virtualbox do |cb|
        cb.customize ["modifyvm", :id, "--memory", "512"]
        cb.customize ["modifyvm", :id, "--cpus", "1"]
    end
    c.vm.hostname = 'slave'
    c.vm.network "private_network", ip: "192.168.112.61"
    c.vm.provision "shell", inline: <<-SHELL
    sudo yum update
    sudo yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
    sudo yum install http://repo.percona.com/centos/7/RPMS/x86_64/Percona-Server-selinux-56-5.6.42-rel84.2.el7.noarch.rpm -y
 	sudo yum install Percona-Server-server-57.x86_64 -y
	sudo cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
	sudo cp /vagrant/conf/my.cnf /etc/my.cnf
	sudo rm /etc/my.cnf.d/01-base.cnf
        sudo rm /etc/my.cnf.d/05-binlog.cnf
	sudo systemctl start mysql
    SHELL
    end
  end
