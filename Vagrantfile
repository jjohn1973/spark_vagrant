
hosts = {
  'node0' => {'hostname' => 'spmaster', 'ip' => '192.168.1.191', 'mac' => '080027001010', 'server_id' => '1' },
  'slave0' => {'hostname' => 'slave0', 'ip' => '192.168.1.192', 'mac' => '080027001011', 'server_id' => '2'},
  'slave1' => {'hostname' => 'slave1', 'ip' => '192.168.1.193', 'mac' => '080027001012', 'server_id' => '3'}
  'slave2' => {'hostname' => 'slave2', 'ip' => '192.168.1.194', 'mac' => '080028000100' , 'server_id' => '4'}
}


Vagrant.configure(2) do |config|
  hosts.keys.sort.each do |host|
    config.vm.define hosts[host]['hostname'] do |node|
      node.vm.box = 'centos/7'
      node.vm.box_url = 'centos/7'
      node.vm.synced_folder '.', '/scripts', disabled: true
      node.vm.network 'public_network', ip: hosts[host]['ip'], mac: hosts[host]['mac'], bridge: "Intel(R) 82574L Gigabit Network Connection #2"
      node.vm.provider 'virtualbox' do |v|
        v.memory = 2048
        v.cpus = 1
        # disable VBox time synchronization and use ntp
        v.customize ['setextradata', :id, 'VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled', 1]
      end
    end
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # stop firewallds
  $systemctl_stop_firewalld = <<SCRIPT
systemctl stop firewalld.service
SCRIPT
  # common settings on all machines
  $etc_hosts = <<SCRIPT
echo "$*" >> /etc/hosts
SCRIPT


$spark_master_service = <<-'SCRIPT'

cat > /etc/systemd/system/spark-master.service << EOF
[Unit]
Description=Apache Spark Master
After=network.target

[Service]
Type=forking
User=spark
Group=spark
ExecStart=/opt/spark/sbin/start-master.sh
ExecStop=/opt/spark/sbin/stop-master.sh

[Install]
WantedBy=multi-user.target
EOF

SCRIPT

$spark_slave_service = <<-'SCRIPT'

cat > /etc/systemd/system/spark-slave.service << EOF
[Unit]

Description=Apache Spark Slave

After=network.target

[Service]
Type=forking
User=spark
Group=spark
ExecStart=/opt/spark/sbin/start-slave.sh spark://spmaster:7077
ExecStop=/opt/spark/sbin/stop-slave.sh

[Install]
WantedBy=multi-user.target
EOF

SCRIPT

$spark_server_cnf = <<-'SCRIPT'

cat > /tmp/spark_server_cnf.sh << EOF
systemctl daemon-reload
systemctl start spark-master
systemctl enable spark-master
EOF
chmod +x /tmp/spark_server_cnf.sh
/tmp/spark_server_cnf.sh
SCRIPT

$spark_start_server_cnf = <<-'SCRIPT'

cat > /tmp/spark_server_cnf.sh << EOF
systemctl daemon-reload
systemctl start spark-slave
systemctl enable spark-slave
EOF
chmod +x /tmp/spark_server_cnf.sh
/tmp/spark_server_cnf.sh
SCRIPT


  # configure the second vagrant eth interface
  $ifcfg = <<SCRIPT
IPADDR=$1
NETMASK=$2
DEVICE=$3
TYPE=$4
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=$IPADDR
NETMASK=$NETMASK
DEVICE=$DEVICE
PEERDNS=no
TYPE=$TYPE
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT


  hosts.keys.sort.each do |host|
    config.vm.define hosts[host]['hostname'] do |node|
      node.vm.provision :shell, :inline => 'hostname ' + hosts[host]['hostname'], run: 'always'
      hosts.keys.sort.each do |k|
        node.vm.provision 'shell' do |s|
          s.inline = $etc_hosts
          s.args   = [hosts[k]['ip'], hosts[k]['hostname']]
        end
      end
      node.vm.provision :shell, :inline => $setenforce_0, run: 'always'
      node.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
      node.vm.provision 'shell' do |s|
        s.inline = $ifcfg
        s.args   = [hosts[host]['ip'], '255.255.255.0', 'eth1', 'Ethernet']
      end
      node.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
      # restarting network fixes RTNETLINK answers: File exists
      node.vm.provision :shell, :inline => 'systemctl restart network'
      node.vm.provision :shell, :inline => $linux_disable_ipv6, run: 'always'
      node.vm.provision :shell, :inline => 'yum -y install wget'
      node.vm.provision "shell", inline: <<-SHELL
        sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config 
        sed -i 's/PermitRootLogin no/PermitRootLogin yes/g' /etc/ssh/sshd_config  
        systemctl restart sshd.service
      SHELL

      node.vm.provision :shell, :inline => 'yum -y install net-tools'
      node.vm.provision :shell, :inline => 'yum -y install java-1.8.0-openjdk'
      node.vm.provision :shell, :inline => 'yum -y install python3'
      node.vm.provision :shell, :inline => 'wget https://dlcdn.apache.org/spark/spark-3.2.0/spark-3.2.0-bin-hadoop3.2.tgz --no-check-certificate'
      node.vm.provision :shell, :inline => 'tar -xvf spark-3.2.0-bin-hadoop3.2.tgz'
      node.vm.provision :shell, :inline => 'mv spark-3.2.0-bin-hadoop3.2 /opt/spark'
      node.vm.provision :shell, :inline => 'useradd spark'
      node.vm.provision :shell, :inline => 'chown -R spark:spark /opt/spark'
      #for master
      if host.start_with?("node")
        node.vm.provision :shell, :inline => $spark_master_service
        #node.vm.provision :shell, :inline => $spark_slave_service
        node.vm.provision :shell, :inline => 'systemctl daemon-reload'
        node.vm.provision :shell, :inline => 'systemctl start spark-master'
        node.vm.provision :shell, :inline => 'systemctl enable spark-master'
      end
      if host.start_with?("slave")
        node.vm.provision :shell, :inline => $spark_slave_service
        node.vm.provision :shell, :inline => 'systemctl daemon-reload'
        node.vm.provision :shell, :inline => 'systemctl start spark-slave'
        node.vm.provision :shell, :inline => 'systemctl enable spark-slave'
      end

      # install and enable ntp
      node.vm.provision :shell, :inline => 'yum -y install ntp'
      node.vm.provision :shell, :inline => 'systemctl enable ntpd'
      node.vm.provision :shell, :inline => 'systemctl start ntpd'
    end
  end
end
