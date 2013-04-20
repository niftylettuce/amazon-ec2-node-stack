
# Amazon EC2 Node Stack


## Index

* [Amazon EC2 Configuration](#amazon-ec2-configuration)
* [Ubuntu Security Configuration](#ubuntu-security-configuration)
* [Apache Legacy Support](#apache-legacy-support)


## Amazon EC2 Configuration

1. Sign up for Amazon's [free trial](http://aws.amazon.com/free/) for EC2.
2. Import your key pair to the [AWS console](https://console.aws.amazon.com/ec2/home#s=KeyPairs).

    > You need to have a public/private [key pair](https://help.github.com/articles/generating-ssh-keys) generated.

3. Click "Launch Instance" at the [AWS console](https://console.aws.amazon.com/ec2/home#s=Instances).

        * Launch a new "micro" instance from "Quick Start" with 64-bit "Ubuntu Server 12.04.1 LTS".
        * Continue through using the defaults until you reach the "Create Key Pair" step.
        * Select the radio button for "Choose from your existing Key Pairs".
        * From the dropdown menu, you should be able to select the SSH public key you uploaded in step #2.
        * Continue through defaults and launch your instance.

4. Copy the instance's "Public DNS" ("hostname") to your clipboard.

    > (e.g. Public DNS: `ec2-12-345-67-89.compute-1.amazonaws.com`)

5. Load up a new terminal window and paste the hostname for a `ssh` conneciton.

    ```bash
    ssh ubuntu@ec2-12-345-67-89.compute-1.amazonaws.com
    ```

    > Type "yes" when you are the prompt asks, "Are you sure you want to continue connecting?".

    > Once you are successfully connected, the prompt will say hello:

    ```bash
    The authenticity of host 'ec2-12-345-67-89.compute-1.amazonaws.com (12.345.67.89)' can't be established.
    ECDSA key fingerprint is ab:cd:ef:gh:jk:lm:no.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added 'ec2-12-345-67-89.compute-1.amazonaws.com,12.345.67.89' (ECDSA) to the list of known hosts.
    Welcome to Ubuntu 12.04.1 LTS (GNU/Linux 3.2.0-36-virtual x86_64)

     * Documentation:  https://help.ubuntu.com/

      System information as of Sat Mar  2 04:05:42 UTC 2013

      System load:  0.2               Processes:           58
      Usage of /:   10.9% of 7.87GB   Users logged in:     0
      Memory usage: 6%                IP address for eth0: 12.345.67.89
      Swap usage:   0%

      Graph this data and manage this system at https://landscape.canonical.com/

    0 packages can be updated.
    0 updates are security updates.

    Get cloud support with Ubuntu Advantage Cloud Guest
      http://www.ubuntu.com/business/services/cloud

    Use Juju to deploy your cloud instances and workloads.
      https://juju.ubuntu.com/#cloud-precise

    The programs included with the Ubuntu system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
    applicable law.

    To run a command as administrator (user "root"), use "sudo <command>".
    See "man sudo_root" for details.

    ubuntu@domU-12-34-56-78:~$
    ```

6. Set the time based on your local.

    ```bash
    sudo dpkg-reconfigure tzdata
    ```

7. Update and upgrade all existing packages.

    ```bash
    sudo apt-get update && sudo apt-get upgrade
    ```


## Ubuntu Security Configuration

1. Connect over SSH to your EC2 instance (see [Amazon EC2 Configuration](#amazon-ec2-configuration) step #5).
2. Change the port for SSH and disable remote root login:

    ```bash
    sudo vim /etc/ssh/sshd_config
    ```

    > Edit "sshd_config" with the following changes:

    ```diff
    # What ports, IPs and protocols we listen for
    +Port 44444
    -Port 22
    ```

    ```diff
    # Authentication:
    LoginGraceTime 120
    +PermitRootLogin no
    -PermitRootLogin yes
    StrictModes yes
    ```

    ```bash
    sudo service ssh restart
    ```

    > Since you changed the port to 44444, next time you SSH you'll need to connect with the `-p 44444` flag.

3. Setup `fail2ban` to automatically ban malicious IP addresses:

    ```bash
    sudo apt-get install fail2ban
    sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    sudo vim /etc/fail2ban/jail.local
    ```

    > Edit "jail.local" with the following changes:

    ```diff
    [ssh]

    enabled  = true
    +port     = 44444
    -port     = ssh
    filter   = sshd
    logpath  = /var/log/auth.log
    maxretry = 6
    ```

    ```diff
    [ssh-ddos]

    +enabled  = true
    -enabled  = false
    +port     = 44444
    -port     = ssh
    filter   = sshd-ddos
    logpath  = /var/log/auth.log
    maxretry = 6
    ```

    > After saving the file, restart the `fail2ban` process:

    ```bash
    sudo service fail2ban restart
    ```

4. Log in to the EC2 Security Group management <https://console.aws.amazon.com/ec2/home#s=SecurityGroups>

5. Remove inbound port 22 and enable port 44444 for your server's security group.

    > This will serve as our new SSH port (feel free to change port number above/throughout these steps).

6. Enable automatic security updates.

    ```bash
    sudo apt-get install unattended-upgrades
    sudo vim /etc/apt/apt.conf.d/10periodic
    ```

    Edit "/etc/apt/apt.conf.d/10periodic":

    ```diff
    APT::Periodic::Update-Package-Lists "1";
    +APT::Periodic::Download-Upgradeable-Packages "1";
    -APT::Periodic::Download-Upgradeable-Packages "0";
    +APT::Periodic::AutocleanInterval "7";
    -APT::Periodic::AutocleanInterval "0";
    +APT::Periodic::Unattended-Upgrade "1";
    ```

7. Install and configure logwatch to email you the daily logs:

    ```bash
    sudo apt-get install logwatch
    sudo vim /etc/cron.daily/00logwatch
    ```

    ```diff
    #execute
    +/usr/sbin/logwatch --output mail --mailto your@email.com --detail high
    -/usr/sbin/logwatch --output mail
    ```


## Node.js with HTTP Proxy Configuration

1. Connect over SSH to your EC2 instance (see [Amazon EC2 Configuration](#amazon-ec2-configuration) step #5).

    > Note that you may need to add the flag `-p 44444` if you disabled normal SSH access on port 22 in the previous section.

2. Install git dependency packages:

    ```bash
    sudo apt-get update && sudo apt-get install git-core curl build-essential openssl libssl-dev
    ```

3. Install node:

    ```bash
    git clone https://github.com/joyent/node.git
    cd node
    git checkout v0.10.4
    ./configure --openssl-libpath=/usr/lib/ssl
    make
    make test
    sudo make install
    ```

4. Configure upstart:

    > Create a new file `/etc/init/cluster`

    ```bash
    sudo vim /etc/init/cluster
    ```

    ```bash
    #!upstart

    description "cluster"
    author "ubuntu"

    start on runlevel [2345]
    stop on runlevel [016]

    # restart when job dies
    respawn

    # give up after 5 respawns in 60s
    respawn limit 5 60

    # start the script and log output
    script
      exec sudo node /home/ubuntu/cluster/server.js >> /var/log/cluster.log 2>&1
    end script
    ```

5. Install http-proxy:

    ```bash
    mkdir /home/ubuntu/cluster
    cd /home/ubuntu/cluster
    npm install --save http-proxy
    ```

6. Setup http-proxy cluster:

    > Create a new file `/home/ubuntu/cluster/server.js`

    ```bash
    sudo vim /home/ubuntu/cluster/server.js
    ```

    ```js
    var cluster   = require('cluster')
      , numCPUS   = require('os').cpus().length
      , sites     = require('./sites')
      , httpProxy = require('http-proxy')

    var options = {
        hostnameOnly: true
      , router: sites
    }

    if (cluster.isMaster) {
      for(var i=0; i < numCPUS; i+=1) {
        cluster.fork()
      }
      cluster.on('exit', clusterExit)
    } else {
      var httpServer = httpProxy.createServer(options)
      httpServer.listen(80)
    }

    function clusterExit(worker, code, signal) {
      console.log('worker %d died', worker.process.pid)
    }
    ```

7. Add a test virtual host with your EC2's Public IP to a new file `/home/ubuntu/cluster/sites.js`:

    ```js
    module.exports = {
      "12.34.56.78": "127.0.0.1:3000"
    }
    ```

8. Start cluster:

    ```bash
    sudo start cluster
    ```

9. Test to ensure that cluster is working properly.

    ```bash
    npm install -g http-server
    http-server -p 3000
    Starting up http-server, serving ./ on port: 3000
    Hit CTRL-C to stop the server
    ```

    > Now visit <http://12.34.56.78/> (replace with your instance's IP).  If you run into issues, you may need to log into EC2's security groups and open up port 3000.

10. Optionally you can run the following command to increase the number of sockets that Node can connect to.

    ```bash
    sudo ulimit -n 999999
    ```


## Apache Legacy Support

If you need to support legacy web applications running on PHP/Apache/MySQL, then we can setup Apache to run on an alternate port such as 8080.

1. Install dependencies including Apache, PHP, and MySQL.

    ```bash
    sudo apt-get install apache2 php5 libapache2-mod-php5 mysql-server
    ```

2. Modify the default port from 80 to 8080:

    ```bash
    sudo vim /etc/apache2/ports.conf
    ```

    > Edit "/etc/apache2/ports.conf":

    ```diff
    +NameVirtualHost *:8080
    -NameVirtualHost *:80
    +Listen 8080
    -Listen 80
    ```

    ```bash
    sudo vim /etc/apache2/sites-available/default
    ```

    > Edit "/etc/apache2/sites-available/default":

    ```diff
    +<VirtualHost *:8080>
    -<VirtualHost *:80>
    ```

3. Reload the Apache configuration and restart the server.

    ```bash
    sudo /etc/init.d/apache2 reload
    sudo service apache2 restart
    ```

4. Add a virtual host to `sites.js` (replace "mysite.com" with your domain):

    ```bash
    vim /home/ubuntu/cluster/sites.js
    ```

    ```diff
    module.exports = {
        "12.34.56.78": "127.0.0.1:3000"
    +  , "www.mysite.com": "127.0.0.1:8080"
    +  , "mysite.com": "127.0.0.1:8080"
    }
    ```

5. Add a virtual host to `/etc/apache2/sites-available/mysite.com`:

    ```bash
    sudo vim /etc/apache2/sites-available/mysite.com
    ```

    ```conf
    <VirtualHost *:8080>
      ServerAdmin	your@email.com
      ServerName	mysite.com
      ServerAlias	www.mysite.com
      DocumentRoot /home/ubuntu/mysite.com
    </VirtualHost>
    ```

6. Create the virtual host folder and a test index file:

    ```bash
    mkdir /home/ubuntu/mysite.com
    vim /home/ubuntu/mysite.com/index.html
    ```

    ```html
    <h1>MySite.com</h1>
    ```

7. Enable the virtual site and reload Apache.

    ```bash
    sudo a2ensite mysite.com
    sudo /etc/init.d/apache2 reload
    ```

8. Visit <http://mysite.com> to test out the http-proxy with Node to Apache.


## Inspiration

* [Feross Aboukhadijeh](http://feross.org/how-to-setup-your-linode/)
* [Bryan Kennedy](http://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers)
