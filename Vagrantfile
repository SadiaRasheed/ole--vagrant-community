# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "debian/jessie64"
  config.vm.box_version = "8.2.1"

  config.vm.hostname = "community"

  config.vm.define "community" do |community|
  end

  config.vm.provider "virtualbox" do |vb|
    vb.name = "community"
  end

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 5984, host: 5985
  config.vm.network "forwarded_port", guest: 8080, host: 5985

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
    vb.memory = "1234"
  end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", inline: <<-SHELL
    echo "deb http://ftp.de.debian.org/debian jessie-backports main" | sudo tee -a /etc/apt/sources.list
    sudo aptitude update
    sudo aptitude install -y docker.io vim vim-syntax-docker nodejs screen htop npm
    # install docker couchdb
    sudo docker pull klaemo/couchdb
    sudo docker run -d -p 5984:5984 --name bell -v /srv/data/bell:/usr/local/var/lib/couchdb -v /srv/log/bell:/usr/local/var/log/couchdb klaemo/couchdb
    # use crontab to start couchdb on boot
    sudo crontab -l | sudo tee -a mycron
    echo "@reboot sudo docker start bell" | sudo tee -a mycron
    echo "@reboot sudo node /root/server.js" | sudo tee -a mycron
    sudo crontab mycron
    sudo rm mycron
    # fix nodejs
    cd /usr/bin
    sudo ln -s nodejs node
    # install BeLL-Apps
    cd /vagrant
    mkdir -p ole
    cd ole
    git clone https://github.com/open-learning-exchange/BeLL-Apps.git
    git checkout 333-Community_side_integration_of_288
    cd BeLL-Apps
    chmod +x node_modules/.bin/couchapp
    ## check if docker is running
    while ! curl -X GET http://127.0.0.1:5984/_all_dbs ; do
      sleep 1
    done

    ## create databases & push design docs into them
    for database in databases/*.js; do
      curl -X PUT http://127.0.0.1:5984/${database:10:-3}
      ## do in all except communities languages configurations
      case ${database:10:-3} in
        "communities" | "languages" | "configurations" ) ;;
        * ) node_modules/.bin/couchapp push $database http://127.0.0.1:5984/${database:10:-3} ;;
      esac
    done

    ## add bare minimal required data to couchdb for launching bell-apps smoothly
    for filename in init_docs/languages/*.txt; do
      curl -d @$filename -H "Content-Type: application/json" -X POST http://127.0.0.1:5984/languages;
    done
    curl -d @init_docs/ConfigurationsDoc-Community.txt -H "Content-Type: application/json" -X POST http://127.0.0.1:5984/configurations
    curl -d @init_docs/admin.txt -H "Content-Type: application/json" -X POST http://127.0.0.1:5984/members

    ## fix of log file
    curl -X PUT 'http://127.0.0.1:5984/_config/log/file' -d '"/usr/local/var/log/couchdb/couch.log"'

    ## favicon.ico
    wget https://open-learning-exchange.github.io/favicon.ico
    mv favicon.ico /srv/data/bell/.
    #curl -X PUT 'http://127.0.0.1:5984/_config/httpd_global_handlers/favicon.ico' -d '"{couch_httpd_misc_handlers, handle_favicon_req, \"/usr/local/var/lib/couchdb\"}"'
    curl -X PUT 'http://127.0.0.1:5984/_config/httpd_global_handlers/favicon.ico' -d '"{couch_httpd_misc_handlers, handle_favicon_req, \\"/usr/local/var/lib/couchdb\\"}"'

    # add redirect on port 8080
    cd /root
    npm install express
    echo '#!/usr/bin/env node' > server.js
    echo '' >> server.js
    echo "var express = require('express')" >> server.js
    echo 'var PortJack = express()' >> server.js
    echo 'PortJack.get(/^(.+)$/, function(req, res) {' >> server.js
    echo 'var options = {' >> server.js
    echo '"127.0.0.1": "http://127.0.0.1:5984/apps/_design/bell/MyApp/index.html",' >> server.js
    echo '"localhost": "http://localhost:5984/apps/_design/bell/MyApp/index.html"' >> server.js
    echo '}' >> server.js
    echo 'if (options.hasOwnProperty(req.hostname)) {' >> server.js
    echo "res.setHeader('Location', options[req.hostname])" >> server.js
    echo '}' >> server.js
    echo 'else {' >> server.js
    echo "res.setHeader('Location', 'http://ole.org')" >> server.js
    echo '}' >> server.js
    echo 'res.statusCode = 302' >> server.js
    echo 'res.end()' >> server.js
    echo '})' >> server.js
    echo 'PortJack.listen(8080)' >> server.js
    chmod +x server.js
    node /root/server.js &
  SHELL
end
