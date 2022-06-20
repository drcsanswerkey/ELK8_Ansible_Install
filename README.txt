(OK Wow... I'm really sorry, my *.md formatting-fu is weak - If things don't work, view it in plain text until I can get this sorted out)


Intending to make walk-through videos for this, stay tuned... they'll show up on youtube and thinkific.

Install ELK Stack 8.x (Actually just EK for now)

TL/DR - aka - Quickstart
Run in the following order:
- Copy/rename group_vars/your_environment_name_here.yml & populate with your variables
- install_elasticsearch-8x.yml
- install_kibana_8x.yml
- (there's an 'optional' note for Kibana below which you'll probably want to do)
- Come back here and work through "Notes for installing Fleet" 
- If you don't do Fleet (above/below) then tweak next 2 playbooks as necessary)
- secure_kibana-https-01.yml
- secure_kibana-https-02.yml

That should be it!

=========

Only tested this as high as version 8.2.3 - fwiw
This role and associated playbooks install Elasticsearch and Kibana, version 8.x on RedHat or Debian servers (baremetal or vm)
Also includes instructions for getting Fleet server stood up. (what a pain that was! _-_)
There's a fair amount of hand-jamming to get Kibana and Fleet up and running - I eliminated as much of that as I could but still...
Hand-jam procedures are found below!

Requirements
------------
Assumptions for this to work:
  - Fundamental understanding of how Ansible playbooks/roles work
  - Linux environment (Elastic-agent part *may* work in a Windows environment, I didn't test it - ymmv)


Role Variables
--------------
Have used the convention "< some descriptive comment >" to denote variables which must be supplied

At a minimum, you'll need to populate the following files with your information: (might have to look at this in plaintext to make sense, until I figure out .md -_-)

├── group_vars
│   ├── <environs>.yml # currently named 'test', this file contains group vars for your environment, rename to something useful to you if you wish (whatever you name it, that's the answer at the vars prompt for 'environs' [sans .yml])
├── hosts # this is the Ansible hosts file, you know what to do...
├── roles
│   ├── install_elk_stack_8
│   │   ├── files             # Put the following files in here appropriate for your environment:
                              # elastic-agent-8.x.tar.gz, elasticsearch-8.x(deb or rpm), kibana-8.x(deb or rpm)

Uncomment appropriate ports (443/80) in /roles/install_elk_stack_8/tasks/install_kibana.yml in accordance with your setup and ansible_os_family

Examples and extra info
----------------
How to kick it off!
From the playbooks directory, run these as a user with sudo privs, which exists on the target servers
./install_elasticsearch-8x.yml -k -K #answer the prompts (`-i /path/2/hosts` if necessary)
./install_kibana-8x.yml -k -K #answer the prompts

N.B. Elasticsearch's installation's stdout gives you a password you'll need later and is written out to a file which you'll find in the role's 'files/output' directory with the naming convention: es_install-{{ ansible_hostname }}-{{ timestamp }}


License
-------
BSD

Author Information
------------------
Written by "Dr. C" of Dr. C's Answer Key
You can reach me at drcsanswerkey - at - protonmail dottus commus (de-latinized utique)
#updateme

------------------
### Begin Notes for installing Elasticsearch 8.x ###
------------------
 For 8.x
 This will install Elasticsearch and NOT start the nodes, required to properly form the cluster for the first time

To form the initial cluster:
  - Go to your first node and verify that cluster.initial_master_nodes: [{{ ansible_hostname }}] at the bottom of /etc/elasticsearch/elasticsearch.yml is uncommented and accurate (should be that first host!)
  - Start the node `sudo systemctl start elasticsearch`
  - If the prompt returns without errors, verify that it's running `sudo systemctl status elasticsearch`
  - Check its health here (credentials are elastic/<install generated password>) <- this is captured by this playbook - see below:
    https://<ip or hostname>:9200/_cluster/health?pretty=true
  - You should see: "cluster_name:<your cluster name>", "status:green", "number_of_nodes:1"
  - When it's green, generate an enrollment token for the other nodes by running the following on the initial node:
    /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node
  - You can also *reset* the elastic user's password now - but you can't *change* it until you get kibana up and running
    /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
  - Then go to the other nodes, reconfigure them to join the cluster like this:
    /usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>
  - Type "y" <enter> when prompted with: "Do you want to continue with the reconfiguration process [y/N]"
  - Repeat this process for all of the nodes you plan to join to the cluster.
  - Then start the other nodes, health should be green and show 3 nodes (or however many you have) once they all start (check url above)
  - N.B. if the last one is an even numbered node - you might have trouble, don't think ES clusters like even numbers of nodes.

------------------
### Begin Notes for installing Kibana 8.x ###
------------------
 - After succussfully installing, run the following command:
   systemctl status kibana
   And copy the url in the bottom line (this may not be strictly necessary going to the url might be enough)
   It should look something like this:
        http://<ip or hostname>:5601/?code=<numbers here>

(Optional) - This step addresses a random error I got at some point when using Kibana ##
 - Then, back on the Kibana server's cli run
   /usr/share/kibana/bin/kibana-encryption-keys generate
  (sample output below)
 - Copy the xpack.* lines it outputs into kibana.yml (I put them at the bottom) 
 - Restart Kibana
 - Go back to the url copied above and reload.
     http://<ip/name of server>:5601<random stuff>
     should bring up a box asking for an enrollment token

 - Go back to the Elasticsearch server and create an enrollment token for it (I think it's actually the same token as for the nodes, but something else may be happening under the covers so, probably best to just run it):
    /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

 - Back on the Kibana web interface...
 - Pop that sucker into the window asking for it, and click "Configure Elastic"

 - Now it's going to prompt you for a verification code... (it never ends...)
 - Run this on the Kibana server (cli) and run this to get the verification code:
    /usr/share/kibana/bin/kibana-verification-code
 - Enter the 6 digit code, click "Verify" and sit back and wait while it configures itself!

 - Log into Kibana (credentials are elastic/<elastic install generated password from initial node>):

Change password (optional)
 Click on the "e" in the upper right hand corner
 Select "profile"
 Enter current password and new password
 Click "Change Password"


------------------
### Begin Notes for installing Fleet 8.x ###
------------------
 - On an Elasticsearch server (I used the one on which I created the initial [single node] cluster - ymmv) ##
 - In /etc/elasticsearch/certs

### Resources used (etc... there were a ton, these are the ones I remembered to document)
#### https://www.elastic.co/guide/en/elasticsearch/reference/8.1/configuring-stack-security.html
### https://www.elastic.co/guide/en/fleet/8.1/secure-connections.html

Generate certs:

`cd /etc/elasticsearch/certs`

- Retrieve passwords for http.p12 and transport.p12:

`/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.keystore.secure_password`

`/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password`

- Convert files to pem format

`openssl pkcs12 -in /etc/elasticsearch/certs/transport.p12 -out cert.crt -clcerts -nokeys`

 (enter transport password from above and hit <enter>)

`openssl pkcs12 -in /etc/elasticsearch/certs/transport.p12 -out private.key -nocerts -nodes`

 (enter transport password from above and hit <enter>)


- Create CA

`/usr/share/elasticsearch/bin/elasticsearch-certutil ca --pem`

(accept defaults, or not...)

`mv /usr/share/elasticsearch/elastic-stack-ca.zip .`


- unzip elastic-stack-ca.zip

Use the certificate authority to generate certificates for Fleet Server. For example:
- On the dns and ip lines (in the sample "Fleet server cert generation command" below), specify the name and IP address of the Fleet Server. 
- Configure and run this "Fleet server cert generation command" (below) for each Fleet Server you plan to deploy.
- Upon running it - accept the defaults (or not...)
- This "Fleet server cert generation command" (below) creates a zip file that includes a .crt and .key file - Extract the zip file:
- Store the files in a secure location (or just proceed with the instructions that follow). You’ll need these files later to encrypt traffic between Elastic Agents and Fleet Server.

_______
 Begin Fleet server cert generation command (referenced above)
-------

/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
--name fleet-server \
--ca-cert /etc/elasticsearch/certs/ca/ca.crt \
--ca-key /etc/elasticsearch/certs/ca/ca.key \
--dns <hostname of your fleet server> \
--ip <ip of your fleet server> \
--pem
_______
End Fleet server cert generation command 
-------

mv /usr/share/elasticsearch/certificate-bundle.zip .  
unzip certificate-bundle.zip

Optional - You may need to do this temporarily in order to pull the certs off 


Recommended - Undo this step as soon as you're finished pulling the certs (below)


Might also want to do the cron trick here in case you jack up sshd_config
vi /etc/ssh/sshd_config
    - paste at the bottom:
PermitRootLogin yes

systemctl restart sshd
    - give root a password if it doesn't already have one
passwd  # as root


Recommended - Undo this step as soon as you're finished pulling the certs (below)

## In Kibana ##
Click on the hamburger menu on the upper left
Scroll down to the Management section
Select "Fleet" (should bring you to "Add a Fleet Server")
1. Fleet Server policy 1 (default should be fine)
    Click on "Create Policy" (not doing so will result in failure a few steps down :-")
2. Download and unzip/untar the elastic-agent somewhere on the fleet server (i.e. /opt)
3. Select "Production" (Back on the Kibana's "Add a Fleet Server" page)
4. Enter Fleet server info (and make sure 8220 is open on the fleet server - other Ansible playbooks in this role might have already done it)
    - click "Add host" (should receive verification)
5. Click "Generate a service token" (copy it somewhere, although they also pre-populate it in the instructions they generate)

## On the Fleet Server ##
- Ensure you have opened 8220 in your firewall
- Go to fleet server and run (as root):
rsync -avp <es-server-where-you-did-the-cert-stuff>:/etc/elasticsearch/certs /etc/pki/
- Go back to Kibana, which is hopefully still spinning and waiting for you - gives you a skeleton command to run (similar to the one below)
- Copy the command from Kibana, then in cli, go to wherever on the fleet you unzipped the elastic-agent, and run it
    (the cert paths below presume you followed the directions above and didn't put your certs elsewhere - adjust as necessary)
        
sudo ./elastic-agent install --url=https://<your fleet server url - prepopulated>:8220 \
  --fleet-server-es=https://<your es server url - prepopulated>:9200 \
  --fleet-server-service-token=<token - prepopulated> \
  --fleet-server-policy=<policy- prepopulated> \
  --fleet-server-es-ca-trusted-fingerprint=<fingerprint> \
  --certificate-authorities=/etc/pki/certs/ca/ca.crt \
  --fleet-server-cert=/etc/pki/certs/fleet-server/fleet-server.crt \
  --fleet-server-cert-key=/etc/pki/certs/fleet-server/fleet-server.key

You'll be asked:
Elastic Agent will be installed at /opt/Elastic/Agent and will run as a service. Do you want to continue? [Y/n]:

Type: y and enter
This may take a while
If successfull you'll receive the following on the Fleet server's cli where you ran the above command:

"Successfully enrolled the Elastic Agent."
"Elastic Agent has been successfully installed."

And...

## In Kibana ##
If successful the:
"Waiting for Fleet Server to connect..." 

will change to:
"Fleet Server connected"

Click "Continue"
Woot!

I have tested these procedures many many times, if you get a failure, look closely at the error messages (which in truth, are not always that helpful :-/) to see if they have any hints, then retrace your steps to ensure you didn't miss/fat finger something.

Alternatively, hit me up to schedule some consulting time! :-D


## Installing Integrations ##
(Looking at "Installed Integrations" you should see that Fleet has already been installed; Yay!)

Click on the hamburger menu on the upper left
Scroll down to the Management section
Select "Integrations"
Then scroll down and click on the "Fleet Server" panel (Which takes you to a page already populated with the fleet server you just added)

At a bare minimum, some others you probably want:
Endpoint Security
OSQuery Logs
OSQuery Manager
### End Notes for installing Fleet 8.x ###

Other Clients
- To connect clients you might need either /etc/elasticsearch/certs/http_ca.crt or its fingerprint - gotten by running:

  openssl x509 -fingerprint -sha256 -in config/certs/http_ca.crt

- Use the following command to retrieve the password for http.p12:

  bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password

- Use the following command to retrieve the password for transport.p12:

 bin/elasticsearch-keystore show xpack.security.transport.ssl.keystore.secure_password


