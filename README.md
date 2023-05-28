# Wazuh Project Learning Lab

#### Leon Scott -- Deployment and configuration of Wazuh
#### July 2023 
-------------------------------------------------------------------------------------------------------------------------------------------

The following contains the installation and reference notes for the configuration of a local Wazuh server used for Learning
I am not a professional employed in the field

HARWARE

An existing laptop has been utilised with 500Gb of storage, a generation 5 Intel i5 on 32Gb RAM
This handles the home network of about 10 Windows systems and linux servers used within the home Lab
It also has a battery - cheap mans UPS

SERVER Base

The base OS is Ubuntu 22.04
This has been chosen due to its extended support, ability to go Ubuntu Pro, and large software respository support
The minimal installation was chosen on installation to remove bloat

## Installation

Base operating system prep work

Set a hostname that makes sense to you
Set a password for the ROOT user - you will need the access
Update the system using terminal
Turn of hybernation and set the screen blanking to something realistic
Check network settings and ensure IPV6 is enabled - you want a static IP address - I manage this via the firewalla/router - it will get a static address.
Install openSSH - nothing fancy - its behind a firewall and wont be getting public access
Reset all the SSH keys
  ```shell
       sudo /bin/rm -v /etc/ssh/ssh_host_*
       sudo dpkg-reconfigure openssh-server && sudo systemctl restart ssh
  ``` 

Test the connection and log in via your daily driver to your ready to install Ubuntu OS
    - Using Windows terminal and log in to your ready to be setup server
    - Note if you are reading this this will allow for cut and paste which is extremly handy as we move along
- Install auditd
    - Reference: https://github.com/socfortress/Wazuh-Rules/blob/main/Auditd/auditd.conf
    - I will get to SOCFortress later - but lets just say I think he is a legend
    - Elevate to the ROOT user you setup earlier
    ```shell
        su
        apt install auditd
        cd /etc/audit/rules.d
        nano audit.rules
	
 - cut and paste the contents of the reference after removing the current contents (yep very handy is cut and paste)

     ```shell
        systemctl restart auditd
        auditctl -l 
	
- The above command should give you a swag load of rules
- Install sysmon for Linux (optional step - this will generate sysmon events for Linux that Wazuh can import later)
    - Reference: https://github.com/socfortress/Wazuh-Rules/tree/main/Sysmon%20Linux
    - Reference: https://github.com/Sysinternals/SysmonForLinux
    - Reference: https://github.com/Sysinternals/SysmonForLinux/blob/main/INSTALL.md
    - Using the 3rd reference - Install sysmon for linux and start it.  Becareful where you run this from its going to drop a file
     ```shell
        wget -q https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && dpkg -i packages-microsoft-prod.deb
        sudo apt-get update && apt-get install sysmonforlinux
        sysmon -i
        systemctl status sysmon
        
 You have now boosted your security logging.



## Installation Wazuh Indexer

I use the step by step configuration installation method - mainly to help me understand
I am not using SSL in any way at this point nor am I complicating the installation.
I did NOT name the servers in any way.  I left it as default.
  - Reference: https://documentation.wazuh.com/current/installation-guide/index.html
  - Reference: https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/installation-assistant.html
  - Simply follow the guide
  - If you are using UbuntuOS like me you will need to install curl
  ```shell
      apt install curl
      Run the below command somewhere convenient.  Its going to drop a few files that you will need later so best to know where they are.
      curl -sO https://packages.wazuh.com/4.4/wazuh-install.sh && curl -sO https://packages.wazuh.com/4.4/config.yml
  ```
  - Edit the configuration file and point it towards the IP address of your server
  - I am not going to do anything fancy here so its just IP address changes - if you are looking at a cluster arrangement with names - then refer to additional references
  ```shell
      bash wazuh-install.sh --wazuh-indexer node-1
      bash wazuh-install.sh --start-cluster
   ```
  - Wazuh Indexer is now installed and running

## Installation Wazuh Manager

I am again running the installation assistant way of doing this.  Since we already have the core files its a simple matter of.
  - Reference: https://documentation.wazuh.com/current/installation-guide/wazuh-server/installation-assistant.html
 ```shell
  bash wazuh-install.sh --wazuh-server wazuh-1
 ```
  - Remember later that its this process that installs Filebeat

## Installation of Wazuh Dashboard

I am again running the installation assistant way of doing this.  Since we already have the core files its a simple matter of.
  - Reference: https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/installation-assistant.html
  ```shell
  bash wazuh-install.sh --wazuh-dashboard dashboard
  ```
  At the end of this process you will see the admin username and password - make sure you copy this or you will be locked out the of webui and will need to reset the passwords of the entire system via a python script that Wazuh provides using a terminal
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
  ```shell
 	apt install git
	curl -so ~/wazuh_socfortress_rules.sh https://raw.githubusercontent.com/socfortress/Wazuh-Rules/main/wazuh_socfortress_rules.sh && bash ~/wazuh_socfortress_rules.sh
  ```
- Assuming no errors - your Wazuh Manager should restart and all is good
- Read the SOCFortress scipt - give you an insight into what has been done
- SOCFortress rules are sent to the /var/ossec/etc/rules
- Default rules exist here /var/ossec/ruleset/rules
- You may need alter the main configuration file ossec.conf to excempt wazuh rules depending on what is contained in the readme of the SOCFortress respository under excemptions
- Since we have already installed auditd and sysmon for Linux upon restart you should see these alerts arriving in your webui although you may need to filter for these via the AgentID=000 as this should be your Wazuh Server
- The SOCFortress script contains code for their decoders.  Refer to table below for the location of these.   The ossec.conf file all allows for excemptions to be set.

Problems
- Lists - the SOCFortress script does not allow for scripts to be imported
- There are three of note
- common-ports, malicious-powershell and another to do with bash
- the following code is general but you will need to modify all files you alter in this directory to the correct permissions and ownership
```shell
	cd /var/ossec/etc/lists
	nano common-ports
	chown wazuh:wazuh common-ports
	chmod common-ports
	systemctl restart wazuh-manager
```
- You will need to do this for every file you create in this directory.
- After restarting the manager you will see the associated CBD file
- Important note - just because the manager restarted and the CBD list created does not mean the file is correct.  Check the lists under managerment in the webui.  I have had issues with this numerous times.

Important file locations

The following important file locations should be noted - you will need to learn these and the ramificaitons of creating files within these locations mainly in relation to the correct ownership and rights

| System location                                    | Importance						                     |
|----------------------------------------------------|-------------------------------------------------------------------------------|
| /etc/wazuh-dashboard                               | Wazuh Dashboard files                                                         |
| /etc/wazuh-indexer                                 | Wazuh Index files - Filebeat installed with these                             |
| /usr/share/filebeat/module/wazuh/alerts/ingest     | pipeline.json file located here - responsible for geotagging                  |
| /var/ossec/etc/lists/                              | List files - used in conjunction with Rules - important                       |
| /var/ossec/etc/shared                              | Used within groups                                                            |
|                                                    | Placing files within the correct groups allows for dissemination to endpoints |
|                                                    | Files of anytype supported by Wazuh can be placed in these directories        |
| /var/ossec/etc/decoders                            | Decoders used by Wazuh to read various ingested logs correctly                |
|                                                    | It is recommended to get these confirmed by third parties                     |
| /var/ossec/etc/rules                               | Custom rules - SOCFortress rules go in here for a start                       |
|                                                    | You can normally edit these rules via webui                                   |
| /var/ossec/ruleset/rules                           | Wazuh rules - rights can be modified to allow editing via webui               |
| /var/ossec/active-response/bin                     | Where files go that are marked as active- response within config files        |
| /var/log                                           | Location of log files that can be imported via decoders                       |

Base installation of Wazuh is finished.


## Post Installation

If you read the Wazuh blog posts there are lots of interesting work
Becareful following the blog posts - they tend to leave the rules in the default rules range and this will cause problems.  I suggest you use your own number conventions
These can be found using the wazuh manager logs.
They also have a nasty habit of not meantioning how to disseminate scripts, bat files, powershell or files to your endpoints
I will fill this out later

#### Remote execution of Wazuh Commands and SCA Commands

By default Wazuh does not allow for the execution of remote commands to endpoints.  This needs to be configured.
Reference: https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/wodle-command.html#centralized-configuration
Enable these via the following

```shell
	cd /var/ossec/etc
	nano local_internal_options.conf
	nano internal_options.conf
	systemctl restart wazuh-manager
```

Append the following commands to the first file - alter the commands in the second file.  Note the second file will overwrite on updates from apt

```shell
	wazuh_command.remote_commands=1
	sca.remote_commands=1
```


#### Geotagging Normalisation - via ingest pipeline

By default certain fields will generate a geotag for country, city, coordinates and so on.  After you start altering wazuh from default you may find fields you wish to add this data for.
For example sysmon Event 3 log events both in Windows and Linux.
To to this you will need to alter a file called pipeline.json.  There are other methods and this might require some research.
I am going to use the wazuh method for the time being.
Reference: https://github.com/wazuh/wazuh/blob/e4fcce131086ecce93623379b3eb7cfa2166b480/extensions/filebeat/7.x/wazuh-module/alerts/ingest/pipeline.json

```shell
	cd /usr/share/filebeat/module/wazuh/alerts/ingest
	cp pipeline.json pipeline.bak
	nano pipeline.json - alter the file very carefully.  I suggest you strongly use a backup file kept in your local files.
	/etc/filebeat#
	cp filebeat.yml filebeat.bak
    	nano filebeat.yml - alter the file to contain the following lines  - filebeat.overwrite_pipelines: true setup.template.overwrite: true
	systemctl restart filebeat
	systemctl status filebeat
```

Access the webui and confirm all is still working.  If it breaks, restore your back up files and try again.  Its probably your formatting in the pipeline.json file
Ensure you are getting the updated data in your events in the wazuh webui

#### Wazuh VMWare Integration 

Fill this in
- Reference: https://wazuh.com/blog/monitoring-vmware-esxi-with-wazuh/ 

#### Domain stats 

Fill this in
- Reference: https://github.com/MarkBaggett/domain_stats
- Reference: https://github.com/socfortress/Wazuh-Rules/tree/main/Domain%20Stats

#### SOC Fortress API intergration

Fill this in

#### Miscellaneous 

Fill this in
  - Rule alternation and fine tuning
  - SOCFortress respository fork and pull request modification
  - 
         
## Post installation notes

I need to get on with helping Taylor and that requires some time and testing.  I have found the following issues
 - Reference: https://wazuh.com/blog/building-ioc-files-for-threat-intelligence-with-wazuh-xdr/ 
 - Not tested

Backup shell script
Update SOCFortress rules script - more about where it gets the rules rather than the script
Review of cron jobs
Investigation of Firewalla API calls and mechanisms and integration into Wazuh
Investigation of Windows Firewall rules scripts


        
Note - a lot of fine tuning is needed
