# Installing Glassfish 4.1.1 on Ubuntu 14.04 LTS (from the command line)
After my short tutorial [Web Application based on WebSocket API for Java](https://github.com/teocci/RealtimeBoard), I received a lot of emails asking about how to deploy a Glassfish server on Ubuntu from the command line.
Therefore, in this tutorial I will explain how to install a Glassfish 4.1.1 Server on an Ubuntu 14.04 LTS Server. It will also cover **some but NOT all*** security concerns. The steps have been executed successfully on Ubuntu 14.04 LTS Server edition (64-bit). I have tested everything by using Physical Servers and Virtual Machines (you might want to use Virtual Machines as well). You can use this tutorial for setting up a Glassfish server. Both Ubuntu root servers and Ubuntu virtual servers should be fine for this tutorial, so you can choose any hosting package offered by the provider of your choice. In all cases you need to make sure to have root access to your server. Of course, in this tutorial you have to execute lots of commands on the shell, so you should also need to be familiar with the Unix/Linux command line. After having this tutorial completed you can use your new Glassfish installation to host your own Java EE 7 compliant applications.

This is my tutorial where I described how to install Glassfish on Ubuntu. I hope it will help others and thank you for reading my tutorial! If you have any questions do not hesitate to contact me. Any feedback is welcome! Also feel free to leave a comment or an issue. For helping me to maintain my tutorials any donation is welcome.

## Table of contents:

1. Setting up the OS environment
2. Setting up Java
3. Downloading and installing Glassfish
4. Setting up an init script
5. Glassfish autostart: adding init script to default runlevels
6. Security configuration before first startup
7. Run Glassfish

Creating this tutorial meant a lot of effort - although I could reuse a lot of the work I invested into my previous tutorial. 

## 1. Setting up the OS environment

  Before you start doing anything you should think about a security concept. A -detailed security concept- is out of scope for this tutorial. Very important from security point of view is not to run your Glassfish server as `root`. This means you need to create a user with restricted rights which you can use for running Glassfish. Once you have added a new user, let's say 'gladmin', you might also want to add a new group called 'gf-admins'. You can use this group for all users that shall be allowed to "administer" your Glassfish in -full depth-. In full depth means also modifying different files in the Glassfish home directory. Below you find user and group related commands that you might want to use.
  Here are the commands:

  ```
  sudo adduser --home /home/glassfish --system --shell /bin/bash glassfish
  sudo groupadd glassfishadm
  sudo usermod -a -G glassfishadm $myAdminUser
  ``` 

  *  Add a new user called glassfish: 

  ```
  sudo adduser --home /home/glassfish --system --shell /bin/bash glassfish
  ```
   
  * Add a new group for glassfish administration: 

  ```
  sudo groupadd glassfishadm
  ``` 

  * Add your users that shall be Glassfish adminstrators: 

  ```
  sudo usermod -a -G glassfishadm $myAdminUser
  ``` 
  * In case you want to delete a group some time later (ignore warnings):

  ```
  delgroup glassfishadm
  ```

  Glassfish allows some of the configuration tasks to be managed via a web based Administration GUI. We will simply call it "Admin GUI" from now on. You can reach the Admin GUI by visiting http://www.yourserver.com:4848/ in your browser (please replace www.yourserver.com with localhost or where ever your Glassfish server is). As you can see port 4848 is used. Of course, we don't want anyone to access our AdminGUI. Therefore we have to restrict access to the AdimnGUI. A way do this is to block port 4848 via the firewall. Anything you can do via AdminGUI is also available via the asadmin tool that ships with Glassfish. So you don't have to worry about not being able to configure Glassfish if you block the AdminGUI.

  Usually you want to run Glassfish on port 80 (everything below port 1024 has to run as `root`). But since we don't suggest to run Glassfish as `root` we cannot run Glassfish on port 80. But there are still ways to run Glassfish as a non-root user and still receive http requests on port 80. One option could be mod_jk, but this would only be another component that needs to be managed. An easy way is to use a simple `iptables` redirection rule, that redirects requests on port 80 to port 8080 (http) and requests on port 443 to port 8181 (https).

  You should make sure that you do not block other important ports, for example your `ssh` port which usually runs on port 22 (else you will be locked out). Changing the default ssh port to some other is actually a good idea, but for now we will simply assume your ssh port is 22. Another helpfull `iptables` rule related to your ssh port 22 is to slow down connection tries from an ip if they fail 3 times. I found a rule for that on the web and added it below. Although I will not mention it here you should also use other techniques and tools to secure your ssh port. Unfortunately, I did not get the time to post a tutorial about that.

  Now we have created a lot of rules. You could enter them always one by one, but we don't want this kind of effort. I suggest to enter the following `iptables` rules in a separate file which contains all of our `iptables` related ideas we discussed so far:

  File: [iptables.ENABLE_4848.rules](https://github.com/teocci/GlassfishServer/blob/master/scripts/iptables.ENABLE_4848.rules)
  File: [iptables.DISABLE_4848.rules](https://github.com/teocci/GlassfishServer/blob/master/scripts/iptables.DISABLE_4848.rules)

  I suggest to create a file called `iptables.DISABLE_4848.rules` which contains exactly everything from the file `iptables.ENABLE_4848.rules` but with line 28 commented. Of course, you have to make both files executable with the command `chmod +x $filename` (see below). Then you can simply run one of the scripts when ever you want to disable or enable the AdminGUI on port 4848, for instance `sudo ./iptables.DISABLE_4848.rules`. Making the `iptables` rules executable:

  ```  
  chmod +x iptables.DISABLE_4848.rules
  chmod +x iptables.ENABLE_4848.rules
  ```

  Please also do not forget that all your `iptables rules` should also be activated if your Ubuntu server is restarted. Otherwise you would have to remember to run your `iptables` rules manually after each restart. If you forget to run them all manually, or if you have simply forgotten that your server has been restarted, then your firewall is open for everyone. If you are lucky nothing will happen, if not you might get some successful instrusion attacks. Lines 58 and 59 will help you to make sure your rules are automatically loaded after each restart. But this is not everything for `iptables` configuration on startup. You also need to create a file at `/etc/network/if-pre-up.d/iptablesload` and one at `/etc/network/if-post-down.d/iptablessave`. For more information please have a look at [the official Ubuntu help sites for `iptables`](http://help.ubuntu.com/community/IptablesHowTo). The following code boxes show the content of our two files and where they have to be created. As you can see in both the latter two code boxes, line 2 is refering to the file /etc/my-iptables.rules, which we have defined in line 58 and 59 of our files iptables.DISABLE_4848.rules and iptables.ENABLE_4848.rules respectively. I have added /sbin/ in front of the iptables commands (see below) because I was facing the problem that iptables commands without /sbin/ could not be found at the time when the files iptablesload or iptablessave were executed during the Ubuntu server startup process.

  Create iptablesload and iptablessave files:

  ```  
  # first we have to create these two files:
  sudo touch /etc/network/if-pre-up.d/iptablesload
  sudo touch /etc/network/if-post-down.d/iptablessave
  ```

  File: /etc/network/if-pre-up.d/iptablesload

  ```
  #!/bin/sh
  /sbin/iptables-restore < /etc/my-iptables.rules
  exit 0
  ```

  File: /etc/network/if-post-down.d/iptablessave

  ```  
  #!/bin/sh
  /sbin/iptables-save -c > /etc/my-iptables.rules
  if [ -f /etc/iptables.downrules ]; then
     /sbin/iptables-restore < /etc/iptables.downrules
  fi
  exit 0
  ```

  Finally you have to make sure that both files are executable. For that you only need to execute the following commands once.
  Here are the commands:

  ```  
  sudo chmod +x /etc/network/if-post-down.d/iptablessave
  sudo chmod +x /etc/network/if-pre-up.d/iptablesload
  ```

  At this point you can try what happens if you reboot your Ubuntu server (sudo reboot). After Ubuntu has restarted just try sudo iptables -L on the shell. It should show you the rules we have defined. You should see something like this if you hit sudo ./iptables.DISABLE_4848.rules before rebooting:
  command output for: sudo iptables -L


  ```
  Chain INPUT (policy ACCEPT)
  target     prot opt source               destination
  DROP       tcp  --  anywhere             anywhere            tcp dpt:ssh state NEW recent: UPDATE seconds: 60 hit_count: 3 TTL-Match name: sshprobe side: source
  ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:ssh state NEW recent: SET name: sshprobe side: source
  ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED
  ACCEPT     all  --  anywhere             anywhere
  ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:www
  ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:ssh
  ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:http-alt
  ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:8181
  ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:https
  DROP       all  --  anywhere             anywhere
   
  Chain FORWARD (policy DROP)
  target     prot opt source               destination
   
  Chain OUTPUT (policy ACCEPT)
  target     prot opt source               destination
  ```

  Your firewall settings are loaded automatically whenever Ubuntu is starting up. We can continue with the next steps. Please do not forget that these are only some minimum firewall settings. For maximum security you might need to add your own `iptables` rules.

## 2. Setting up Java

  The next step is to set up Java. As you can see on [the official Oracle Glassfish 4.1.1 download site](http://download.java.net/glassfish/4.1.1/release/glassfish-4.1.1.zip) it is recommended to use the latest Oracle JDK 8 for GlassFish 4.1. On the other hand you can see that Java EE 7 requires at least JDK 7 or above. That means these are the minimum versions you want to use! I suggest to remove the ObenJDK first if you have it installed and install the Oracle JDK and JRE. I also suggest to remove any previous Sun/Oracle JDK and JRE already installed on your system. On a production system I don't suggest to use the ObenJDK because it is not certified (but that's another story...).

  So what we want now is to install the latest JDK 8 from Oracle. Unfortunately Oracle's JDK 8 is not available via Ubuntu's aptitude. So if you want to install Oracles's JDK 8 you will have to download the Linux distribution from [Oracle's JDK 8 Download site](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) and install it manually. Installing Oracle's JDK 8 manually means downloading it manually, uploading it manually to your Ubuntu server and the executing the actual installation process. See [How can I install Sun/Oracle's proprietary Java JDK 6/7/8 or JRE?](http://askubuntu.com/questions/56104/how-can-i-install-sun-oracles-proprietary-java-jdk-6-7-8-or-jre) on askubuntu.com for more information. The following commands expect that you have already downloaded Oracle's JDK 8 (i.e. jdk-8u101-linux-x64.tar.gz) manually to your local machine (i.e. by the browser of your choice) and that the same file has been uploaded to ~/downloads on your Ubuntu server (i.e. via scp).

  Bash commands (Ubuntu 14.04 LTS):

  ```  
  #remove OpenJDK if installed - this is how you would remove OpenJDK :
  sudo apt-get remove openjdk-6-jre openjdk-6-jdk
  sudo apt-get remove openjdk-7-jre openjdk-7-jdk
  #OpenJDK 8 packages not yet in the official 14.04 LTS repos 
  #sudo apt-get remove openjdk-8-jre openjdk-8-jdk
   
  #remove sun's jdk if you have it (I am sure you don't have it):
  #sudo apt-get remove sun-java6-jdk sun-java6-jre
   
  #get rid of several automatically-installed packages that are not needed anymore...
  sudo apt-get autoremove
  sudo apt-get autoclean

  #download and extract oracle's jdk
  wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.tar.gz
  tar -xvf jdk-8u101-linux-x64.tar.gz
   
  #remove previous installations if available
  sudo rm -rf /usr/local/jdk1.8.0
   
  #make sure our destination folder exists
  sudo mkdir /usr/lib/jvm
   
  #move extracted files
  sudo mv ./jdk1.8.0_101 /usr/local/jdk1.8.0
  sudo chgrp -R root /usr/local/jdk1.8.0
  sudo chown -R root /usr/local/jdk1.8.0
   
  #update alternatives - make sure environment points to a valid install if you are replacing...
  sudo update-alternatives --install "/usr/bin/java" "java" "/usr/local/jdk1.8.0/bin/java" 1
  sudo update-alternatives --install "/usr/bin/javac" "javac" "/usr/local/jdk1.8.0/bin/javac" 1
  sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/usr/local/jdk1.8.0/bin/javaws" 1
  sudo update-alternatives --config java
   
  #check JDK by looking in the /etc/alternatives/ directory
  cd /etc/alternatives
  ls -lrt java*
   
  #this is optionally since we will add everything to the PATH variable (i usually don't use this):
  sudo update-alternatives --install "/usr/bin/apt" "apt" "/usr/local/jdk1.8.0/bin/apt" 1
  sudo update-alternatives --install "/usr/bin/idlj" "idlj" "/usr/local/jdk1.8.0/bin/idlj" 1
  sudo update-alternatives --install "/usr/bin/jarsigner" "jarsigner" "/usr/local/jdk1.8.0/bin/jarsigner" 1
  sudo update-alternatives --install "/usr/bin/java-rmi.cgi" "java-rmi.cgi" "/usr/local/jdk1.8.0/bin/java-rmi.cgi" 1
  sudo update-alternatives --install "/usr/bin/javadoc" "javadoc" "/usr/local/jdk1.8.0/bin/javadoc" 1
  sudo update-alternatives --install "/usr/bin/javah" "javah" "/usr/local/jdk1.8.0/bin/javah" 1
  sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/usr/local/jdk1.8.0/bin/javaws" 1
  sudo update-alternatives --install "/usr/bin/jconsole" "jconsole" "/usr/local/jdk1.8.0/bin/jconsole" 1
  sudo update-alternatives --install "/usr/bin/jdb" "jdb" "/usr/local/jdk1.8.0/bin/jdb" 1
  sudo update-alternatives --install "/usr/bin/jinfo" "jinfo" "/usr/local/jdk1.8.0/bin/jinfo" 1
  sudo update-alternatives --install "/usr/bin/jps" "jps" "/usr/local/jdk1.8.0/bin/jps" 1
  sudo update-alternatives --install "/usr/bin/jsadebugd" "jsadebugd" "/usr/local/jdk1.8.0/bin/jsadebugd" 1
  sudo update-alternatives --install "/usr/bin/jstat" "jstat" "/usr/local/jdk1.8.0/bin/jstat" 1
  sudo update-alternatives --install "/usr/bin/jvisualvm" "jvisualvm" "/usr/local/jdk1.8.0/bin/jvisualvm" 1
  sudo update-alternatives --install "/usr/bin/native2ascii" "native2ascii" "/usr/local/jdk1.8.0/bin/native2ascii" 1
  sudo update-alternatives --install "/usr/bin/pack200" "pack200" "/usr/local/jdk1.8.0/bin/pack200" 1
  sudo update-alternatives --install "/usr/bin/rmic" "rmic" "/usr/local/jdk1.8.0/bin/rmic" 1
  sudo update-alternatives --install "/usr/bin/rmiregistry" "rmiregistry" "/usr/local/jdk1.8.0/bin/rmiregistry" 1
  sudo update-alternatives --install "/usr/bin/serialver" "serialver" "/usr/local/jdk1.8.0/bin/serialver" 1
  sudo update-alternatives --install "/usr/bin/tnameserv" "tnameserv" "/usr/local/jdk1.8.0/bin/tnameserv" 1
  sudo update-alternatives --install "/usr/bin/wsgen" "wsgen" "/usr/local/jdk1.8.0/bin/wsgen" 1
  sudo update-alternatives --install "/usr/bin/xjc" "xjc" "/usr/local/jdk1.8.0/bin/xjc" 1
  sudo update-alternatives --install "/usr/bin/appletviewer" "appletviewer" "/usr/local/jdk1.8.0/bin/appletviewer" 1
  sudo update-alternatives --install "/usr/bin/extcheck" "extcheck" "/usr/local/jdk1.8.0/bin/extcheck" 1
  sudo update-alternatives --install "/usr/bin/jar" "jar" "/usr/local/jdk1.8.0/bin/jar" 1
  sudo update-alternatives --install "/usr/bin/javafxpackager" "javafxpackager" "/usr/local/jdk1.8.0/bin/javafxpackager" 1
  sudo update-alternatives --install "/usr/bin/javap" "javap" "/usr/local/jdk1.8.0/bin/javap" 1
  sudo update-alternatives --install "/usr/bin/jcmd" "jcmd" "/usr/local/jdk1.8.0/bin/jcmd" 1
  sudo update-alternatives --install "/usr/bin/jcontrol" "jcontrol" "/usr/local/jdk1.8.0/bin/jcontrol" 1
  sudo update-alternatives --install "/usr/bin/jhat" "jhat" "/usr/local/jdk1.8.0/bin/jhat" 1
  sudo update-alternatives --install "/usr/bin/jmap" "jmap" "/usr/local/jdk1.8.0/bin/jmap" 1
  sudo update-alternatives --install "/usr/bin/jrunscript" "jrunscript" "/usr/local/jdk1.8.0/bin/jrunscript" 1
  sudo update-alternatives --install "/usr/bin/jstack" "jstack" "/usr/local/jdk1.8.0/bin/jstack" 1
  sudo update-alternatives --install "/usr/bin/jstatd" "jstatd" "/usr/local/jdk1.8.0/bin/jstatd" 1
  sudo update-alternatives --install "/usr/bin/keytool" "keytool" "/usr/local/jdk1.8.0/bin/keytool" 1
  sudo update-alternatives --install "/usr/bin/orbd" "orbd" "/usr/local/jdk1.8.0/bin/orbd" 1
  sudo update-alternatives --install "/usr/bin/policytool" "policytool" "/usr/local/jdk1.8.0/bin/policytool" 1
  sudo update-alternatives --install "/usr/bin/rmid" "rmid" "/usr/local/jdk1.8.0/bin/rmid" 1
  sudo update-alternatives --install "/usr/bin/schemagen" "schemagen" "/usr/local/jdk1.8.0/bin/schemagen" 1
  sudo update-alternatives --install "/usr/bin/servertool" "servertool" "/usr/local/jdk1.8.0/bin/servertool" 1
  sudo update-alternatives --install "/usr/bin/unpack200" "unpack200" "/usr/local/jdk1.8.0/bin/unpack200" 1
  sudo update-alternatives --install "/usr/bin/wsimport" "wsimport" "/usr/local/jdk1.8.0/bin/wsimport" 1
  sudo update-alternatives --config java
   
  #setting JAVA_HOME and AS_JAVA globally for all users (only for bash)
  sudo nano /etc/bash.bashrc
  #append the following lines:
  export GLASSFISH_HOME=/home/glassfish
  export JAVA_HOME=/usr/local/jdk1.8.0
  export PATH=$PATH:$JAVA_HOME/bin:$GLASSFISH_HOME/bin
   
  #setting JAVA_HOME ans AS_JAVA for everyone (best place for setting env vars globally)
  #see https://help.ubuntu.com/community/EnvironmentVariables for details
  sudo nano /etc/environment
  #append the following lines:
  JAVA_HOME=/usr/local/jdk1.8.0
  #we will set this here because it might prevent problems later
  AS_JAVA=/usr/local/jdk1.8.0
   
  #check JDK by looking in the /etc/alternatives/ directory...
  cd /etc/alternatives
  ls -lrt java*
  ```

  SSL (Secure Sockets Layer) and TLS (Transport Layer Security) are protocols designed to help protect the privacy and integrity of data while it is being transferred across a network. SSLv3 (or simply SSL 3) is not safe anymore, that's why we will disable SSL in step 6. Security configuration before first startup (see below). However, if you run Glassfish with a standard JRE you will realize soon that only 128-bit ciphers suites are supported in your Glassfish. It is quite easy to enable cipher suites with more than 128-bit encryption, i.e. 256-bit AES. The restriction in Glassfish comes from the fact that Glassfish uses what ever is supported by the underlying JSSE. To enable 256-bit cipher suites we need to download the Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files 8 and copy them to /usr/local/jdk1.8.0/jre/lib/security/ (all Java gurus should know that step very well). The following commands assume that you have downloaded the JCE archive with a browser to your local machine and saved it to ~/Downloads:
  Bash commands for enabling Java Cryptography Extension (JCE) Unlimited Strength:

  ```
  #upload jce to ubuntu server via scp (assuming you have scp, else upload somehow else...)
  #(replace myuser with your ubuntu user and ipAddressOfUbuntuServer with the ip address of you ubuntu server)
  scp ~/Downloads/jce_policy-8.zip myuser@ipAddressOfUbuntuServer:downloads
  #now the archive is on the ubuntu server at /home/myuser/downloads
   
  #now we execute the next commands on the ubuntu server
  #first, backup old files
  sudo mv /usr/local/jdk1.8.0/jre/lib/security/local_policy.jar /usr/local/jdk1.8.0/jre/lib/security/local_policy.jar.bak
  sudo mv /usr/local/jdk1.8.0/jre/lib/security/US_export_policy.jar /usr/local/jdk1.8.0/jre/lib/security/US_export_policy.jar.bak
   
  #now copy local_policy.jar and US_export_policy.jar to /usr/local/jdk1.8.0/jre/lib/security/
  cd ~/downloads
  sudo unzip ./jce_policy-8.zip
  sudo cp ./UnlimitedJCEPolicyJDK8/local_policy.jar /usr/local/jdk1.8.0/jre/lib/security/
  sudo cp ./UnlimitedJCEPolicyJDK8/US_export_policy.jar /usr/local/jdk1.8.0/jre/lib/security/
  sudo chgrp -R root /usr/local/jdk1.8.0/jre/lib/security
  sudo chown -R root /usr/local/jdk1.8.0/jre/lib/security
  ```

  Now Glassfish will be allowed to offer even AES-256. You could also allow only a certain set of cipher suites by changing the settings of your Http Listeners and IIOP Listeners in the admin console or via asadmin later, once Glassfish is installed. If you are interested to learn more about security I suggest to read 256-bit AES Encryption for SSL and TLS: Maximal Security. This paper also tells you how to force certain ciphers in different browsers. However, to be very sure in risky environments, it's a good decision to force a certain set of allowed ciphers directly on your Glassfish server (we will not focus too much on that).

  A few days ago the Internet Engineering Task Force (IETF) has announced with Request for Comment 7465 to prohibit RC4 Cipher Suites in TLS. RC4 is considered to be very weak and insecure, so let's follow RFC 7465 and disable RC4. You could do this directly in Glassfish via the admin console or asadmin CLI. However, I believe it's a better idea to do this globally in your JRE. The following configuration will make sure that your Glassfish server will not use any RC4-based cipher suites (not disabling RC4 is a risk these days):
  Disable weak ciphers globally, i.e. RC4:

  ```
  sudo nano /usr/local/jdk1.8.0/jre/lib/security/java.security
  #then add RC4 to jdk.tls.disabledAlgorithms, i.e. (SSLv3 is the only default in java 1.8.0_31):
  jdk.tls.disabledAlgorithms=SSLv3, RC4
  # for higher security add maybe this:
  # jdk.tls.disabledAlgorithms=MD5, SSLv3, SHA1, DSA, RC4, RSA keySize < 2048, DH keySize < 768
  #also add this because of Logjam attack:
  jdk.tls.ephemeralDHKeySize=2048
  ```
## 3. Downloading and Installing Glassfish

  Now we can download Glassfish. I suggest to switch the user now to glassfish, which we have created in the first step. We want to download the Glassfish zip installation file to /home/glassfish/downloads/. Afterwards the zip file has to be extracted and the content can be moved to /home/glassfish/ - this is everything needed for installing Glassfish. Usually the zip file is extracted to a directory called ./glassfish4/. Make sure to move the content of ./glassfish4/ and not ./glassfish4 itself to /home/glassfish/.
  Here are the commands:

  ```  
  #if you dont't have "unzip" installed run this here first
  sudo apt-get install unzip
   
  #now switch user to the glassfish user we created (see step 1)
  sudo su glassfish
   
  #change to home dir of glassfish
  cd /home/glassfish/
   
  #create new directory if not already available
  mkdir downloads
   
  #go to the directory we created
  cd /home/glassfish/downloads/
   
  #download Glassfish and unzip
  #Hint: get the correct link from  https://glassfish.java.net/download.html if wget fails
  wget http://download.java.net/glassfish/4.1/release/glassfish-4.1.zip
  unzip glassfish-4.1.zip
   
   
  #move the relevant content to home directory
  mv /home/glassfish/downloads/glassfish4/* /home/glassfish/
  #if something has not been moved, then move it manually, i.e.:
  mv /home/glassfish/downloads/glassfish4/.org.opensolaris,pkg /home/glassfish/.org.opensolaris,pkg
   
  #exit from glassfish user
  exit
   
  #change group of glassfish home directory to glassfishadm
  sudo chgrp -R glassfishadm /home/glassfish
   
  #just to make sure: change owner of glassfish home directory to glassfish
  sudo chown -R glassfish /home/glassfish
   
  #make sure the relevant files are executable/modifyable/readable for owner and group
  sudo chmod -R ug+rwx /home/glassfish/bin/
  sudo chmod -R ug+rwx /home/glassfish/glassfish/bin/
   
  #others are not allowed to execute/modify/read them
  sudo chmod -R o-rwx /home/glassfish/bin/
  sudo chmod -R o-rwx /home/glassfish/glassfish/bin/
  ```

  At this point you can give it a try and start you Glassfish server. But do not forget to stop it again before you continue with the next steps. Here are the commands for starting and stopping Glassfish:
  Here are the commands:

  ```  
  #now switch user to the glassfish user
  sudo su glassfish
   
  #start glassfish
  /home/glassfish/bin/asadmin start-domain domain1
  #check the output...
   
  #stop glassfish
  /home/glassfish/bin/asadmin stop-domain domain1
  #check the output...
   
  #exit from glassfish user
  exit
  ```

## 4. Setting up an init script

  Let's create an init script now. It helps you to start, stop and restart your Glassfish easily. We also need this to make Glassfish start up automatically whenever Ubuntu is rebooting. The file we need to create is /etc/init.d/glassfish. For starting and stopping Glassfish we will use the asadmin tool that ships with Glassfish (we used it a little in the previous step). Here are the commands:

  ```  
  #create and edit file
  sudo vi /etc/init.d/glassfish
   
  #(paste the lines below into the file and save it...):
   
  #! /bin/sh
   
  #to prevent some possible problems
  export AS_JAVA=/usr/local/jdk1.8.0
   
  GLASSFISHPATH=/home/glassfish/bin
   
  case "$1" in
  start)
  echo "starting glassfish from $GLASSFISHPATH"
  sudo -u glassfish $GLASSFISHPATH/asadmin start-domain domain1
  ;;
  restart)
  $0 stop
  $0 start
  ;;
  stop)
  echo "stopping glassfish from $GLASSFISHPATH"
  sudo -u glassfish $GLASSFISHPATH/asadmin stop-domain domain1
  ;;
  *)
  echo $"usage: $0 {start|stop|restart}"
  exit 3
  ;;
  esac
  :
  ```

  As you can see Glassfish is started with the user glassfish. It's always a bad idea to run a webserver with root. You should always use a restricted user - in our case this will be the user glassfish. You will learn how to use the script we just created in the next steps.

## 5. Glassfish autostart: adding init script to default runlevels

  The init script is set up. Now we can add it to the default run levels. This way our Glassfish will startup whenever Ubuntu is restarted.
  Here are the commands:

  ```  
  #make the init script file executable
  sudo chmod a+x /etc/init.d/glassfish
   
  #configure Glassfish for autostart on ubuntu boot
  sudo update-rc.d glassfish defaults
   
  #if apache2 is installed:
  #stopping apache2
  sudo /etc/init.d/apache2 stop
  #removing apache2 from autostart
  update-rc.d -f apache2 remove
  ```

  From now on you can start, stop or restart your Glassfish like this (Ubuntu will also do it this way):

  Here are the commands:

  ```
  #start
  /etc/init.d/glassfish start
   
  #stop
  /etc/init.d/glassfish stop
   
  #restart
  /etc/init.d/glassfish restart
  ```

## 6. Security configuration before first startup

  Even now we should not really use Glassfish in production. We will now begin the configuration of Glassfish itself. You should always run these steps, for example changing the default passwords, enabling `https`, changing the default ssl certificate to be used for `https` etc. We will also put our attention on Glassfish obfuscation.

  * Our first step is to change the master password. Glassfish uses it to protect the domain-encrypted files from unauthorized access, i.e. the certificate store which contains the certificates for `https` communication. When Glassfish is starting up it tries to read such "secured" files - for exactly this purpose Glassfish needs to be provided with the master password either in an interactive way or in a non-interactive way. I will choose the non-interactive way because we want our Glassfish to start up on Ubuntu reboot as a deamon (in the Windows world this would be called a service). This is necessary so that the `start-domain` command can start the server without having to prompt the user. To accomplish this we need to set the `savemasterpassword` option to true. This option indicates whether the master password should be written to the file system. The file is called master-password and can be found at `/home/glassfish/glassfish/domains/domain1/master-password`. To change the master password you have to ensure that Glassfish is not running - only then you can call the command `change-master-password` which will interactively ask you for the new password. Here are the commands:

  ```
  #switch user to glassfish (stay with this user for complete Step 6!)
  sudo su glassfish

  #change master password, default=changeit
  /home/glassfish/bin/asadmin change-master-password --savemasterpassword=true
  #prompt: choose your new master password ==> myMasterPwd
  # ==> stores file to /home/glassfish/glassfish/domains/domain1/master-password
  ```

  * The next step is to change the administration password with `change-admin-password`. Because this command is a remote command we need to ensure that Glassfish is running before we can execute the command. Since we want "automatic login" we will create an admin password file allowing us to login without being asked for credetials (it will be stored at `/home/glassfish/.gfclient/pass`). Here are the commands:

  ```
  #change admin password
  /home/glassfish/bin/asadmin change-admin-password
  #1. enter "admin" for user (default)
  #2. hit enter because default pwd is empty
  #3. choose you new pwd ==> myAdminPwd

  #now we have to start Glassfish
  /home/glassfish/bin/asadmin start-domain domain1

  #login for automatic login...
  /home/glassfish/bin/asadmin login
  #prompt:
  #user = admin
  #password = myAdminPwd
  #==> stores file to /home/glassfish/.gfclient/pass

  #now stop Glassfish
  /home/glassfish/bin/asadmin stop-domain domain1
  ```

  * Glassfish is coming with a pre-configured certificate which is used for ssl (`https`). You can see it in the `keystore.jks` file if you check for the alias `s1as`. Since Glassfish 3.1 there is even another preconfigured certificate available: glassfish-instance. But that also means that everybody else can get these two certificates, the public keys, private keys, etc. With that information you could never be safe because "others" could "read" your data sent to Glassfish via `https`. That means you should always make sure to replace the pre-configured `s1as` and `glassfish-instance` entries in your `keystore`. But you should not delete them as long as the alias `s1as` and `glassfish-instance` are still in use. I faced some strange behavior as I did not think of that at the beginning when I simply deleted `s1as` - learn from my mistake and do not delete it for now... But we can help us with generating a new alias first `myAlias` and when ever needed or wanted we could change each occurrence of `s1as` to `myAlias` (i.e. via admin console) and then we could finally delete that `s1as`. The same has to be done also for `glassfish-instance`.

  The following code box shows you the commands we need for modifying our Glassfish keystore. As you can see we first delete our pre-configured `s1as` entry (Glassfish mustn't be running!). Later a new `s1as` entry is generated - it is now unique for us! Similar steps have to be executed also for our second certificate `glassfish-instance`. Here are the commands:
  
  ```
  #create new certs
  cd /home/glassfish/glassfish/domains/domain1/config/
  keytool -list -keystore keystore.jks -storepass myMasterPwd
  keytool -delete -alias s1as -keystore keystore.jks -storepass myMasterPwd
  keytool -delete -alias glassfish-instance -keystore keystore.jks -storepass myMasterPwd
  keytool -keysize 4096 -genkey -alias myAlias -keyalg RSA -dname "CN=nabisoft,O=nabisoft,L=Mannheim,S=Germany,C=DE" -validity 3650 -keypass myMasterPwd -storepass myMasterPwd -keystore keystore.jks
  keytool -keysize 4096 -genkey -alias s1as -keyalg RSA -dname "CN=nabisoft,O=nabisoft,L=Mannheim,S=Germany,C=DE" -validity 3650 -keypass myMasterPwd -storepass myMasterPwd -keystore keystore.jks
  keytool -keysize 4096 -genkey -alias glassfish-instance -keyalg RSA -dname "CN=nabisoft,O=nabisoft,L=Mannheim,S=Germany,C=DE" -validity 3650 -keypass myMasterPwd -storepass myMasterPwd -keystore keystore.jks
  keytool -list -keystore keystore.jks -storepass myMasterPwd

  #make sure that keystore.jks and cacerts.jks stay in sync, so let's update cacerts.jks:
  #1. export the two certificates we just created to files: 
  keytool -export -alias glassfish-instance -file glassfish-instance.cert -keystore keystore.jks -storepass myMasterPwd
  keytool -export -alias s1as -file s1as.cert -keystore keystore.jks -storepass myMasterPwd
  #2. now delete the old/existing aliases initially shipped with glassfish:
  keytool -delete -alias glassfish-instance -keystore cacerts.jks -storepass myMasterPwd
  keytool -delete -alias s1as -keystore cacerts.jks -storepass myMasterPwd
  #3. now import the two certificates we stored into files:
  keytool -import -alias glassfish-instance -file glassfish-instance.cert -keystore cacerts.jks -storepass myMasterPwd
  keytool -import -alias s1as -file s1as.cert -keystore cacerts.jks -storepass myMasterPwd

  #we can delete the two cert files now
  rm glassfish-instance.cert s1as.cert
  ```

  Now we want to enable `https` for the admin console. Once we have done that we can be sure that nobody can listen to our data sent via `https` because nobody else has our certificates, i.e. nobody can decrypt our password used for entering the admin console via browser (in case someone cought our data packages). However, sometimes clients want to connect via `https` using `SSLv3`, which is considered to be vulnerable nowadays. In other words: **disable SSLv3 completely where ever you find it - no discussion about that allowed!** 

  Here is a great post about [different ways of disabling SSLv3 in Glassfish 4.1][6.5]. I recommend use the command line version (`asadmin`). Furthermore, it seems that Glassfish 4.1.1 does not disable TLS client-initiated renegotiation by default, which is considered to open some doors for DoS attacks. The article [TLS Renegotiation and Denial of Service Attacks][6.4] by Ivan Ristic is a good read. For us this means we need to disable client-initiated renegotiation somehow. But this is not all we want to do here. We want to change some of the default JVM Options and we want to make our Glassfish not telling too much ("obfuscation").

  The first JVM Option we will change is replacing the `-client` option with the `-server` option. I expect the java option `-server` to be the better choice when it comes to performance. I have also decided to change `-Xmx512m` (Glassfish default) to a higher value: `-Xmx4096m`. Additionally, I have added `-Xms4096m`. Furthermore, we will remove `-XX:MaxPermSize=192m` and replace it with `-XX:MaxMetaspaceSize=512m` because [`-XX:MaxPermSize` was deprecated in JDK 8][6.3]. If you want to find out more about it then have a look at [this InfoQ article][6.2]. For more information about JVM options please check the [Oracle's official documentation][6.1].
  All JVM Options so far are optional. But at least adding `-Dproduct.name=""` is a good idea for everyone. If you would not add this then each http/https response will contain a header field like this: **Server: GlassFish Server Open Source Edition 4.1**
  This is some great piece of information for hackers - that's why you should disable it. We don't want Glassfish to talk too much for security reasons!

  We also don't want Glassfish to send a header similar to **X-Powered-By: Servlet/3.1 JSP/2.3** because this is telling everyone we are using a Servlet 3.1 container and that we are (of course) using Java etc. So we have to disable sending x-powered-by in the http/https headers. After executing the commands below our Glassfish will run in silent mode - it is not telling too much any more. Glassfish obfuscation accomplished. SSLv3 and cient-initiated renegotiation will also be disabled, which reduces the surface for attacks. Here are the commands:

  [6.1]: http://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
  [6.2]: http://www.infoq.com/articles/Java-PERMGEN-Removed
  [6.3]: http://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
  [6.4]: https://community.qualys.com/blogs/securitylabs/2011/10/31/tls-renegotiation-and-denial-of-service-attacks
  [6.5]: http://blog.c2b2.co.uk/2014/11/disabling-sslv3-in-glassfish-41.html

  
  ```
  # the commands here change the file at
  # /home/glassfish/glassfish/domains/domain1/config/domain.xml

  #first we have to start Glassfish
  /home/glassfish/bin/asadmin start-domain domain1

  # enable https for remote access to admin console
  # requests to http://xxx:4848 are redirected to https://xxx:4848
  /home/glassfish/bin/asadmin enable-secure-admin

  #change JVM Options
  #list current jvm options
  /home/glassfish/bin/asadmin list-jvm-options
  #now start setting some important jvm settings (change this as you wish)
  /home/glassfish/bin/asadmin delete-jvm-options -- -client
  /home/glassfish/bin/asadmin create-jvm-options -- -server
  #For server deployments, -Xms and -Xmx are often set to the same value
  /home/glassfish/bin/asadmin delete-jvm-options -- -Xmx512m
  /home/glassfish/bin/asadmin create-jvm-options -- -Xms4096m
  /home/glassfish/bin/asadmin create-jvm-options -- -Xmx4096m

  #-XX:MaxPermSize was deprecated in JDK 8, and superseded by the -XX:MaxMetaspaceSize option, see http://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
  # see also this good article: http://www.infoq.com/articles/Java-PERMGEN-Removed
  /home/glassfish/bin/asadmin delete-jvm-options '-XX\:MaxPermSize=192m'
  /home/glassfish/bin/asadmin create-jvm-options -- '-XX\:MaxMetaspaceSize=512m'
  /home/glassfish/bin/asadmin create-jvm-options -- '-XX\:MetaspaceSize=512m'

  #disable client-initiated renegotiation (to decrease the surface for DoS attacks)
  /home/glassfish/bin/asadmin create-jvm-options -Djdk.tls.rejectClientInitiatedRenegotiation=true

  #get rid of http header field value "server" (Glassfish obfuscation)
  /home/glassfish/bin/asadmin create-jvm-options -Dproduct.name=""

  #disable sending x-powered-by in http header (Glassfish obfuscation)
  /home/glassfish/bin/asadmin set server.network-config.protocols.protocol.http-listener-1.http.xpowered-by=false
  /home/glassfish/bin/asadmin set server.network-config.protocols.protocol.http-listener-2.http.xpowered-by=false
  /home/glassfish/bin/asadmin set server.network-config.protocols.protocol.admin-listener.http.xpowered-by=false
  /home/glassfish/bin/asadmin set server.network-config.protocols.protocol.sec-admin-listener.http.xpowered-by=false

  #optional: let's disable autodeploy (via autodeploy folder)
  /home/glassfish/bin/asadmin set server.admin-service.das-config.autodeploy-enabled=false

  #disable SSLv3
  /home/glassfish/bin/asadmin set server.network-config.protocols.protocol.http-listener-2.ssl.ssl3-enabled=false
  /home/glassfish/bin/asadmin set server.network-config.protocols.protocol.sec-admin-listener.ssl.ssl3-enabled=false
  /home/glassfish/bin/asadmin set server.iiop-service.iiop-listener.SSL.ssl.ssl3-enabled=false
  /home/glassfish/bin/asadmin set server.iiop-service.iiop-listener.SSL_MUTUALAUTH.ssl.ssl3-enabled=false

  #restart to take effect
  /home/glassfish/bin/asadmin stop-domain domain1
  /home/glassfish/bin/asadmin start-domain domain1
  #what jvm options are configured now?
  /home/glassfish/bin/asadmin list-jvm-options

  #we are done with user glassfish
  exit
  ```

## 7. Run Glassfish
  Finally we have come to where we wanted. We have installed, secured and configured our Glassfish installation. Here are the commands:

  ```
  #starting glassfish
  sudo /etc/init.d/glassfish start

  #remove glassfish from autostart
  #update-rc.d -f glassfish remove
  ```

  If you want a quick and free ssl security audit for your Glassfish server then go to [Qualys SSL Labs][4] and check your server configuration (the server must be accessible from the internet). I suggest every server available on the internet should use and even force HTTPS because it makes the web much safer. Even Google has announced to take HTTPS as a ranking signal. The "bad performance" argument is not really relevant anymore: [TLS has exactly one performance problem: it is not used widely enough][3]. Google gives you some good hints to [Secure your site with HTTPS][2]. Another good read is Ivan Ristić's paper: [SSL/TLS Deployment Best Practices][1].

  [1]: https://www.ssllabs.com/downloads/SSL_TLS_Deployment_Best_Practices.pdf
  [2]: https://support.google.com/webmasters/answer/6073543?utm_source=wmx_blog&utm_medium=referral&utm_campaign=tls_en_post
  [3]: https://istlsfastyet.com/?utm_source=wmx_blog&utm_medium=referral&utm_campaign=tls_en_post
  [4]: https://www.ssllabs.com/ssltest/analyze.html
