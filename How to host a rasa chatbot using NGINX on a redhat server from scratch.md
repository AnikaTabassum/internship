# How to host a rasa chatbot using NGINX on a redhat server. from scratch

<h2>Introduction</h2>

<h3>RASA</h3>
<p>Rasa is an Open Source is a machine learning framework to automate text- and voice-based assistants. We have used rasa to build a conversational chatbot.</p>

<h3>NGINX </h3>
<p>A Nginx HTTPS reverse proxy is an intermediary proxy service which takes a client request, passes it on to one or more servers, and subsequently delivers the server's response back to the client. We are using NGINX here to host the application.</p>

<h3>Part 1: Installing RASA on a the machine</h3>
To install rasa, we have to give the following commands. 

<b>Step 1: Install python3.6 </b>
<pre>
yum install python3
</pre>

<b>Step 2: Install the required development tools</b>
<pre>
sudo subscription-manager repos --enable rhel-7-server-optional-rpms

</pre>

<b>Step 3: Create a virtual environment </b><br>We must create a virtual environment where the nlu and core of rasa will be installed. To create the virtual environment we have to give the following command:
<pre>
sudo apt-get install -y python3.6-venv
</pre>

<b> Step 4: Now we have to create a directory where we are going to make our virtual environment. Let's name it chatBot and set up a virtual environment there named "rasaChatBot" </b>
<pre>
mkdir chatBot
cd chatBot
python3.6 -m venv rasaChatBot
</pre>

<b> Now we have to activate the virtual environment by the following command: </b>
<pre>
. rasaChatBot/bin/activate
</pre>

Now that we have our virtual environment set perfectly, we will start installing packages.</br>
apt-get is not installed in redhat. So to install and give proper permission the following commands must be given:
<pre>
curl https://raw.githubusercontent.com/dvershinin/apt-get-centos/master/apt-get.sh -o /usr/local/bin/apt-get
chmod 0755 /usr/local/bin/apt-get
</pre>
As, rasa has a dependency issue in redhat, rasa must be installed with all the dependencies. You have to create a file "requirements.txt" and install all the modules with specific version from there. First we need to update pip to 19.3.1 with
<pre>
pip install --upgrade pip==19.3.1
</pre>
The requirements in the 
<a href="https://github.com/AnikaTabassum/internship/blob/master/requirements.txt">requirements.txt </a> file. Download the file and install the requirements.</br>
<b> Step 5: Install all the requirements with-</b>
<pre>
pip install -r requirements.txt -y
</pre>
Now we have a perfect environment for rasa. We have to move to the next part to install nginx and host the application on nginx

<h3>Part 2: Installing NGINX</h3>
<b> Step 1: Install Nginx with the following command:</b>
<pre>
sudo apt update
sudo apt install nginx
</pre>

<b> Step 2: Allow nginx</b>
<pre>
sudo ufw allow 'Nginx HTTP'
sudo ufw status
</pre>

<b> Step 3: start nginx with the following command: </b>
<pre>
sudo systemctl start nginx
</pre>
 
Now open the configuration file of nginx with:
<pre>
sudo nano /etc/nginx/conf.d/default.conf
</pre>

and make the following changes:
<pre>
server_name  [your server ip];
root /data/myChatBotWeb;
</pre>
The server name must be your ip address which you can check by giving the following command:
<pre> ifconfig</pre>
And the root folder is the path where your application, which  you want to host, is.
This root folder should be accessible. It's safe to make a new directory with permission and keep your application there. 
<pre>
sudo mkdir /data
</pre>
And then copy your web application directory in that directory with:
<pre>
sudo cp -r /your/path/to/application /data/myChatBotWeb
</pre>
Now, if you start nginx with :
<pre>
sudo systemctl restart nginx
</pre>And hit your ip address from a browser you should be able to see the homepage of your application.

Now that your nginx is running, it's final part to run rasa as a service in the background 
<h3>Running rasa as e service</h3>
As you know, to run a rasa chatbot, we have to run rasa action server and rasa ui. So we have to make two shell scripts and run these two scripts from two separate services.
<br>
First, we are going to create a script to run the action server. Open the script file with:
<pre>
sudo nano /usr/local/bin/action.sh
</pre>
Add there the following lines:
<pre>
#!/bin/bash
cd /data/rasabot; 														        //path where the chatbot is
. /your/path/to/the/virtual/environment/bin/activate							//activate environment
rasa run actions													            //run action server
</pre>

Next, we need to create a script to run the ui. Open the script file with:
<pre>
sudo nano /usr/local/bin/runUi.sh
</pre>
<pre>
#!/bin/bash
cd /data/rasabot; 																//path where the chatbot is
. /your/path/to/th/virtual/environment/bin/activate											//activate environment
python -m rasa run --m ./models --endpoints endpoints.yml --port 5005 -vv --enable-api --cors "*"     //run rasa ui
</pre>

Now, you have to create two services to run two script files.
To create a service for action, give the following command:
<pre>
sudo nano /etc/systemd/system/botAction.service 
</pre>
And add the following lines:
<pre>
[Unit]
Description=chatbot demo service
After=network.target
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
RemainAfterExit=yes
RestartSec=1
ExecStart=/usr/local/bin/action.sh
[Install]
WantedBy=multi-user.target

</pre>
Which runs the action.sh script file as a service, which runs the action server.</br>
Next, we need to make the service file for UI by giving the following command:

<pre>
sudo nano /etc/systemd/system/botUI.service 
</pre>
And add the following lines:
<pre>
[Unit]
Description=chatbot demo service
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RemainAfterExit=yes
RestartSec=1
ExecStart=/usr/local/bin/runUi.sh
[Install]
WantedBy=multi-user.target
</pre>
This service will run rasa ui as a service </br>
Now we have to enable and start the services. To start the botAction service give the following commands:
<pre>
sudo systemctl enable botAction
sudo systemctl daemon-reload
sudo systemctl restart botAction
sudo systemctl status botAction
</pre>
To start the botUI service give the following commands:
<pre>
sudo systemctl enable botUI
sudo systemctl daemon-reload
sudo systemctl restart botUI
sudo systemctl status botUI
</pre>
now restart nginx, hope your application is successfully hosted:
<pre>
sudo systemctl restart nginx</pre>
Now if you open your IP address in a browser, you will be able to chat with your rasa application. But if your port is blocked, you can open it with the following commands:
<pre>
sudo /sbin/iptables -I INPUT -p tcp -m tcp --dport 5005 -j ACCEPT
sudo /sbin/iptables-save
firewall-cmd --zone=public --add-port=5005/tcp
firewall-cmd --zone=public --add-port=5005/tcp --permanent
firewall-cmd --reload
</pre>
Now your rasa is hosted successfully. Hope you are able to chat perfectly.
