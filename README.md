# GitLab-CE

## Prepare System
- Multiple VM Ubuntu version 20.04 on Hyper-V   [set up hyper-v on windows10](https://github.com/EknarongAphiphutthikul/Install-Hyper-V)
- DNS Server  [set up](https://github.com/EknarongAphiphutthikul/Install-Dns-bind9)
- Update Package On Ubuntu 20.04
  ```sh
  sudo apt-get update
  ```
- Show hostname
  ```sh
  hostnamectl
  ```
- Set hostname
  ```sh
  sudo hostnamectl set-hostname gitserver.ake.com
  ```
- Show ip
  ```sh
  ifconfig
  ```
- Set ipv4 internal network (vEthernet-Internal-ME)
  - On cloud : you'll need to disable.
    ```sh
    sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
    ```
    ```console
    network: {config: disabled}
    ```
  - Show file config in netplan
    ```sh
    ls /etc/netplan/
    ```
    ```console
    00-installer-config.yaml
    ```
  - Edit file config in netplan
    ```sh
    sudo nano /etc/netplan/00-installer-config.yaml
    ```
    ```console
    network:
      ethernets:
        eth0:
          dhcp4: false
          addresses:
            -  169.254.19.104/16
          nameservers:
            search: [ ake.com ]
            addresses:
              - 169.254.19.105
        eth1:
          dhcp4: true
      version: 2
    ```
  - Apply Config
    ```sh
    sudo netplan apply
    ```

- Set DNS (change default 127.0.0.53 to 169.254.19.105)  
  > **Important** : Workaround for  [Bug #1624320](https://bugs.launchpad.net/ubuntu/+source/systemd/+bug/1624320)
  ```sh
  sudo rm -f /etc/resolv.conf
  sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
  sudo reboot
  ```
----

<br/>

## Install GitLab-CE
- Install Required package dependencies
  ```sh  
  sudo apt install curl tzdata ca-certificates openssh-server
  ```
- Gitlab package repo
  ```sh
  curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
  ```
- Install
  ```sh
  sudo apt install gitlab-ce
  ```
  ```console
  Reading package lists... Done
  Building dependency tree
  Reading state information... Done
  The following NEW packages will be installed:
    gitlab-ce
  0 upgraded, 1 newly installed, 0 to remove and 94 not upgraded.
  Need to get 918 MB of archives.
  After this operation, 2,482 MB of additional disk space will be used.
  Get:1 https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu focal/main amd64 gitlab-ce amd64 13.12.4-ce.0 [918 MB]
  Fetched 918 MB in 5min 14s (2,926 kB/s)
  Selecting previously unselected package gitlab-ce.
  (Reading database ... 71397 files and directories currently installed.)
  Preparing to unpack .../gitlab-ce_13.12.4-ce.0_amd64.deb ...
  Unpacking gitlab-ce (13.12.4-ce.0) ...
  Setting up gitlab-ce (13.12.4-ce.0) ...
  It looks like GitLab has not been configured yet; skipping the upgrade script.

        *.                  *.
        ***                 ***
      *****               *****
      .******             *******
      ********            ********
    ,,,,,,,,,***********,,,,,,,,,
    ,,,,,,,,,,,*********,,,,,,,,,,,
    .,,,,,,,,,,,*******,,,,,,,,,,,,
        ,,,,,,,,,*****,,,,,,,,,.
          ,,,,,,,****,,,,,,
              .,,,***,,,,
                  ,*,.



      _______ __  __          __
      / ____(_) /_/ /   ____ _/ /_
    / / __/ / __/ /   / __ `/ __ \
    / /_/ / / /_/ /___/ /_/ / /_/ /
    \____/_/\__/_____/\__,_/_.___/


  Thank you for installing GitLab!
  GitLab was unable to detect a valid hostname for your instance.
  Please configure a URL for your GitLab instance by setting `external_url`
  configuration in /etc/gitlab/gitlab.rb file.
  Then, you can start your GitLab instance by running the following command:
    sudo gitlab-ctl reconfigure

  For a comprehensive list of configuration options please see the Omnibus GitLab readme
  https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/README.md

  Help us improve the installation experience, let us know how we did with a 1 minute survey:
  https://gitlab.fra1.qualtrics.com/jfe/form/SV_6kVqZANThUQ1bZb?installation=omnibus&release=13-12
  ```
- Configure Gitlab with SSL/TLS Certificate
  - Generate cert
    ```sh
    sudo mkdir /etc/gitlab/ssl

    cd /etc/gitlab/ssl

    sudo openssl req -newkey rsa:4096 -x509 -sha512 -days 3650 -nodes -out /etc/gitlab/ssl/gitserver-demo.crt -keyout /etc/gitlab/ssl/gitserver-demo.key -subj "/C=TH/ST=Bangkok/L=Phayathai/O=ake-demo/CN=*.ake.com/"

    sudo mkdir /etc/gitlab/trusted-certs

    sudo cp /etc/gitlab/ssl/gitserver-demo.crt /etc/gitlab/trusted-certs/
    ```
    Want to use Let’s Encrypt instead? Check this [link](https://docs.gitlab.com/omnibus/settings/ssl.html#lets-encrypt-integration).

    OR use CA, [click](https://github.com/EknarongAphiphutthikul/OpenSSL-Certificate-Authority)  
    copy gitserver.cert.pem gitserver.key.pem ca-chain.cert.pem to /etc/gitlab/ssl
    ```sh
    ls /etc/gitlab/ssl
    ```
    ```console
    ca-chain.cert.pem  gitserver.cert.pem  gitserver.key.pem
    ```

    ```sh
    # decrypt key
    sudo openssl rsa -in gitserver.key.pem -out gitserver-demo.key -passin pass:changeit

    # merge file crt
    sudo cat gitserver.cert.pem ca-chain.cert.pem > gitserver-demo.crt

    # change file name
    sudo mv ca-chain.cert.pem ca.crt

    # copy file
    sudo mkdir /etc/gitlab/trusted-certs
    sudo cp /etc/gitlab/ssl/gitserver-demo.crt /etc/gitlab/trusted-certs/

    # delete file
    sudo rm gitserver.cert.pem  gitserver.key.pem
    ```

  - Configure a URL for GitLab Server
    ```sh
    sudo nano /etc/gitlab/gitlab.rb
    ```
    ```console
    external_url 'https://gitserver.ake.com'
    ```
  - Enable Gitlab SSL Settings
    ```console
    ################################################################################
    ## GitLab NGINX
    ##! Docs: https://docs.gitlab.com/omnibus/settings/nginx.html
    ################################################################################

    nginx['enable'] = true
    nginx['client_max_body_size'] = '250m'
    nginx['redirect_http_to_https'] = true
    # nginx['redirect_http_to_https_port'] = 80

    ##! Most root CA's are included by default
    # nginx['ssl_client_certificate'] = "/etc/gitlab/ssl/ca.crt"

    ##! enable/disable 2-way SSL client authentication
    # nginx['ssl_verify_client'] = "off"

    ##! if ssl_verify_client on, verification depth in the client certificates chain
    # nginx['ssl_verify_depth'] = "1"

    nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitserver-demo.crt"
    nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitserver-demo.key"
    # nginx['ssl_ciphers'] = "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE->
    # nginx['ssl_prefer_server_ciphers'] = "on"

    ##! **Recommended by: https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
    ##!                   https://cipherli.st/**
    nginx['ssl_protocols'] = "TLSv1.2 TLSv1.3"

    ```
    if use CA must uncomment nginx['ssl_client_certificate'] in /etc/gitlab/gitlab.rb
    ```console
    nginx['ssl_client_certificate'] = "/etc/gitlab/ssl/ca.crt"
    ```
  - Reconfigure GitLab
    ```sh
    sudo gitlab-ctl reconfigure
    ```
  - Check status of GitLab
    ```sh
    sudo gitlab-ctl status
    ```
  - If you need to restart
    ```sh
    sudo gitlab-ctl restart
    ```
  - If you need to start
    ```sh
    sudo gitlab-ctl start
    ```
  - If you need to stop
    ```sh
    sudo gitlab-ctl stop
    ```
  > To start, stop or restart an individual component, eg nginx  
  > ```sh 
  > sudo gitlab-ctl start|stop|restart nginx
  > ```

- Enable Firewall
  ```sh
  sudo ufw allow 80/tcp
  sudo ufw allow 443/tcp
  sudo ufw status
  ```
- Access Web  
  url : https://gitserver.ake.com/  
  
  <br/>

  ![](https://github.com/EknarongAphiphutthikul/Install-GitLab-CE/blob/main/ChangeRootPassword.JPG)

  <br/>
