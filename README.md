# Somleng IVR with RapidPro and GoIP (simBox)
This documentation refers to the original somleng deployment page [Somleng](http://www.somleng.org/)

The Somleng (voice in Khmer) Project is a collection of open source telephony tools which can be used to build powerful Voice applications. The goal of the project is to break down the economic and accessibility barriers to building telephony applications. Read more about Somleng

The somleng platform is composed of 4 main software components or tools:
PostgreSQL RBMS
Freeswitch open source soft-switch platform used to power voice, video, and chat communications [FreeSWITCH](https://freeswitch.com/)
Twilreapi an Open Source implementation of Twilio's REST API [Twilreapi](https://github.com/somleng/twilreapi).
and then somleng-adhearsion an adhearsion app with twilio support feature [Somleng-Adhearsion] (https://github.com/somleng/somleng).

The somleng platform is connected to rapidPro [rapidPro](http://docs.rapidpro.io/) used to build the call flow through the twilML channel and the call routing to the GSM channel is handle by the GoIP [Freeswitch+GoIP](https://freeswitch.org/confluence/display/FREESWITCH/Goip+HowTo).

## Installation
The OS used to deploy all these components is Ubuntu 14.04 LTS.
The for main somleng components are deployed as dockers to allow user to easily manage and deploy them without going deep in their configuration.
To get more information on the somleng docker installation go to [Somleng docker installation] (Ref:https://github.com/somleng/somleng-project/blob/master/docs/GETTING_STARTED.md).
Rapidpro can  be installed as described by this link [rapidpro server development](http://rapidpro.github.io/rapidpro/docs/development/)	
We have also observed a problem of call not triggered sometimes when the last version, so we have pulled a specific version of rapidPro that trigger the call using twilML channel.
For the GoIP appliance, there is no installation to do, but some configuration must be done in the freeswitch and GoIP to allow freeswitch to bind the configuration to GoIP to be able to handle the VoIP calls and to route it to the GSM network
For additionnal information on freeswitch configuration please visit the link [Freeswitch on ubunut 14](https://freeswitch.org/confluence/display/FREESWITCH/Ubuntu+14.04+Trusty)
It has been observed that sometimes the same software components that uses the same port are installed on the system when deploy rapidpro and somleng on the same servers. So it is advise to deploy rapidPro and somleng on different computer to avoid this conflict (just in case you want to make your life easy :-)).
### Somleng installation
Prepare a computer with Ubuntu 14.04 LTS and install the following components and assign a static IP address. Keep in mind that the rapidPro computer and the GoIP device must be in the same network segment.
#### Install docker components to manage docker processes
Install docker CE [Get Docker CE for Debian](https://docs.docker.com/install/linux/docker-ce/debian/)
```sh
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88 #To verify that you have the good fingerprint with the last 8 characters
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce
sudo docker run hello-world #To verify that docker CE is configured correctly and work
```
Install docker compose
```ssh
sudo curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version #test if docker compose it is running
```
#### Install now somleng
```ssh
git clone https://github.com/somleng/somleng-project --depth 1 
cd somleng-project
sudo docker-compose pull
docker-compose run --rm -e INCOMING_PHONE_NUMBER="{\"phone_number\":\"1001\",\"voice_url\":\"http://demo.twilio.com/docs/voice.xml\"}" twilreapi /bin/bash -c './bin/rails db:setup' #Setup and seed somleng API, 1001 is the VoIP freeswitch profile used to connect to GoIP
```
When performing the seed of somleng API, the postgres DB object are created and the account SID and account auth token are created. Write them down somewhere, you will need them for twiML channel configuration

Start somleng service in the somleng repository
```ssh
sudo docker-compose up
```
Once somleng is runing, you could check and visualize the docker processes of the 4 with
```ssh
sudo docker ps
```
#### Simulate an inbound call
Once installed, we can test in the plate-forme is working with the default configuration by making an inbound call.
Open a new terminal and run these commands:
```ssh
cd somleng-project
IFS=: read ACCOUNT_SID AUTH_TOKEN <<< $(sudo docker-compose run --rm -e FORMAT=basic_auth twilreapi /bin/bash -c './bin/rails db:seed') && echo "Account SID: $ACCOUNT_SID" && echo "Auth Token:  $AUTH_TOKEN"
curl -XPOST http://<IP twilreapi docker>:3000/api/2010-04-01/Accounts/$ACCOUNT_SID/Calls.json -d "Method=GET" -d "Url=http://demo.twilio.com/docs/voice.xml" -d "To=+243890000001" -d "From=1001" -u $ACCOUNT_SID:$AUTH_TOKEN # Normally your computer should ring,make sure that your speaker or headset are not muted
```
To have the <IP twilreapi docker>:
```ssh
docker inspect <container ID> # under "NetworkSettings" within the JSON response, find "IPAddress" attribute
```
To responde to the call, open a new terminal and run these commands
```ssh
sudo docker-compose exec linphone /bin/bash -c 'linphonecsh generic answer' #If your computer is connected to the internet,after the ring you should listen to the hold on music
```
To terminate the call
```ssh
sudo docker-compose exec linphone /bin/bash -c 'linphonecsh generic terminate'
```
### Customized somleng for GoIP installation
Freeswitch, twilreapi and somleng-adherson as been modified to allow the to make outbound calls through the simBox. The process consist to rebuild the docker images of the 3 tools and to point to them in the docker-compose.yml file instead of using original images from dwilkie dockerhub repository.The original tools image for somleng are under dwilkie repositories[dwilkie repo on docker-hub] (https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=0&q=dwilkie&starCount=0)
Stop the default somleng project (Ctrl+C) 
#### Rebuild the freeswitch docker image
```ssh
cd
git clone https://github.com/gerard-bisama/freeswitch-rayo-goip.git
cd freeswitch-rayo-goip
sudo docker build -t gbfree . #build the new image
sudo docker run -p 5060:5060/udp gbfree #try to run the new docker to see if it works. to stop Ctrl+C
```
#### Rebuild the twilreapi docker image
```ssh
cd
git clone https://github.com/gerard-bisama/twilreapi-goip.git
cd twilreapi-goip
sudo docker build -t gbtwilreapi .
```
#### Rebuild the somleng-adhearsion docker image
```ssh
cd
git clone https://github.com/gerard-bisama/somleng-adhersion-goip.git
cd somleng-adhersion-goip
sudo docker build -t gbsomleng .
```
Then update docker-compose.yml compose file in the somleng-project repository to take into account the new built images. So replace the content of your docker-compose.yml file by the the one in this repository.

Start somleng service in the somleng repository
```ssh
cd somleng-project
sudo docker-compose up
```
In an other terminal,check is all the docker image are running with
```ssh
sudo docker ps
```
### Configure now GoIP
#### Login the GoIP device to the somleng-freeswitch tools
Insert SIM cards in the GoIP and configure the web built-in server as indicated in the GoIP user manual [GoIP user manual](https://www.lojamundi.com.br/download/gateways-gsm/GoIP/GoIP-16.pdf). Read this document for more option.
Just as a remind, Insert your SIM in the device as indicated for your model, connect you computer to the PC port. Change your computer network setting to be in the same network as the preset network for the GoIP. For example set you computer to 192.168.8.2. Open your browser and enter the 192.168.8.1 to connect to the built-in device web server. Enter the default admin username and password: admin/admin then assign a static IP address to your device.
GoIP>Configuration>Network
LAN Port:Static IP	
IP Address:192.168.0.100 #this IP address should be in the same network with the somleng server
Subnet Mask(optional):255.255.255.0
Default Route:192.168.0.1
Primary DNS:192.168.0.1
save and reboot your device (GoIP>tools>reboot)

To activate the SIM channel, connect now the GoIP to the LAN port. And ping the device from the somleng computer to check if there connextion betwenn them.
To check if the channel is activated and network config saved, send the SMS "###INFO###" to one of the SIM card.

Now login freeswitch to the GoIP

goipSetup -> Call Setting -> Config Mode -> Sip Phone
Config Mode -> Trunk Gateway Mode
SIP Trunk Gateway1:somleng server IP addr
Phone Number: 1001 (Freeswitch userid)
Authentication :1001 (Freeswitch userid)
Password: 12345(Freeswitch password)
check the line that you want to use for this service. Then save.

Then goto inbound call routing>
CID Forward Mode: Use CID as SIP Caller ID	
Configuration Forwarding to VoIP Number:10000

Then allow DTMF Signaling for IVR
Configuration>Advanced VoIP>DTMF Signaling :Outband
Configuration>Advanced VoIP>Outband DTMF type: SIP INFO #leave other "none" by default

Set talk time limit to 2 minutes
> GoIP>Configuration>SIM>Gprs registration disable
> GoIP>Configuration>SIM>Module Registration when limit runout (ok)
> GoIP>Configuration>SIM> SIP Registration when limit run out (ok)
> GoIP>Configuration>SIM> Drop Call when Talk time limit expires (ok)
> GoIP>Configuration>SIM> Talk Time limit (m)/call (=1) only integer is accepted
> GoIP>Configuration>SIM> Talk Time adjust(s)/call (=1)
Save then reboot

#### Test to make outbont call to the GSM network
Make sure that the somleng server is running. Open a new terminal and run the command
```ssh
IFS=: read ACCOUNT_SID AUTH_TOKEN <<< $(sudo docker-compose run --rm -e FORMAT=basic_auth twilreapi /bin/bash -c './bin/rails db:seed') && echo "Account SID: $ACCOUNT_SID" && echo "Auth Token:  $AUTH_TOKEN"
curl -XPOST http://<somleng server IP>:3000/api/2010-04-01/Accounts/$ACCOUNT_SID/Calls.json -d "Method=GET" -d "Url=http://demo.twilio.com/docs/voice.xml" -d "To=+243xxxxxxxx" -d "From=1001" -u $ACCOUNT_SID:$AUTH_TOKEN
```
### Rapidpro installation
#### Install rapidpro dev server
Follow the installation procedure described for rapidpro development server [rapidpro installation](http://rapidpro.github.io/rapidpro/docs/development/)
Somleng works well for a certain version of rapidpro, so after cloning the rapidpro source from github, check it out to a specific version
```ssh
git clone git@github.com:rapidpro/rapidpro.git
cd rapidpro
git checkout 49522433c62000ad9ee82f8b06d9e5732b4c2e64
```
#### Install stunnel to use ssl with django
This allow to create a secure tunnel for the rapidpro api operation. Ngrok could be used as well but since audio file are bigger than test, ngrok create a delay between the call pickup and the reception of the voice message, also timeout occurs often will routing the request to internet. For additionnal information on the [use of ssl with django](https://gist.github.com/claudiosanches/7012524)
```ssh
sudo apt-get install stunnel
ls -l /etc/stunnel/
cd /etc/stunnel/
sudo -s
openssl req -new -x509 -days 365 -nodes -out stunnel.pem -keyout stunnel.pem
openssl gendh 2048 >> stunnel.pem
cd /pathtorapidpro/rapidpro/
mkdir stunnel
cd stunnel/
sudo cp /etc/stunnel/stunnel.pem .
sudo chown gerard:gerard stunnel.pem
touch dev_https #in rapidpro/stunnel
```
Then add the following content to the file
```ssh
"
pid=

cert = stunnel/stunnel.pem
#sslVersion = SSLv3; this has been comment out since it raised exception
foreground = yes
output = stunnel.log

[https]
accept=8443
connect=8000
TIMEOUTclose=1
"
```
#### Run the rapidPro with stunnel
```ssh
virtualenv env
source env/bin/activate
stunnel4 stunnel/dev_https &
python manage.py runserver &
```
Now you can try to access rapidpro with https://localhost:8443 and with http://localhost:8000
Then change the rapidPro setting
```ssh
gedit temba/settings.py
"
HOSTNAME = '192.168.0.102:8443' #192.168.0.102 refers to the IP address of rapidPro server
TEMBA_HOST = '192.168.0.102:8443'
ALLOWED_HOSTS = ['localhost', 'temba.ngrok.io','192.168.0.102']
"
```




































 
