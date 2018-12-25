# configure L2TP over IPsec transport mode

## step1 - check Vagrantfile for vpn-server


	[2018-12-25 16:28.47]  /drives/c/home/ws/VMs/vpn-server
	[andrew.andrew-NB] ➤ cat Vagrantfile
	# -*- mode: ruby -*-
	# vi: set ft=ruby :
	
	Vagrant.configure(2) do |config|
	  # ----- my variables -----	
	  myvm_name = "vpn-server"
	  ipsec_psk=""
	  l2tpd_ip_range="172.16.1.100-172.16.1.200"
	  l2tpd_local_ip="172.16.1.1"
	  l2tpd_ppp_user="andrew"
	  l2tpd_ppp_passwd="G1y"
	  
	  # ----- Global -----	
	  config.vm.box_check_update = false
	  #config.vm.usable_port_range =  (2200..2250)
	
	  # ----- VM1 -----
	  config.vm.define "#{myvm_name}" do |config|
		config.vm.box = "trusty32"
		config.vm.box_url = "https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-i386-vagrant-disk1.box"
		config.vm.hostname = "#{myvm_name}"
		config.vm.host_name = "#{myvm_name}"
	
		config.vm.network :forwarded_port, guest: 22, host: 2222, id: "ssh", disabled: "true"
		config.vm.network :forwarded_port, guest: 22, host: 2202, auto_correct: true
		#config.vm.network "public_network", bridge: "Realtek PCIe GBE Family Controller", adapter: "1", ip: "192.168.90.222"
		config.vm.network "public_network",
							use_dhcp_assigned_default_route: true,
							bridge: "Realtek PCIe GBE Family Controller",
							#bridge: "Intel(R) Dual Band Wireless-AC 8260",
							#:dev => "eth1",
							#adapter: "1",
							:mode => 'bridge',
							auto_config: true
	
		config.vm.synced_folder "C:/home/ws/VMs/#{myvm_name}", "/vagrant"
	
		config.ssh.username = "vagrant"
		config.ssh.password = "vagrant"
		#config.ssh.insert_key = true
		config.ssh.insert_key = false
	
		config.vm.provider "virtualbox" do |vb|
			vb.memory = 768
			vb.cpus = 1
			vb.name = "vbox-#{myvm_name}"
			vb.customize ["modifyvm", :id, "--usb", "on"]
			vb.customize ["modifyvm", :id, "--usbehci", "on"]
			vb.customize ["modifyvm", :id, "--usbxhci", "on"]
			vb.customize ["usbfilter",
						  "add", "0",
						  "--target", :id,
						  "--active", "yes",
						  "--name", "Prolific Technology Inc. USB-Serial Controller",
						  "--vendorid", "067b",
						  "--productid", "2303",
						  "--revision", "0300",
						  "--manufacturer", "Prolific Technology Inc.",
						  "--product", "USB-Serial Controller",
						  "--remote", "no"]
		end
	
		# ----- VM1:provision:INIT_CONFIG -----
		config.vm.provision "shell", 
		run: "never", 
		inline: <<-INIT_CONFIG
			echo "*** initial config ***"
			export LANGUAGE=en_US.UTF-8
			export LANG=en_US.UTF-8
			export LC_ALL=en_US.UTF-8
			locale-gen en_US.UTF-8
			dpkg-reconfigure locales
		INIT_CONFIG
	
		# ----- VM1:provision:PKG_INSTALL -----
		#   you can also use: run: "never|always",
		config.vm.provision "shell", 
		#run: "never",
		inline: <<-PKG_INSTALL
			echo "*** update apt-repository ***"
			apt-get update -qq
	
			echo "*** installing useful pkgs ***"
			DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends lftp tree vim unzip git
			#DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends git-review subversion ctags tmux dos2unix samba
	
			#echo "*** installing pkgs for OpenWRT build system ***"
			#DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
			#					 build-essential libncurses5-dev zlib1g-dev git-core \
			#					 gawk flex quilt xsltproc mercurial unzip
			#DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
			#					 texlive-base texlive-fonts-recommended texlive-fonts-extra \
			#					 texlive-latex-base texlive-latex-extra tex4ht
	
			#echo "*** installing repo ***"
			#if [ ! -f /usr/local/bin/repo ]; then
			#	curl -s https://storage.googleapis.com/git-repo-downloads/repo > /usr/local/bin/repo
			#	chmod a+x /usr/local/bin/repo
			#fi
	
		PKG_INSTALL
	
		# ----- VM1:provision:ENV_CONFIG -----
		config.vm.provision "shell", 
		#run: "never",
		inline: <<-ENV_CONFIG
			echo "*** config timezone ***"
			sudo timedatectl set-ntp true
			sudo timedatectl set-timezone 'Asia/Taipei'
	
			echo "*** linking /bin/sh to bash instead of dash ***"
			echo "dash dash/sh boolean false" | debconf-set-selections
			DEBIAN_FRONTEND=noninteractive dpkg-reconfigure dash
		ENV_CONFIG
	
		# ----- VM1:provision:TEST_CONFIG -----
		config.vm.provision "shell", 
		run: "never",
		inline: <<-TEST_CONFIG
			echo "*** only for test ***"
		TEST_CONFIG
	
		# ----- VM1:provision:NET_CONFIG -----
		config.vm.provision "shell", 
		run: "always", 
		#run: "never",
		inline: <<-NET_CONFIG
			echo "*** customize network config ***"
			#route del default gw 10.0.2.2
			#route add default gw 10.5.161.254
			for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done
		NET_CONFIG
	
		# ----- VM1:provision:CONFIG_VPN -----
		config.vm.provision "shell", 
		#run: "never",
		inline: <<-CONFIG_VPN
			echo "*** customize VPN config ***"
			DEBIAN_FRONTEND=noninteractive apt-get install -y -qq openswan xl2tpd ppp lsof
			
			config_file="/etc/sysctl.conf"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			echo "net.ipv4.ip_forward = 1" | tee -a $config_file
			echo "net.ipv4.conf.all.accept_redirects = 0"| tee -a $config_file
			echo "net.ipv4.conf.all.send_redirects = 0"| tee -a $config_file
			
			for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done
			sysctl -p
			#ipsec verify
	
			config_file="/etc/ipsec.conf"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	#
	
	version 2.0
	
	config setup
	    dumpdir=/var/run/pluto/
	    nat_traversal=yes
	    virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v4:25.0.0.0/8,%v6:fd00::/8,%v6:fe80::/10
	    oe=off
	    protostack=netkey
	    #plutoopts="--interface eth1"
	    #plutostderrlog=/var/log/ipsec.log
	    #force_keepalive=yes
	    #keep_alive=60
	
	conn %default
	    auto=add
	    authby=secret
	    pfs=no
	    keyingtries=3
	    ikelifetime=8h
	    keylife=1h
	    dpddelay=10
	    dpdtimeout=20
	    dpdaction=clear
	
	conn L2TP-PSK-noNAT
	    #type=tunnel
	    type=transport
	    ike=aes256-sha1,aes128-sha1,3des-sha1
	    phase2alg=aes256-sha1,aes128-sha1,3des-sha1
	    left=%defaultroute
	    #left=`ifconfig eth1 | awk '/inet addr/{print substr($2,6)}'`
	    leftprotoport=17/1701
	    right=%any
	    rightprotoport=17/%any
	EOF
	
			config_file="/etc/ipsec.secrets"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			ipsec_psk=`openssl rand -hex 30`
			cat <<EOF >$config_file
	`ifconfig eth1 | awk '/inet addr/{print substr($2,6)}'`   %any:  PSK "$ipsec_psk"
	EOF
			config_file="/etc/xl2tpd/xl2tpd.conf"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	[global]
	ipsec saref = no
	; saref refinfo = 30
	; debug avp = yes
	; debug network = yes
	; debug state = yes
	; debug tunnel = yes
	
	[lns default]
	ip range = #{l2tpd_ip_range}
	local ip = #{l2tpd_local_ip}
	require chap = yes
	refuse pap = yes
	require authentication = yes
	ppp debug = yes
	pppoptfile = /etc/ppp/options.xl2tpd
	length bit = yes
	EOF
	
			config_file="/etc/ppp/options.xl2tpd"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	require-mschap-v2
	ms-dns 8.8.8.8
	ms-dns 8.8.4.4
	auth
	mtu 1200
	mru 1000
	crtscts
	hide-password
	modem
	name l2tpd
	proxyarp
	lcp-echo-interval 30
	lcp-echo-failure 4
	EOF
	
			config_file="/etc/ppp/chap-secrets"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	# Secrets for authentication using CHAP
	# client	server	secret			IP addresses
	#{l2tpd_ppp_user}		l2tpd	#{l2tpd_ppp_passwd}		*
	EOF
		CONFIG_VPN
	
		# ----- VM1:provision:CONFIG_VPN_START -----
		config.vm.provision "shell", 
		run: "always",
		inline: <<-CONFIG_VPN_START
		for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done
		/etc/init.d/ipsec restart
		/etc/init.d/xl2tpd restart
		CONFIG_VPN_START
		
		# ----- VM1:provision:CONFIG_DONE -----
		config.vm.provision "shell", 
		run: "always", 
		inline: <<-CONFIG_DONE
			echo ""
			echo "*** ip address: $(ifconfig eth1 | awk '/inet addr/{print substr($2,6)}') ***"
			echo ""
			echo "*** VPN info ***"
			test -f /etc/ipsec.conf && echo "vpn server: $(ifconfig eth1 | awk '/inet addr/{print substr($2,6)}')"
			test -f /etc/ppp/chap-secrets && echo "l2tpd ppp : $(cat /etc/ppp/chap-secrets | tail -n 1 | awk '{print $1}') / $(cat /etc/ppp/chap-secrets | tail -n 1 | awk '{print $3}')"
			test -f /etc/ipsec.secrets && echo "ipsec psk : $(cat /etc/ipsec.secrets | tail -n 1 | awk '{print $4}')"
		CONFIG_DONE
	
		config.vm.post_up_message = <<-MESSAGE
	*** config.vm.was started :-) ***
		MESSAGE
	  end
	  # ----- END VM1 -----
	end


## step2 - start up a vpn-server


	[2018-12-25 16:28.50]  /drives/c/home/ws/VMs/vpn-server
	[andrew.andrew-NB] ➤ vagrant.exe up
	Bringing machine 'vpn-server' up with 'virtualbox' provider...
	==> vpn-server: Importing base box 'trusty32'...
	==> vpn-server: Matching MAC address for NAT networking...
	==> vpn-server: Setting the name of the VM: vbox-vpn-server
	==> vpn-server: Clearing any previously set forwarded ports...
	==> vpn-server: Clearing any previously set network interfaces...
	==> vpn-server: Preparing network interfaces based on configuration...
	    vpn-server: Adapter 1: nat
	    vpn-server: Adapter 2: bridged
	==> vpn-server: Forwarding ports...
	    vpn-server: 22 (guest) => 2202 (host) (adapter 1)
	==> vpn-server: Running 'pre-boot' VM customizations...
	==> vpn-server: Booting VM...
	==> vpn-server: Waiting for machine to boot. This may take a few minutes...
	    vpn-server: SSH address: 127.0.0.1:2202
	    vpn-server: SSH username: vagrant
	    vpn-server: SSH auth method: password
	    vpn-server: Warning: Authentication failure. Retrying...
	==> vpn-server: Machine booted and ready!
	==> vpn-server: Checking for guest additions in VM...
	    vpn-server: The guest additions on this VM do not match the installed version of
	    vpn-server: VirtualBox! In most cases this is fine, but in rare cases it can
	    vpn-server: prevent things such as shared folders from working properly. If you see
	    vpn-server: shared folder errors, please make sure the guest additions within the
	    vpn-server: virtual machine match the version of VirtualBox you have installed on
	    vpn-server: your host and reload your VM.
	    vpn-server:
	    vpn-server: Guest Additions Version: 4.3.36
	    vpn-server: VirtualBox Version: 5.2
	==> vpn-server: Setting hostname...
	==> vpn-server: Configuring and enabling network interfaces...
	==> vpn-server: Mounting shared folders...
	    vpn-server: /vagrant => C:/home/ws/VMs/vpn-server
	==> vpn-server: Running provisioner: shell...
	    vpn-server: Running: inline script
	    vpn-server: *** update apt-repository ***
	    vpn-server: *** installing useful pkgs ***
	    vpn-server: Selecting previously unselected package liberror-perl.
	    vpn-server: (Reading database ... 63230 files and directories currently installed.)
	    vpn-server: Preparing to unpack .../liberror-perl_0.17-1.1_all.deb ...
	    vpn-server: Unpacking liberror-perl (0.17-1.1) ...
	    vpn-server: Selecting previously unselected package git-man.
	    vpn-server: Preparing to unpack .../git-man_1%3a1.9.1-1ubuntu0.10_all.deb ...
	    vpn-server: Unpacking git-man (1:1.9.1-1ubuntu0.10) ...
	    vpn-server: Selecting previously unselected package git.
	    vpn-server: Preparing to unpack .../git_1%3a1.9.1-1ubuntu0.10_i386.deb ...
	    vpn-server: Unpacking git (1:1.9.1-1ubuntu0.10) ...
	    vpn-server: Selecting previously unselected package lftp.
	    vpn-server: Preparing to unpack .../lftp_4.4.13-1ubuntu0.1_i386.deb ...
	    vpn-server: Unpacking lftp (4.4.13-1ubuntu0.1) ...
	    vpn-server: Selecting previously unselected package tree.
	    vpn-server: Preparing to unpack .../archives/tree_1.6.0-1_i386.deb ...
	    vpn-server: Unpacking tree (1.6.0-1) ...
	    vpn-server: Selecting previously unselected package unzip.
	    vpn-server: Preparing to unpack .../unzip_6.0-9ubuntu1.5_i386.deb ...
	    vpn-server: Unpacking unzip (6.0-9ubuntu1.5) ...
	    vpn-server: Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
	    vpn-server: Processing triggers for mime-support (3.54ubuntu1.1) ...
	    vpn-server: Setting up liberror-perl (0.17-1.1) ...
	    vpn-server: Setting up git-man (1:1.9.1-1ubuntu0.10) ...
	    vpn-server: Setting up git (1:1.9.1-1ubuntu0.10) ...
	    vpn-server: Setting up lftp (4.4.13-1ubuntu0.1) ...
	    vpn-server: Setting up tree (1.6.0-1) ...
	    vpn-server: Setting up unzip (6.0-9ubuntu1.5) ...
	==> vpn-server: Running provisioner: shell...
	    vpn-server: Running: inline script
	    vpn-server: *** config timezone ***
	    vpn-server: *** linking /bin/sh to bash instead of dash ***
	    vpn-server: Removing 'diversion of /bin/sh to /bin/sh.distrib by dash'
	    vpn-server: Adding 'diversion of /bin/sh to /bin/sh.distrib by bash'
	    vpn-server: Removing 'diversion of /usr/share/man/man1/sh.1.gz to /usr/share/man/man1/sh.distrib.1.gz by dash'
	    vpn-server: Adding 'diversion of /usr/share/man/man1/sh.1.gz to /usr/share/man/man1/sh.distrib.1.gz by bash'
	==> vpn-server: Running provisioner: shell...
	    vpn-server: Running: inline script
	    vpn-server: *** customize network config ***
	==> vpn-server: Running provisioner: shell...
	    vpn-server: Running: inline script
	    vpn-server: *** customize VPN config ***
	    vpn-server: Preconfiguring packages ...
	    vpn-server: Selecting previously unselected package iproute.
	    vpn-server: (Reading database ... 64026 files and directories currently installed.)
	    vpn-server: Preparing to unpack .../iproute_1%3a3.12.0-2ubuntu1.2_all.deb ...
	    vpn-server: Unpacking iproute (1:3.12.0-2ubuntu1.2) ...
	    vpn-server: Selecting previously unselected package openswan.
	    vpn-server: Preparing to unpack .../openswan_1%3a2.6.38-1_i386.deb ...
	    vpn-server: Unpacking openswan (1:2.6.38-1) ...
	    vpn-server: Preparing to unpack .../ppp_2.4.5-5.1ubuntu2.3_i386.deb ...
	    vpn-server: Unpacking ppp (2.4.5-5.1ubuntu2.3) over (2.4.5-5.1ubuntu2.2) ...
	    vpn-server: Selecting previously unselected package xl2tpd.
	    vpn-server: Preparing to unpack .../xl2tpd_1.3.6+dfsg-1_i386.deb ...
	    vpn-server: Unpacking xl2tpd (1.3.6+dfsg-1) ...
	    vpn-server: Processing triggers for ureadahead (0.100.0-16) ...
	    vpn-server: Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
	    vpn-server: Setting up iproute (1:3.12.0-2ubuntu1.2) ...
	    vpn-server: Setting up openswan (1:2.6.38-1) ...
	    vpn-server: ipsec_setup: Starting Openswan IPsec 2.6.38...
	    vpn-server: ipsec_setup: No KLIPS support found while requested, desperately falling back to netkey
	    vpn-server: ipsec_setup: NETKEY support found. Use protostack=netkey in /etc/ipsec.conf to avoid attempts to use KLIPS. Attempting to continue with NETKEY
	    vpn-server: Setting up ppp (2.4.5-5.1ubuntu2.3) ...
	    vpn-server: Setting up xl2tpd (1.3.6+dfsg-1) ...
	    vpn-server: Starting xl2tpd:
	    vpn-server: xl2tpd.
	    vpn-server: Processing triggers for ureadahead (0.100.0-16) ...
	    vpn-server: net.ipv4.ip_forward = 1
	    vpn-server: net.ipv4.conf.all.accept_redirects = 0
	    vpn-server: net.ipv4.conf.all.send_redirects = 0
	    vpn-server: net.ipv4.ip_forward = 1
	    vpn-server: net.ipv4.conf.all.accept_redirects = 0
	    vpn-server: net.ipv4.conf.all.send_redirects = 0
	==> vpn-server: Running provisioner: shell...
	    vpn-server: Running: inline script
	    vpn-server: ipsec_setup: Stopping Openswan IPsec...
	    vpn-server: ipsec_setup: Starting Openswan IPsec U2.6.38/K3.13.0-157-generic...
	    vpn-server: Restarting xl2tpd:
	    vpn-server: xl2tpd.
	==> vpn-server: Running provisioner: shell...
	    vpn-server: Running: inline script
	    vpn-server: *** ip address: 192.168.90.146 ***
	    vpn-server: *** VPN info ***
	    vpn-server: vpn server: 192.168.90.146
	    vpn-server: l2tpd ppp : andrew / G1y
	    vpn-server: ipsec psk : "dd955d5ac64cee1e9ca56a8036ee70150f835aefebea5654dbd39070cad8"
	
	==> vpn-server: Machine 'vpn-server' has a post `vagrant up` message. This is a message
	==> vpn-server: from the creator of the Vagrantfile, and not from Vagrant itself:
	==> vpn-server:
	==> vpn-server: *** config.vm.was started :-) ***


Remember the info:

    vpn-server: vpn server: 192.168.90.146
    vpn-server: l2tpd ppp : andrew / G1y
    vpn-server: ipsec psk : "dd955d5ac64cee1e9ca56a8036ee70150f835aefebea5654dbd39070cad8"


## step3 - edit & check Vagrantfile for vpn-client


Edit:


	vpn_server_ip="192.168.90.146"	# refer to vpn server settings (left@/etc/ipsec.conf)
	vpn_ipsec_psk="dd955d5ac64cee1e9ca56a8036ee70150f835aefebea5654dbd39070cad8"	# refer to vpn server settings (/etc/ipsec.secrets)
	l2tpd_ppp_user="andrew"			# refer to vpn server settings (/etc/ppp/chap-secrets)
	l2tpd_ppp_passwd="G1y"			# refer to vpn server settings (/etc/ppp/chap-secrets)


Check:


	[2018-12-25 16:18.25]  /drives/c/home/ws/VMs/vpn-client
	[andrew.andrew-NB] ➤ cat Vagrantfile
	# -*- mode: ruby -*-
	# vi: set ft=ruby :
	
	Vagrant.configure(2) do |config|
	  # ----- my variables -----	
	  myvm_name = "vpn-client"
	  vpn_server_ip="192.168.90.146"	# refer to vpn server settings (left@/etc/ipsec.conf)
	  vpn_ipsec_psk="dd955d5ac64cee1e9ca56a8036ee70150f835aefebea5654dbd39070cad8"	# refer to vpn server settings (/etc/ipsec.secrets)
	  l2tpd_ppp_user="andrew"			# refer to vpn server settings (/etc/ppp/chap-secrets)
	  l2tpd_ppp_passwd="G1y"			# refer to vpn server settings (/etc/ppp/chap-secrets)
	  ipsec_conn_name="L2TP-PSK"
	  vpn_lac_name="vpn-connection"
	  
	  # ----- Global -----	
	  config.vm.box_check_update = false
	  #config.vm.usable_port_range =  (2200..2250)
	
	  # ----- VM1 -----
	  config.vm.define "#{myvm_name}" do |config|
		config.vm.box = "trusty32"
		config.vm.box_url = "https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-i386-vagrant-disk1.box"
		config.vm.hostname = "#{myvm_name}"
		config.vm.host_name = "#{myvm_name}"
	
		config.vm.network :forwarded_port, guest: 22, host: 2222, id: "ssh", disabled: "true"
		config.vm.network :forwarded_port, guest: 22, host: 2203, auto_correct: true
		#config.vm.network "public_network", bridge: "Realtek PCIe GBE Family Controller", adapter: "1", ip: "192.168.90.222"
		config.vm.network "public_network",
							use_dhcp_assigned_default_route: true,
							bridge: "Realtek PCIe GBE Family Controller",
							#bridge: "Intel(R) Dual Band Wireless-AC 8260",
							#:dev => "eth1",
							#adapter: "1",
							:mode => 'bridge',
							auto_config: true
	
		config.vm.synced_folder "C:/home/ws/VMs/#{myvm_name}", "/vagrant"
	
		config.ssh.username = "vagrant"
		config.ssh.password = "vagrant"
		#config.ssh.insert_key = true
		config.ssh.insert_key = false
	
		config.vm.provider "virtualbox" do |vb|
			vb.memory = 768
			vb.cpus = 1
			vb.name = "vbox-#{myvm_name}"
			vb.customize ["modifyvm", :id, "--usb", "on"]
			vb.customize ["modifyvm", :id, "--usbehci", "on"]
			vb.customize ["modifyvm", :id, "--usbxhci", "on"]
			vb.customize ["usbfilter",
						  "add", "0",
						  "--target", :id,
						  "--active", "yes",
						  "--name", "Prolific Technology Inc. USB-Serial Controller",
						  "--vendorid", "067b",
						  "--productid", "2303",
						  "--revision", "0300",
						  "--manufacturer", "Prolific Technology Inc.",
						  "--product", "USB-Serial Controller",
						  "--remote", "no"]
		end
	
		# ----- VM1:provision:INIT_CONFIG -----
		config.vm.provision "shell", 
		run: "never", 
		inline: <<-INIT_CONFIG
			echo "*** initial config ***"
			export LANGUAGE=en_US.UTF-8
			export LANG=en_US.UTF-8
			export LC_ALL=en_US.UTF-8
			locale-gen en_US.UTF-8
			dpkg-reconfigure locales
		INIT_CONFIG
	
		# ----- VM1:provision:PKG_INSTALL -----
		#   you can also use: run: "never|always",
		config.vm.provision "shell", 
		#run: "never",
		inline: <<-PKG_INSTALL
			echo "*** update apt-repository ***"
			apt-get update -qq
	
			echo "*** installing useful pkgs ***"
			DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends lftp tree vim unzip git
			#DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends git-review subversion ctags tmux dos2unix samba
	
			#echo "*** installing pkgs for OpenWRT build system ***"
			#DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
			#					 build-essential libncurses5-dev zlib1g-dev git-core \
			#					 gawk flex quilt xsltproc mercurial unzip
			#DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
			#					 texlive-base texlive-fonts-recommended texlive-fonts-extra \
			#					 texlive-latex-base texlive-latex-extra tex4ht
	
			#echo "*** installing repo ***"
			#if [ ! -f /usr/local/bin/repo ]; then
			#	curl -s https://storage.googleapis.com/git-repo-downloads/repo > /usr/local/bin/repo
			#	chmod a+x /usr/local/bin/repo
			#fi
	
		PKG_INSTALL
	
		# ----- VM1:provision:ENV_CONFIG -----
		config.vm.provision "shell", 
		#run: "never",
		inline: <<-ENV_CONFIG
			echo "*** config timezone ***"
			sudo timedatectl set-ntp true
			sudo timedatectl set-timezone 'Asia/Taipei'
	
			echo "*** linking /bin/sh to bash instead of dash ***"
			echo "dash dash/sh boolean false" | debconf-set-selections
			DEBIAN_FRONTEND=noninteractive dpkg-reconfigure dash
		ENV_CONFIG
	
		# ----- VM1:provision:TEST_CONFIG -----
		config.vm.provision "shell", 
		run: "never",
		inline: <<-TEST_CONFIG
			echo "*** only for test ***"
		TEST_CONFIG
	
		# ----- VM1:provision:NET_CONFIG -----
		config.vm.provision "shell", 
		#run: "always", 
		run: "never",
		inline: <<-NET_CONFIG
			echo "*** customize network config ***"
			#route del default gw 10.0.2.2
			#route add default gw 10.5.161.254
		NET_CONFIG
	
		# ----- VM1:provision:CONFIG_VPN -----
		config.vm.provision "shell", 
		#run: "never",
		inline: <<-CONFIG_VPN
			echo "*** setup VPN client ***"
			DEBIAN_FRONTEND=noninteractive apt-get install -y -qq openswan xl2tpd ppp
			
			config_file="/etc/ipsec.conf"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	# /etc/ipsec.conf - Openswan IPsec configuration file
	
	version	2.0
	
	config setup
		dumpdir=/var/run/pluto/
		nat_traversal=yes
		virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v4:25.0.0.0/8,%v6:fd00::/8,%v6:fe80::/10
		oe=off
		protostack=netkey
		#plutoopts="--interface eth1"
		#plutostderrlog=/var/log/ipsec.log
		#force_keepalive=yes
		#keep_alive=60
	
	conn %default
		auto=add
		authby=secret
		pfs=no
		keyingtries=3
		ikelifetime=8h
		keylife=1h
		dpddelay=10
		dpdtimeout=20
		dpdaction=clear
	
	conn #{ipsec_conn_name}
		#type=tunnel
		type=transport
		ike=aes256-sha1,aes128-sha1,3des-sha1
		phase2alg=aes256-sha1,aes128-sha1,3des-sha1
		#left=192.168.90.123
		left=%defaultroute
		leftprotoport=17/1701
		right=#{vpn_server_ip}
		#rightprotoport=17/%any
		rightprotoport=17/1701
	EOF
	
			config_file="/etc/ipsec.secrets"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	#%any #{vpn_server_ip}: PSK "#{vpn_ipsec_psk}"
	%any %any: PSK "#{vpn_ipsec_psk}"
	EOF
	
			config_file="/etc/xl2tpd/xl2tpd.conf"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	[global]
	ipsec saref = no
	; saref refinfo = 30
	; debug avp = yes
	; debug network = yes
	; debug state = yes
	; debug tunnel = yes
	
	[lac #{vpn_lac_name}]
	lns = #{vpn_server_ip}
	refuse chap = yes
	refuse pap = yes
	require authentication = yes
	name = vpn-server
	; ppp debug = yes
	pppoptfile = /etc/ppp/options.l2tpd.client
	length bit = yes
	
	EOF
	
			config_file="/etc/ppp/options.l2tpd.client"
			if [ -f $config_file ] && [ ! -f ${config_file}.BAK ]; then
				cp $config_file ${config_file}.BAK
			fi
			cat <<EOF >$config_file
	ipcp-accept-local
	ipcp-accept-remote
	refuse-eap
	require-mschap-v2
	noccp
	noauth
	idle 1800
	mtu 1410
	mru 1410
	defaultroute
	usepeerdns
	debug
	lock
	connect-delay 5000
	name #{l2tpd_ppp_user}
	password #{l2tpd_ppp_passwd}
	EOF
		CONFIG_VPN
	
		# ----- VM1:provision:CONFIG_VPN_START -----
		config.vm.provision "shell", 
		run: "always",
		inline: <<-CONFIG_VPN_START
		/etc/init.d/ipsec restart
		/etc/init.d/xl2tpd restart
		sleep 3
		echo "c #{vpn_lac_name}" > /var/run/xl2tpd/l2tp-control
		sleep 3
		ipsec auto --add #{ipsec_conn_name}
		ipsec auto --up #{ipsec_conn_name}
		CONFIG_VPN_START
		
		# ----- VM1:provision:CONFIG_DONE -----
		config.vm.provision "shell", 
		run: "always", 
		inline: <<-CONFIG_DONE
			echo ""
			echo "*** eth1 ip address: $(ifconfig eth1 | awk '/inet addr/{print substr($2,6)}') ***"
			echo "*** ppp0 ip address: $(ifconfig ppp0 | awk '/inet addr/{print substr($2,6)}') ***"
			echo "*** try to ping ppp0 gateway ip on VPN client and do tcpdump for route interface on VPN server $(ifconfig ppp0 | awk '/P-t-P/{print substr($3,7)}') ***"
			echo ""
			ping -c 5 $(ifconfig ppp0 | awk '/P-t-P/{print substr($3,7)}')
		CONFIG_DONE
	
		config.vm.post_up_message = <<-MESSAGE
	*** config.vm.was started :-) ***
		MESSAGE
	  end
	  # ----- END VM1 -----
	end


## step4 - start up a vpn-client


	[2018-12-25 16:33.53]  /drives/c/home/ws/VMs/vpn-client
	[andrew.andrew-NB] ➤ vagrant.exe up
	Bringing machine 'vpn-client' up with 'virtualbox' provider...
	==> vpn-client: Importing base box 'trusty32'...
	==> vpn-client: Matching MAC address for NAT networking...
	==> vpn-client: Setting the name of the VM: vbox-vpn-client
	==> vpn-client: Clearing any previously set forwarded ports...
	==> vpn-client: Clearing any previously set network interfaces...
	==> vpn-client: Preparing network interfaces based on configuration...
	    vpn-client: Adapter 1: nat
	    vpn-client: Adapter 2: bridged
	==> vpn-client: Forwarding ports...
	    vpn-client: 22 (guest) => 2203 (host) (adapter 1)
	==> vpn-client: Running 'pre-boot' VM customizations...
	==> vpn-client: Booting VM...
	==> vpn-client: Waiting for machine to boot. This may take a few minutes...
	    vpn-client: SSH address: 127.0.0.1:2203
	    vpn-client: SSH username: vagrant
	    vpn-client: SSH auth method: password
	    vpn-client: Warning: Authentication failure. Retrying...
	==> vpn-client: Machine booted and ready!
	==> vpn-client: Checking for guest additions in VM...
	    vpn-client: The guest additions on this VM do not match the installed version of
	    vpn-client: VirtualBox! In most cases this is fine, but in rare cases it can
	    vpn-client: prevent things such as shared folders from working properly. If you see
	    vpn-client: shared folder errors, please make sure the guest additions within the
	    vpn-client: virtual machine match the version of VirtualBox you have installed on
	    vpn-client: your host and reload your VM.
	    vpn-client:
	    vpn-client: Guest Additions Version: 4.3.36
	    vpn-client: VirtualBox Version: 5.2
	==> vpn-client: Setting hostname...
	==> vpn-client: Configuring and enabling network interfaces...
	==> vpn-client: Mounting shared folders...
	    vpn-client: /vagrant => C:/home/ws/VMs/vpn-client
	==> vpn-client: Running provisioner: shell...
	    vpn-client: Running: inline script
	    vpn-client: *** update apt-repository ***
	    vpn-client: *** installing useful pkgs ***
	    vpn-client: Selecting previously unselected package liberror-perl.
	    vpn-client: (Reading database ... 63230 files and directories currently installed.)
	    vpn-client: Preparing to unpack .../liberror-perl_0.17-1.1_all.deb ...
	    vpn-client: Unpacking liberror-perl (0.17-1.1) ...
	    vpn-client: Selecting previously unselected package git-man.
	    vpn-client: Preparing to unpack .../git-man_1%3a1.9.1-1ubuntu0.10_all.deb ...
	    vpn-client: Unpacking git-man (1:1.9.1-1ubuntu0.10) ...
	    vpn-client: Selecting previously unselected package git.
	    vpn-client: Preparing to unpack .../git_1%3a1.9.1-1ubuntu0.10_i386.deb ...
	    vpn-client: Unpacking git (1:1.9.1-1ubuntu0.10) ...
	    vpn-client: Selecting previously unselected package lftp.
	    vpn-client: Preparing to unpack .../lftp_4.4.13-1ubuntu0.1_i386.deb ...
	    vpn-client: Unpacking lftp (4.4.13-1ubuntu0.1) ...
	    vpn-client: Selecting previously unselected package tree.
	    vpn-client: Preparing to unpack .../archives/tree_1.6.0-1_i386.deb ...
	    vpn-client: Unpacking tree (1.6.0-1) ...
	    vpn-client: Selecting previously unselected package unzip.
	    vpn-client: Preparing to unpack .../unzip_6.0-9ubuntu1.5_i386.deb ...
	    vpn-client: Unpacking unzip (6.0-9ubuntu1.5) ...
	    vpn-client: Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
	    vpn-client: Processing triggers for mime-support (3.54ubuntu1.1) ...
	    vpn-client: Setting up liberror-perl (0.17-1.1) ...
	    vpn-client: Setting up git-man (1:1.9.1-1ubuntu0.10) ...
	    vpn-client: Setting up git (1:1.9.1-1ubuntu0.10) ...
	    vpn-client: Setting up lftp (4.4.13-1ubuntu0.1) ...
	    vpn-client: Setting up tree (1.6.0-1) ...
	    vpn-client: Setting up unzip (6.0-9ubuntu1.5) ...
	==> vpn-client: Running provisioner: shell...
	    vpn-client: Running: inline script
	    vpn-client: *** config timezone ***
	    vpn-client: *** linking /bin/sh to bash instead of dash ***
	    vpn-client: Removing 'diversion of /bin/sh to /bin/sh.distrib by dash'
	    vpn-client: Adding 'diversion of /bin/sh to /bin/sh.distrib by bash'
	    vpn-client: Removing 'diversion of /usr/share/man/man1/sh.1.gz to /usr/share/man/man1/sh.distrib.1.gz by dash'
	    vpn-client: Adding 'diversion of /usr/share/man/man1/sh.1.gz to /usr/share/man/man1/sh.distrib.1.gz by bash'
	==> vpn-client: Running provisioner: shell...
	    vpn-client: Running: inline script
	    vpn-client: *** setup VPN client ***
	    vpn-client: Preconfiguring packages ...
	    vpn-client: Selecting previously unselected package iproute.
	    vpn-client: (Reading database ... 64026 files and directories currently installed.)
	    vpn-client: Preparing to unpack .../iproute_1%3a3.12.0-2ubuntu1.2_all.deb ...
	    vpn-client: Unpacking iproute (1:3.12.0-2ubuntu1.2) ...
	    vpn-client: Selecting previously unselected package openswan.
	    vpn-client: Preparing to unpack .../openswan_1%3a2.6.38-1_i386.deb ...
	    vpn-client: Unpacking openswan (1:2.6.38-1) ...
	    vpn-client: Preparing to unpack .../ppp_2.4.5-5.1ubuntu2.3_i386.deb ...
	    vpn-client: Unpacking ppp (2.4.5-5.1ubuntu2.3) over (2.4.5-5.1ubuntu2.2) ...
	    vpn-client: Selecting previously unselected package xl2tpd.
	    vpn-client: Preparing to unpack .../xl2tpd_1.3.6+dfsg-1_i386.deb ...
	    vpn-client: Unpacking xl2tpd (1.3.6+dfsg-1) ...
	    vpn-client: Processing triggers for ureadahead (0.100.0-16) ...
	    vpn-client: Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
	    vpn-client: Setting up iproute (1:3.12.0-2ubuntu1.2) ...
	    vpn-client: Setting up openswan (1:2.6.38-1) ...
	    vpn-client: ipsec_setup: Starting Openswan IPsec 2.6.38...
	    vpn-client: ipsec_setup: No KLIPS support found while requested, desperately falling back to netkey
	    vpn-client: ipsec_setup: NETKEY support found. Use protostack=netkey in /etc/ipsec.conf to avoid attempts to use KLIPS. Attempting to continue with NETKEY
	    vpn-client: Setting up ppp (2.4.5-5.1ubuntu2.3) ...
	    vpn-client: Setting up xl2tpd (1.3.6+dfsg-1) ...
	    vpn-client: Starting xl2tpd:
	    vpn-client: xl2tpd.
	    vpn-client: Processing triggers for ureadahead (0.100.0-16) ...
	==> vpn-client: Running provisioner: shell...
	    vpn-client: Running: inline script
	    vpn-client: ipsec_setup: Stopping Openswan IPsec...
	    vpn-client: ipsec_setup: Starting Openswan IPsec U2.6.38/K3.13.0-157-generic...
	    vpn-client: Restarting xl2tpd:
	    vpn-client: xl2tpd.
	    vpn-client: 104 "L2TP-PSK" #1: STATE_MAIN_I1: initiate
	    vpn-client: 003 "L2TP-PSK" #1: received Vendor ID payload [Openswan (this version) 2.6.38 ]
	    vpn-client: 003 "L2TP-PSK" #1: received Vendor ID payload [Dead Peer Detection]
	    vpn-client: 003 "L2TP-PSK" #1: received Vendor ID payload [RFC 3947] method set to=115
	    vpn-client: 106 "L2TP-PSK" #1: STATE_MAIN_I2: sent MI2, expecting MR2
	    vpn-client: 003 "L2TP-PSK" #1: NAT-Traversal: Result using draft-ietf-ipsec-nat-t-ike (MacOS X): no NAT detected
	    vpn-client: 108 "L2TP-PSK" #1: STATE_MAIN_I3: sent MI3, expecting MR3
	    vpn-client: 003 "L2TP-PSK" #1: received Vendor ID payload [CAN-IKEv2]
	    vpn-client: 004 "L2TP-PSK" #1: STATE_MAIN_I4: ISAKMP SA established {auth=OAKLEY_PRESHARED_KEY cipher=aes_256 prf=oakley_sha group=modp1536}
	    vpn-client: 117 "L2TP-PSK" #2: STATE_QUICK_I1: initiate
	    vpn-client: 004 "L2TP-PSK" #2: STATE_QUICK_I2: sent QI2, IPsec SA established transport mode {ESP=>0x8a464c90 <0xdc2266da xfrm=AES_256-HMAC_SHA1 NATOA=none NATD=none DPD=enabled}
	==> vpn-client: Running provisioner: shell...
	    vpn-client: Running: inline script
	    vpn-client: *** eth1 ip address: 192.168.90.147 ***
	    vpn-client: *** ppp0 ip address: 172.16.1.100 ***
	    vpn-client: *** try to ping ppp0 gateway ip on VPN client and do tcpdump for route interface on VPN server 172.16.1.1 ***
	    vpn-client: PING 172.16.1.1 (172.16.1.1) 56(84) bytes of data.
	    vpn-client: 64 bytes from 172.16.1.1: icmp_seq=1 ttl=64 time=0.832 ms
	    vpn-client: 64 bytes from 172.16.1.1: icmp_seq=2 ttl=64 time=1.39 ms
	    vpn-client: 64 bytes from 172.16.1.1: icmp_seq=3 ttl=64 time=1.07 ms
	    vpn-client: 64 bytes from 172.16.1.1: icmp_seq=4 ttl=64 time=1.11 ms
	    vpn-client: 64 bytes from 172.16.1.1: icmp_seq=5 ttl=64 time=1.85 ms
	    vpn-client: --- 172.16.1.1 ping statistics ---
	    vpn-client: 5 packets transmitted, 5 received, 0% packet loss, time 4011ms
	    vpn-client: rtt min/avg/max/mdev = 0.832/1.253/1.854/0.351 ms
	
	==> vpn-client: Machine 'vpn-client' has a post `vagrant up` message. This is a message
	==> vpn-client: from the creator of the Vagrantfile, and not from Vagrant itself:
	==> vpn-client:
	==> vpn-client: *** config.vm.was started :-) ***
	                                                                                                                                                                             ✔
	──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
	[2018-12-25 16:36.36]  /drives/c/home/ws/VMs/vpn-client
	[andrew.andrew-NB] ➤


## step5 - verify the network


### login to vpn-server


You can see the ppp0 interface:

	vagrant@vpn-server:~$ sudo bash
	root@vpn-server:~# ifconfig 
	eth0      Link encap:Ethernet  HWaddr 08:00:27:ba:fa:96  
	          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
	          inet6 addr: fe80::a00:27ff:feba:fa96/64 Scope:Link
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:1513 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:1128 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000 
	          RX bytes:155941 (155.9 KB)  TX bytes:158284 (158.2 KB)
	
	eth1      Link encap:Ethernet  HWaddr 08:00:27:23:78:31  
	          inet addr:192.168.90.146  Bcast:192.168.90.255  Mask:255.255.255.0
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:12981 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:7530 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000 
	          RX bytes:19386308 (19.3 MB)  TX bytes:608241 (608.2 KB)
	
	lo        Link encap:Local Loopback  
	          inet addr:127.0.0.1  Mask:255.0.0.0
	          inet6 addr: ::1/128 Scope:Host
	          UP LOOPBACK RUNNING  MTU:65536  Metric:1
	          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0 
	          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
	
	ppp0      Link encap:Point-to-Point Protocol  
	          inet addr:172.16.1.1  P-t-P:172.16.1.100  Mask:255.255.255.255
	          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1000  Metric:1
	          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:9 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:3 
	          RX bytes:492 (492.0 B)  TX bytes:501 (501.0 B)


ipsec verify

	root@vpn-server:~# ipsec verify
	Checking your system to see if IPsec got installed and started correctly:
	Version check and ipsec on-path                             	[OK]
	Linux Openswan U2.6.38/K3.13.0-157-generic (netkey)
	Checking for IPsec support in kernel                        	[OK]
	 SAref kernel support                                       	[N/A]
	 NETKEY:  Testing XFRM related proc values                  	[OK]
		[OK]
		[OK]
	Checking that pluto is running                              	[OK]
	 Pluto listening for IKE on udp 500                         	[OK]
	 Pluto listening for NAT-T on udp 4500                      	[OK]
	Two or more interfaces found, checking IP forwarding        	[FAILED]
	Checking NAT and MASQUERADEing                              	[OK]
	Checking for 'ip' command                                   	[OK]
	Checking /bin/sh is not /bin/dash                           	[OK]
	Checking for 'iptables' command                             	[OK]
	Opportunistic Encryption Support                            	[DISABLED]


check ipsec status:

	root@vpn-server:~# ipsec auto --status 
	000 using kernel interface: netkey
	000 interface lo/lo ::1
	000 interface lo/lo 127.0.0.1
	000 interface lo/lo 127.0.0.1
	000 interface eth0/eth0 10.0.2.15
	000 interface eth0/eth0 10.0.2.15
	000 interface eth1/eth1 192.168.90.146
	000 interface eth1/eth1 192.168.90.146
	000 %myid = (none)
	000 debug none
	000  
	000 virtual_private (%priv):
	000 - allowed 6 subnets: 10.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, 25.0.0.0/8, fd00::/8, fe80::/10
	000 - disallowed 0 subnets: 
	000 WARNING: Disallowed subnets in virtual_private= is empty. If you have 
	000          private address space in internal use, it should be excluded!
	000  
	000 algorithm ESP encrypt: id=2, name=ESP_DES, ivlen=8, keysizemin=64, keysizemax=64
	000 algorithm ESP encrypt: id=3, name=ESP_3DES, ivlen=8, keysizemin=192, keysizemax=192
	000 algorithm ESP encrypt: id=6, name=ESP_CAST, ivlen=8, keysizemin=40, keysizemax=128
	000 algorithm ESP encrypt: id=7, name=ESP_BLOWFISH, ivlen=8, keysizemin=40, keysizemax=448
	000 algorithm ESP encrypt: id=11, name=ESP_NULL, ivlen=0, keysizemin=0, keysizemax=0
	000 algorithm ESP encrypt: id=12, name=ESP_AES, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=13, name=ESP_AES_CTR, ivlen=8, keysizemin=160, keysizemax=288
	000 algorithm ESP encrypt: id=14, name=ESP_AES_CCM_A, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=15, name=ESP_AES_CCM_B, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=16, name=ESP_AES_CCM_C, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=18, name=ESP_AES_GCM_A, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=19, name=ESP_AES_GCM_B, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=20, name=ESP_AES_GCM_C, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=22, name=ESP_CAMELLIA, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=252, name=ESP_SERPENT, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP encrypt: id=253, name=ESP_TWOFISH, ivlen=8, keysizemin=128, keysizemax=256
	000 algorithm ESP auth attr: id=1, name=AUTH_ALGORITHM_HMAC_MD5, keysizemin=128, keysizemax=128
	000 algorithm ESP auth attr: id=2, name=AUTH_ALGORITHM_HMAC_SHA1, keysizemin=160, keysizemax=160
	000 algorithm ESP auth attr: id=5, name=AUTH_ALGORITHM_HMAC_SHA2_256, keysizemin=256, keysizemax=256
	000 algorithm ESP auth attr: id=6, name=AUTH_ALGORITHM_HMAC_SHA2_384, keysizemin=384, keysizemax=384
	000 algorithm ESP auth attr: id=7, name=AUTH_ALGORITHM_HMAC_SHA2_512, keysizemin=512, keysizemax=512
	000 algorithm ESP auth attr: id=8, name=AUTH_ALGORITHM_HMAC_RIPEMD, keysizemin=160, keysizemax=160
	000 algorithm ESP auth attr: id=9, name=AUTH_ALGORITHM_AES_CBC, keysizemin=128, keysizemax=128
	000 algorithm ESP auth attr: id=251, name=AUTH_ALGORITHM_NULL_KAME, keysizemin=0, keysizemax=0
	000  
	000 algorithm IKE encrypt: id=0, name=(null), blocksize=16, keydeflen=131
	000 algorithm IKE encrypt: id=5, name=OAKLEY_3DES_CBC, blocksize=8, keydeflen=192
	000 algorithm IKE encrypt: id=7, name=OAKLEY_AES_CBC, blocksize=16, keydeflen=128
	000 algorithm IKE hash: id=1, name=OAKLEY_MD5, hashsize=16
	000 algorithm IKE hash: id=2, name=OAKLEY_SHA1, hashsize=20
	000 algorithm IKE hash: id=4, name=OAKLEY_SHA2_256, hashsize=32
	000 algorithm IKE hash: id=6, name=OAKLEY_SHA2_512, hashsize=64
	000 algorithm IKE dh group: id=2, name=OAKLEY_GROUP_MODP1024, bits=1024
	000 algorithm IKE dh group: id=5, name=OAKLEY_GROUP_MODP1536, bits=1536
	000 algorithm IKE dh group: id=14, name=OAKLEY_GROUP_MODP2048, bits=2048
	000 algorithm IKE dh group: id=15, name=OAKLEY_GROUP_MODP3072, bits=3072
	000 algorithm IKE dh group: id=16, name=OAKLEY_GROUP_MODP4096, bits=4096
	000 algorithm IKE dh group: id=17, name=OAKLEY_GROUP_MODP6144, bits=6144
	000 algorithm IKE dh group: id=18, name=OAKLEY_GROUP_MODP8192, bits=8192
	000 algorithm IKE dh group: id=22, name=OAKLEY_GROUP_DH22, bits=1024
	000 algorithm IKE dh group: id=23, name=OAKLEY_GROUP_DH23, bits=2048
	000 algorithm IKE dh group: id=24, name=OAKLEY_GROUP_DH24, bits=2048
	000  
	000 stats db_ops: {curr_cnt, total_cnt, maxsz} :context={0,0,0} trans={0,0,0} attrs={0,0,0} 
	000  
	000 "L2TP-PSK-noNAT": 192.168.90.146:17/1701...%any:17/%any; unrouted; eroute owner: #0
	000 "L2TP-PSK-noNAT":     myip=unset; hisip=unset;
	000 "L2TP-PSK-noNAT":   ike_life: 28800s; ipsec_life: 3600s; rekey_margin: 540s; rekey_fuzz: 100%; keyingtries: 3 
	000 "L2TP-PSK-noNAT":   policy: PSK+ENCRYPT+IKEv2ALLOW+SAREFTRACK+lKOD+rKOD; prio: 32,32; interface: eth1; 
	000 "L2TP-PSK-noNAT":   dpd: action:clear; delay:10; timeout:20;  
	000 "L2TP-PSK-noNAT":   newest ISAKMP SA: #0; newest IPsec SA: #0; 
	000 "L2TP-PSK-noNAT":   IKE algorithms wanted: AES_CBC(7)_256-SHA1(2)_000-MODP1536(5), AES_CBC(7)_256-SHA1(2)_000-MODP1024(2), AES_CBC(7)_128-SHA1(2)_000-MODP1536(5), AES_CBC(7)_128-SHA1(2)_000-MODP1024(2), 3DES_CBC(5)_000-SHA1(2)_000-MODP1536(5), 3DES_CBC(5)_000-SHA1(2)_000-MODP1024(2); flags=-strict
	000 "L2TP-PSK-noNAT":   IKE algorithms found:  AES_CBC(7)_256-SHA1(2)_160-MODP1536(5)AES_CBC(7)_256-SHA1(2)_160-MODP1024(2)AES_CBC(7)_128-SHA1(2)_160-MODP1536(5)AES_CBC(7)_128-SHA1(2)_160-MODP1024(2)3DES_CBC(5)_192-SHA1(2)_160-MODP1536(5)3DES_CBC(5)_192-SHA1(2)_160-MODP1024(2)
	000 "L2TP-PSK-noNAT":   ESP algorithms wanted: AES(12)_256-SHA1(2)_000, AES(12)_128-SHA1(2)_000, 3DES(3)_000-SHA1(2)_000; flags=-strict
	000 "L2TP-PSK-noNAT":   ESP algorithms loaded: AES(12)_256-SHA1(2)_160, AES(12)_128-SHA1(2)_160, 3DES(3)_192-SHA1(2)_160
	000 "L2TP-PSK-noNAT"[1]: 192.168.90.146:17/1701...192.168.90.147:17/1701; erouted; eroute owner: #2
	000 "L2TP-PSK-noNAT"[1]:     myip=unset; hisip=unset;
	000 "L2TP-PSK-noNAT"[1]:   ike_life: 28800s; ipsec_life: 3600s; rekey_margin: 540s; rekey_fuzz: 100%; keyingtries: 3 
	000 "L2TP-PSK-noNAT"[1]:   policy: PSK+ENCRYPT+IKEv2ALLOW+SAREFTRACK+lKOD+rKOD; prio: 32,32; interface: eth1; 
	000 "L2TP-PSK-noNAT"[1]:   dpd: action:clear; delay:10; timeout:20;  
	000 "L2TP-PSK-noNAT"[1]:   newest ISAKMP SA: #1; newest IPsec SA: #2; 
	000 "L2TP-PSK-noNAT"[1]:   IKE algorithms wanted: AES_CBC(7)_256-SHA1(2)_000-MODP1536(5), AES_CBC(7)_256-SHA1(2)_000-MODP1024(2), AES_CBC(7)_128-SHA1(2)_000-MODP1536(5), AES_CBC(7)_128-SHA1(2)_000-MODP1024(2), 3DES_CBC(5)_000-SHA1(2)_000-MODP1536(5), 3DES_CBC(5)_000-SHA1(2)_000-MODP1024(2); flags=-strict
	000 "L2TP-PSK-noNAT"[1]:   IKE algorithms found:  AES_CBC(7)_256-SHA1(2)_160-MODP1536(5)AES_CBC(7)_256-SHA1(2)_160-MODP1024(2)AES_CBC(7)_128-SHA1(2)_160-MODP1536(5)AES_CBC(7)_128-SHA1(2)_160-MODP1024(2)3DES_CBC(5)_192-SHA1(2)_160-MODP1536(5)3DES_CBC(5)_192-SHA1(2)_160-MODP1024(2)
	000 "L2TP-PSK-noNAT"[1]:   IKE algorithm newest: AES_CBC_256-SHA1-MODP1536
	000 "L2TP-PSK-noNAT"[1]:   ESP algorithms wanted: AES(12)_256-SHA1(2)_000, AES(12)_128-SHA1(2)_000, 3DES(3)_000-SHA1(2)_000; flags=-strict
	000 "L2TP-PSK-noNAT"[1]:   ESP algorithms loaded: AES(12)_256-SHA1(2)_160, AES(12)_128-SHA1(2)_160, 3DES(3)_192-SHA1(2)_160
	000 "L2TP-PSK-noNAT"[1]:   ESP algorithm newest: AES_256-HMAC_SHA1; pfsgroup=<N/A>
	000  
	000 #2: "L2TP-PSK-noNAT"[1] 192.168.90.147:500 STATE_QUICK_R2 (IPsec SA established); EVENT_SA_REPLACE in 3198s; newest IPSEC; eroute owner; isakmp#1; idle; import:not set
	000 #2: "L2TP-PSK-noNAT"[1] 192.168.90.147 esp.dc2266da@192.168.90.147 esp.8a464c90@192.168.90.146 ref=0 refhim=4294901761
	000 #1: "L2TP-PSK-noNAT"[1] 192.168.90.147:500 STATE_MAIN_R3 (sent MR3, ISAKMP SA established); EVENT_SA_REPLACE in 28398s; newest ISAKMP; lastdpd=2s(seq in:4958 out:0); idle; import:not set
	000  

192.168.90.147 should be vpn-client ip address. Use tcpdump

	root@vpn-server:~# tcpdump -i eth1 host 192.168.90.147
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
	16:41:34.233718 ARP, Request who-has vpn-server tell 192.168.90.147, length 46
	16:41:34.233739 ARP, Reply vpn-server is-at 08:00:27:23:78:31 (oui Unknown), length 28


ssh to vpn-client:

	vagrant@vpn-client:~$ sudo bash
	root@vpn-client:~# ifconfig 
	eth0      Link encap:Ethernet  HWaddr 08:00:27:ba:fa:96  
	          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
	          inet6 addr: fe80::a00:27ff:feba:fa96/64 Scope:Link
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:1133 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:861 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000 
	          RX bytes:125725 (125.7 KB)  TX bytes:114644 (114.6 KB)
	
	eth1      Link encap:Ethernet  HWaddr 08:00:27:41:89:65  
	          inet addr:192.168.90.147  Bcast:192.168.90.255  Mask:255.255.255.0
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:13052 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:6642 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000 
	          RX bytes:19435379 (19.4 MB)  TX bytes:491820 (491.8 KB)
	
	lo        Link encap:Local Loopback  
	          inet addr:127.0.0.1  Mask:255.0.0.0
	          inet6 addr: ::1/128 Scope:Host
	          UP LOOPBACK RUNNING  MTU:65536  Metric:1
	          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0 
	          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
	
	ppp0      Link encap:Point-to-Point Protocol  
	          inet addr:172.16.1.100  P-t-P:172.16.1.1  Mask:255.255.255.255
	          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1000  Metric:1
	          RX packets:9 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:3 
	          RX bytes:501 (501.0 B)  TX bytes:492 (492.0 B)

Try to ping ppp0 gateway ip (172.16.1.1) and monitoring the tchdump result in vpn-server:


(ping in vpn-client)

	root@vpn-client:~# ping 172.16.1.1
	PING 172.16.1.1 (172.16.1.1) 56(84) bytes of data.
	64 bytes from 172.16.1.1: icmp_seq=1 ttl=64 time=6.44 ms
	64 bytes from 172.16.1.1: icmp_seq=2 ttl=64 time=2.75 ms
	64 bytes from 172.16.1.1: icmp_seq=3 ttl=64 time=1.33 ms
	64 bytes from 172.16.1.1: icmp_seq=4 ttl=64 time=1.44 ms
	^C

(tchdump in vpn-server)

	16:44:58.353777 IP 192.168.90.147 > vpn-server: ESP(spi=0x8a464c90,seq=0x36), length 148
	16:44:58.355441 IP vpn-server > 192.168.90.147: ESP(spi=0xdc2266da,seq=0x36), length 148
	16:44:59.355062 IP 192.168.90.147 > vpn-server: ESP(spi=0x8a464c90,seq=0x37), length 148
	16:44:59.356129 IP vpn-server > 192.168.90.147: ESP(spi=0xdc2266da,seq=0x37), length 148
	16:44:59.459356 IP 192.168.90.147 > vpn-server: ESP(spi=0x8a464c90,seq=0x38), length 68
	16:44:59.460297 IP vpn-server > 192.168.90.147: ESP(spi=0xdc2266da,seq=0x38), length 68
	16:44:59.460631 IP vpn-server > 192.168.90.147: ESP(spi=0xdc2266da,seq=0x39), length 68
	16:44:59.463565 IP 192.168.90.147 > vpn-server: ESP(spi=0x8a464c90,seq=0x39), length 68
	16:45:00.355909 IP 192.168.90.147 > vpn-server: ESP(spi=0x8a464c90,seq=0x3a), length 148
	16:45:00.356407 IP vpn-server > 192.168.90.147: ESP(spi=0xdc2266da,seq=0x3a), length 148
	16:45:01.356999 IP 192.168.90.147 > vpn-server: ESP(spi=0x8a464c90,seq=0x3b), length 148
	16:45:01.357388 IP vpn-server > 192.168.90.147: ESP(spi=0xdc2266da,seq=0x3b), length 148


ESP means `Encapsulating Security Payload`. It means the connection is protecting by IPsec.


## step6 - verify the network (part2)

Stop ipsec and ping again


(ping in vpn-client)

	root@vpn-client:~# service ipsec stop
	ipsec_setup: Stopping Openswan IPsec...
	root@vpn-client:~#  
	root@vpn-client:~# ping 172.16.1.1
	PING 172.16.1.1 (172.16.1.1) 56(84) bytes of data.
	64 bytes from 172.16.1.1: icmp_seq=1 ttl=64 time=5.72 ms
	64 bytes from 172.16.1.1: icmp_seq=2 ttl=64 time=2.15 ms
	64 bytes from 172.16.1.1: icmp_seq=3 ttl=64 time=3.09 ms
	64 bytes from 172.16.1.1: icmp_seq=4 ttl=64 time=1.28 ms
	^C
	--- 172.16.1.1 ping statistics ---
	4 packets transmitted, 4 received, 0% packet loss, time 3012ms
	rtt min/avg/max/mdev = 1.285/3.063/5.720/1.663 ms

(tchdump in vpn-server)

	16:51:42.842625 IP 192.168.90.147.l2f > vpn-server.l2f:  l2tp:[L](36331/22659) {IP 172.16.1.100 > vpn-server: ICMP echo request, id 7338, seq 1, length 64}
	16:51:42.844088 IP vpn-server.l2f > 192.168.90.147.l2f:  l2tp:[L](34678/38517) {IP vpn-server > 172.16.1.100: ICMP echo reply, id 7338, seq 1, length 64}
	16:51:43.844181 IP 192.168.90.147.l2f > vpn-server.l2f:  l2tp:[L](36331/22659) {IP 172.16.1.100 > vpn-server: ICMP echo request, id 7338, seq 2, length 64}
	16:51:43.845244 IP vpn-server.l2f > 192.168.90.147.l2f:  l2tp:[L](34678/38517) {IP vpn-server > 172.16.1.100: ICMP echo reply, id 7338, seq 2, length 64}
	16:51:44.845657 IP 192.168.90.147.l2f > vpn-server.l2f:  l2tp:[L](36331/22659) {IP 172.16.1.100 > vpn-server: ICMP echo request, id 7338, seq 3, length 64}
	16:51:44.846493 IP vpn-server.l2f > 192.168.90.147.l2f:  l2tp:[L](34678/38517) {IP vpn-server > 172.16.1.100: ICMP echo reply, id 7338, seq 3, length 64}
	16:51:45.853042 IP 192.168.90.147.l2f > vpn-server.l2f:  l2tp:[L](36331/22659) {IP 172.16.1.100 > vpn-server: ICMP echo request, id 7338, seq 4, length 64}
	16:51:45.853404 IP vpn-server.l2f > 192.168.90.147.l2f:  l2tp:[L](34678/38517) {IP vpn-server > 172.16.1.100: ICMP echo reply, id 7338, seq 4, length 64}


There is no ESP packet. Now, restart IPsec again.

(ping in vpn-client)

	root@vpn-client:~# service ipsec start
	ipsec_setup: Openswan IPsec apparently already active, start aborted

	root@vpn-client:~# ipsec auto --up L2TP-PSK
	104 "L2TP-PSK" #1: STATE_MAIN_I1: initiate
	003 "L2TP-PSK" #1: received Vendor ID payload [Openswan (this version) 2.6.38 ]
	003 "L2TP-PSK" #1: received Vendor ID payload [Dead Peer Detection]
	003 "L2TP-PSK" #1: received Vendor ID payload [RFC 3947] method set to=115 
	106 "L2TP-PSK" #1: STATE_MAIN_I2: sent MI2, expecting MR2
	003 "L2TP-PSK" #1: NAT-Traversal: Result using draft-ietf-ipsec-nat-t-ike (MacOS X): no NAT detected
	108 "L2TP-PSK" #1: STATE_MAIN_I3: sent MI3, expecting MR3
	003 "L2TP-PSK" #1: received Vendor ID payload [CAN-IKEv2]
	004 "L2TP-PSK" #1: STATE_MAIN_I4: ISAKMP SA established {auth=OAKLEY_PRESHARED_KEY cipher=aes_256 prf=oakley_sha group=modp1536}
	117 "L2TP-PSK" #2: STATE_QUICK_I1: initiate
	004 "L2TP-PSK" #2: STATE_QUICK_I2: sent QI2, IPsec SA established transport mode {ESP=>0x2d6ed539 <0xedd32e5f xfrm=AES_256-HMAC_SHA1 NATOA=none NATD=none DPD=enabled}

	root@vpn-client:~# ping 172.16.1.1
	PING 172.16.1.1 (172.16.1.1) 56(84) bytes of data.
	64 bytes from 172.16.1.1: icmp_seq=1 ttl=64 time=2.61 ms
	64 bytes from 172.16.1.1: icmp_seq=2 ttl=64 time=1.40 ms
	64 bytes from 172.16.1.1: icmp_seq=3 ttl=64 time=1.06 ms
	64 bytes from 172.16.1.1: icmp_seq=4 ttl=64 time=2.43 ms
	^C
	--- 172.16.1.1 ping statistics ---
	4 packets transmitted, 4 received, 0% packet loss, time 3009ms
	rtt min/avg/max/mdev = 1.064/1.880/2.612/0.660 ms
	root@vpn-client:~# 


(tchdump in vpn-server)

	16:53:59.605118 IP 192.168.90.147 > vpn-server: ESP(spi=0x2d6ed539,seq=0x3), length 68
	16:53:59.605499 IP vpn-server > 192.168.90.147: ESP(spi=0xedd32e5f,seq=0x3), length 68
	16:53:59.605601 IP vpn-server > 192.168.90.147: ESP(spi=0xedd32e5f,seq=0x4), length 68
	16:53:59.607180 IP 192.168.90.147 > vpn-server: ESP(spi=0x2d6ed539,seq=0x4), length 68
	16:53:59.932603 IP 192.168.90.147 > vpn-server: ESP(spi=0x2d6ed539,seq=0x5), length 148
	16:53:59.932910 IP vpn-server > 192.168.90.147: ESP(spi=0xedd32e5f,seq=0x5), length 148
	16:54:00.935163 IP 192.168.90.147 > vpn-server: ESP(spi=0x2d6ed539,seq=0x6), length 148
	16:54:00.935826 IP vpn-server > 192.168.90.147: ESP(spi=0xedd32e5f,seq=0x6), length 148

use ESP packet again :)

~ END ~ 


