# Wazuh Project Learning Lab

#### Leon Scott -- Deployment and configuration of Wazuh
#### July 2023 
-------------------------------------------------------------------------------------------------------------------------------------------

The following contains the installation and reference notes for the configuration of a local Wazuh server used for Learning
I am not a professional employed in the field

HARWARE
- An existing laptop has been utilised with 500Gb of storage, a generation 5 Intel i5 on 32Gb RAM
- This handles the home network of about 10 Windows systems and linux servers used within the home Lab
- It also has a battery - cheap mans UPS

SERVER Base
- The base OS is Ubuntu 22.04
- This has been chosen due to its extended support, ability to go Ubuntu Pro, and large software respository support
- The minimal installation was chosen on installation to remove bloat

## Installation

Base operating system prep work
- Set a hostname that makes sense to you
- Set a password for the ROOT user - you will need the access
- Update the system using terminal
- Turn of hybernation and set the screen blanking to something realistic
- Check network settings and ensure IPV6 is enabled - you want a static IP address - I manage this via the firewalla/router - it will get a static address.
- Install openSSH - nothing fancy - its behind a firewall and wont be getting public access
- Reset all the SSH keys
    - sudo /bin/rm -v /etc/ssh/ssh_host_*
    - sudo dpkg-reconfigure openssh-server && sudo systemctl restart ssh
- Test the connection and log in via your daily driver to your ready to install Ubuntu OS
    - Using Windows terminal and log in to your ready to be setup server
    - Note if you are reading this this will allow for cut and paste which is extremly handy as we move along
- Install auditd
    - Reference: https://github.com/socfortress/Wazuh-Rules/blob/main/Auditd/auditd.conf
    - I will get to SOCFortress later - but lets just say I think he is a legend
    - Elevate to the ROOT user you setup earlier
        - su
        - apt install auditd
        - cd /etc/audit/rules.d
        - nano audit.rules
        - cut and paste the contents of the reference after removing the current contents (yep very handy is cut and paste)
        - systemctl restart auditd
        - auditctl -l 
        - The above command should give you a swag load of rules
- Install sysmon for Linux (optional step - this will generate sysmon events for Linux that Wazuh can import later)
    - Reference: https://github.com/socfortress/Wazuh-Rules/tree/main/Sysmon%20Linux
    - Reference: https://github.com/Sysinternals/SysmonForLinux
    - Reference: https://github.com/Sysinternals/SysmonForLinux/blob/main/INSTALL.md
    - Using the 3rd reference - Install sysmon for linux and start it.  Becareful where you run this from its going to drop a file
        - wget -q https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && dpkg -i packages-microsoft-prod.deb
        - sudo apt-get update && apt-get install sysmonforlinux
        - sysmon -i
        - systemctl status sysmon
    - You have now boosted your security and its logging away in the background



## Installation Wazuh Indexer

I use the step by step configuration installation method - mainly to help me understand
I am not using SSL in any way at this point nor am I complicating the installation.
I did NOT name the servers in any way.  I left it as default.
  - Reference: https://documentation.wazuh.com/current/installation-guide/index.html
  - Reference: https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/installation-assistant.html
  - Simply follow the guide
  - If you are using UbuntuOS like me you will need to install curl
      - apt install curl
      - Run the below command somewhere convenient.  Its going to drop a few files that you will need later so best to know where they are.
      - curl -sO https://packages.wazuh.com/4.4/wazuh-install.sh && curl -sO https://packages.wazuh.com/4.4/config.yml
      - edit the configuration file and point it towards the IP address of your server
      - I am not going to do anything fancy here so its just IP address changes - if you are looking at a cluster arrangement with names - then refer to additional references
      - bash wazuh-install.sh --wazuh-indexer node-1
      - bash wazuh-install.sh --start-cluster
  - Wazuh Indexer is now installed and running

## Installation Wazuh Manager

I am again running the installation assistant way of doing this.  Since we already have the core files its a simple matter of.
  - Reference: https://documentation.wazuh.com/current/installation-guide/wazuh-server/installation-assistant.html
  - bash wazuh-install.sh --wazuh-server wazuh-1
  - Remember later that its this process that installs Filebeat

## Installation of Wazuh Dashboard

I am again running the installation assistant way of doing this.  Since we already have the core files its a simple matter of.
  - Reference: https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/installation-assistant.html
  - bash wazuh-install.sh --wazuh-dashboard dashboard
  - At the end of this process you will see the admin username and password - make sure you copy this or you will be locked out the of webui and will need to reset the passwords of the entire system via a python script that Wazuh provides using a terminal
  - Note : The wazuh webui is very sensitve to spaces - so make sure you havent added any extra ones.

Wazuh is now installed and running.  Now for the extra stuff.

## Installation Footnotes

I already have Wazuh agents running on Windows Endpoints so they are going to start trying to connect to the server via the IP address that was set when the agent was installed
I already have installed auditd and sysmon on the server.  The first has default rules and a decoder so no problem, the latter does not
I already had groups set up so I need now to connect to the Wazuh webui and start making drastic changes to the configuration files

The first thing I do after logging into the wazuh webui is grab the master ossec.conf file and back it up
  - I simply do this by navigation to management/configuration and copy the contents of the file and using Visual code I create a backup in my documents
Since Wazuh on default had only 1 group called default it has only one configuration file which is empty so it is of no importance to me.

Now this only has to be done by me as I already had a Wazuh Server and I am constantly playing with it but now that I have reinstalled it, I have to rapidly do the following
  - Reestalish the groups - In my case - Windows, Windows-Test, default in my case is Linux - and although I have a configuration file for it the default will do for the time being.
  - As agents check back in and connect to the server - some where originally installed using default - they are going to run home to mumma and will connect to the default group again.  I need to move these back to Windows - such is life
  - You can NOT remove a agent from Default if it does not belong to another group - that is the way (of Wazuh)
  - So if your in production - better think good on the above.
  - So as I have been typing and re-establishing my groups and configuration file agents are logging in.  There are going to be issues in the first hour as I rapidly try to out configure the machine.  I loose - so I get errors.  KISS applies.
  - Might take a while so moving on.


## SOCFortress Rules Implementation

I have briefly meantioned SOCFortress annd I need to explain
  - SOCFortress are a commercial operation based in Texas
  - They are providing services to clients that are SOC related but are also into opensource and Wazuh in a major way.  After all who can afford a Microsoft Defender Endpoint protection at scale but the big guys
  - Wazuh is a labour intensive SEIM solution.  Its going to generate a lot of level 12 alerts - I have been playing with it for 6 months and it keeps me busy.
  - So any free help I can get is appreciated.
  - So I am going to reconfigure my Wazuh server with SOCFortress rules that are being maintained by Taylor Walton.
  - Reference: https://github.com/socfortress/Wazuh-Rules
  - Reference: https://socfortress.medium.com/
  - Reference: https://www.linkedin.com/in/taylor-walton-b772a6b2/
  - Reference: https://www.linkedin.com/company/socfortressmdr/

So lets begin
  - Using the terminal SSH on your daily driver connect back to your Wazuh server
  - Ensure you are in your local home directory (for convenience) and you will need GIT
  	- apt install git
	- curl -so ~/wazuh_socfortress_rules.sh https://raw.githubusercontent.com/socfortress/Wazuh-Rules/main/wazuh_socfortress_rules.sh && bash ~/wazuh_socfortress_rules.sh
	- Assuming no errors - your Wazuh Manager should restart and all is good
	- Since we have already installed auditd and sysmon for Linux upon restart you should see these alerts arriving in your webui although you may need to filter for these via the AgentID=000 as this should be your Wazuh Server

Problems
	- Wazuh lists - Wazuh list are not included in this script
	- The rules and decoders are however - I strongly suggest you read the script that you have run assuming you are following along
	- You will need to recreate this lists that are located here
		- /var/ossec/etc/lists
	- The following lists are of immedidate concern
		- common-ports
		- malicious-powershell
		- You will see errors after importing SOCFortress rules into your system via the Wazuh webui under management/logs
	- Wazuh is a complex little beast and as it receives updates via apt it will overwrite files as such you will need to learn about custom configurations a few places of note are here.
	
	| System location			| Importance							|
	|---------------------------------------|---------------------------------------------------------------|
	| 

		- /etc/wazuh-dashboard
		- /etc/wazuh-indexer
		- /usr/share/filebeat/module/wazuh/alerts/ingest
		- /var/ossec/etc/lists/
		- /var/ossec/etc/shared
		- /var/ossec/etc/lists
		- /var/ossec/etc/decoders
		- /var/ossec/ruleset/decoders
		- /var/ossec/etc/rules
		- /var/ossec/ruleset/rules
		- /var/ossec/active-response/bin
		- /var/log/

		
		


