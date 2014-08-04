---
layout: page
title: "Getting Started"
description: ""
---
{% include JB/setup %}

### Installation

1.You’ll need an Ubuntu 14.04 host to perform the installation. The first step after OS installation is installing docker. The quickest way to install docker is the following:

<pre>
# wget -O - http://get.docker.io | sudo su
</pre>

The install script will add the docker repository to apt and install docker. 

2.After it is installed, you must either run docker commands as root or add your user to the "docker" group. You can perform this task using:

<pre>
# sudo usermod -aG docker $USER
</pre>

… with $USER being the username you want to add to the docker group. You’ll need to log out all sessions of the user before the change will take effect. 

3.Next, you will need to install serviced.

_Note: Please make sure you have set the correct hostname/IP in the host file. Control Center will not function correctly if the hostname is mapped to the loopback adapter IP - 127.0.0.1_

<pre>
# sudo sh -c 'echo "deb [ arch=amd64 ] http://apt.zendev.org/apt/ubuntu trusty universe multiverse" \
  > /etc/apt/sources.list.d/zenoss.list'
# sudo apt-get update
# sudo apt-get install serviced
</pre>

Serviced should now be installed in /opt/serviced. There may be an initial pause before serviced starts up as it loads its internal services images into the running docker instance. Once that is complete you should see the Control Center serving up an AngularJS UI at [https://[HOSTIP]]/]. 

If you don’t see serviced starting, start it via the following upstart command: 
<pre>
# sudo start serviced
</pre>

4.You will need to log in using a user that is in the **_wheel _**or **_sudoers_** group. Authentication is performed using PAM; you may need to configure PAM appropriately if you use Kerberos or LDAP for authentication.

If in doubt about whether Control Center has started properly, check the log ->/var/log/upstart/serviced.log and also check if the host is listening on TCP 443:
<pre>
# netstat -an | grep [:]443
</pre>

### Deploying Applications

Next, you will need to deploy an application. Applications are deployed from an application template definition.

Below is the basic process on how you can take a Docker app and manage it via Control Center. The tutorial will use Logstash, Elasticsearch and Kibana as the target application stack:

####Step 1 - Preparation

For this how-to-guide we will be using a popular ELK (Elasticsearch, Logstash and Kibana) Docker image (qnib/elk).

If you are adventurous, you may want to try one of the Solr images available or any other application of your own choosing. 

Docker Hub hosts many prebuilt ready to run applications -> (https://hub.docker.com/) OR build your own image using Docker make, Packer or another Docker make tool. 

Download target image via Docker: docker pull qnib/elk 

You may use Control Center to do all of the heavy lifting, including downloading the image (which simply invokes Docker to pull the image).

The caveat is that you will require a complete service template ahead of time. You can find one here, ready to run if you can’t wait: [https://gist.github.com/mmaloney/5b990225d614ca413d33](https://gist.github.com/mmaloney/5b990225d614ca413d33)

####Step 2 - Test Target Application

Test target application in Docker first in interactive mode to make sure it runs as expected.

You may need to connect to the running container to see what is really going on. A popular method is described below, nsinit is the preferred Docker method but requires additional dependencies. 

Nsenter example command: 
<pre>
# PID=$(docker inspect --format '{{.State.Pid}}' my_container_id)
# nsenter --target $PID --mount --uts --ipc --net --pid
</pre>
Also, you can run a shell command on startup to access the container directly. This may be required for any customization of configuration files, though Control Center takes care of this for you by injecting any required configuration at application runtime.

####Example: 
<pre>
# docker run -i -t &lt;image_name&gt; bash
</pre>

####Step 3 - Create an Application template 

It’s now time to build a Control Center template. Below is a simple walk-through of how to create an ELK (Elasticsearch, Logstash and Kibana) template in Control Center. 

First you will need to setup a suitable application template folder structure on your Control Center host. 

####Example:

####Root Application Template Folder
~:/LogAnalyzer  

_Reference sample template:_ 

[https://gist.github.com/mmaloney/c4e4c653300515e8b679](https://gist.github.com/mmaloney/c4e4c653300515e8b679)

_Elasticsearch Service Template Folder_

~:/LogAnalyzer/elasticsearch

_Reference sample template:_ 

[https://gist.github.com/mmaloney/637a0b8f5252ea057c5b](https://gist.github.com/mmaloney/637a0b8f5252ea057c5b)

_Logstash Service Template Folder_
~:/LogAnalyzer/logstash

_Reference sample template_

[https://gist.github.com/mmaloney/1392be0332e3e8089adf](https://gist.github.com/mmaloney/1392be0332e3e8089adf)

_Kibana Search Template Folder_
~:/LogAnalyzer/kibana

_Reference sample template_

[https://gist.github.com/mmaloney/de150ef7130a8e05b85c](https://gist.github.com/mmaloney/de150ef7130a8e05b85c)

Also, you should generate any required configuration files for the target applications. 

Control Center injects configuration at runtime as opposed to mutating one or more configuration files within the container. This means that you will need to use the special configuration folder for each service and use a relative path. 

Simply copy the requisite config files to the appropriate service/config folder using the configuration file path as seen from within the container (relative path) and you should be good to go.

_Note: "-CONFIGS-" is a special folder name used only for housing application configuration files, based on the relative path of the configuration files. Another folder called “-FILTERS-” is used for application log filters used by the Control Center version of Logstash._

####Example:

_Elasticsearch Service Config Files_

~:/LogAnalyzer/elasticsearch/-CONFIGS-/elasticsearch/config/elasticsearch.yml

[https://gist.github.com/mmaloney/5e1da5daa58b70a3a671](https://gist.github.com/mmaloney/5e1da5daa58b70a3a671)

~:/LogAnalyzer/elasticsearch/-CONFIGS-/elasticsearch/config/logging.yml

[https://gist.github.com/mmaloney/0f44115aa742361d39de](https://gist.github.com/mmaloney/0f44115aa742361d39de)

_Logstash Service Config Files_

~:/LogAnalyzer/logstash/-CONFIGS-/etc/logstash/conf.d/syslog.conf

[https://gist.github.com/mmaloney/90fd5047156108b944b9](https://gist.github.com/mmaloney/90fd5047156108b944b9)

_Kibana Service Config Files_

~:/LogAnalyzer/lkibana/-CONFIGS-/etc/nginx/nginx.conf

[https://gist.github.com/mmaloney/3931911b551837132ff8](https://gist.github.com/mmaloney/3931911b551837132ff8)

_Note: You do not have to use the configuration file feature in Control CEnter but it does make it much easier to quickly iterate on an application whether it be to modify port mappings, configure more workers or shard the application service etc._

The next step will be to create several basic template files:

_Logstash example_

[https://gist.github.com/mmaloney/c88fcbb6fd03e3066684](https://gist.github.com/mmaloney/c88fcbb6fd03e3066684)

_Zenoss example_

[https://gist.github.com/mmaloney/6e6df605d55fe0d40491](https://gist.github.com/mmaloney/6e6df605d55fe0d40491)

_Memcached example_

[https://gist.github.com/mmaloney/0681871a89df24fd2615](https://gist.github.com/mmaloney/0681871a89df24fd2615)

_Rabbitmq example_

[https://gist.github.com/mmaloney/d2083f87f8f534c3216e](https://gist.github.com/mmaloney/d2083f87f8f534c3216e)

####Step 4 - Application template authoring

Create target folder structure on master Control Center host with the following schema:

~:/&lt;Application Name&gt;/

~:/&lt;Application Name&gt;/&lt;Service Group Name&gt;/

~:/&lt;Application Name>/&lt;Service Group Name&gt;/&lt;Service Name&gt;/

~:/&lt;Application Name>/&lt;Service Group Name>/&lt;Service Name&gt;/-CONFIGS-/

_Note: You will need to specify the PWD in order to create the "-CONFIGS-” folder i.e. mkdir ./-CONFIGS-_

~:/&lt;Application Name&gt;/&lt;Service Group Name&gt;/&lt;Service Name&gt;/-CONFIGS-/<APPLICATION CONFIG PATHS&gt;/

Next copy any configuration files required for the application into the expected application config path you created earlier. 

Create a service template called "service.json" in each service folder. Below is the basic anatomy of a service template, it is not meant to be exhaustive but enough to get you going.
<pre>
{
    "Command": "/opt/elasticsearch/bin/elasticsearch",
    "Endpoints": [
        {
            "Name": "http",
            "Application": "http",
            "PortNumber": 9200,
            "Protocol": "tcp",
            "Purpose": "export"
         },
        {
            "Name": "transport",
            "Application": "transport",
            "PortNumber": 9300,
            "Protocol": "tcp",
            "Purpose": "export"
        }
    ],
    "ImageID": "dockerfile/elasticsearch:v2",
    "Instances": {
        "min": 1
    },
    "RAMCommitment": 536870912,
    "Launch": "auto",
    "LogConfigs": [
        {
            "filters": [
                "error",
                "search"
            ],
            "path": "/var/log/elasticsearch.log",
            "type": "elasticsearch"
        }],
        "Name": "elasticsearch",
    "Services": [],
    "Tags": [
        "daemon"
    ],
"HealthChecks": {
        "running": {
            "Script": "echo hello",
            "Interval": 2
        }
    }
}
</pre>

_Note: If you have more than one configuration file in your application, you must explicitly define it in the service.json at the service child level. If you do not, your application will not start correctly._ 

####Step 5 - Compile and deploy Application template

One you are satisfied with your application templates and have tested the application manually, you are now ready to compile all of your service definitions into an application template.

Run the following command:
<pre>
# serviced template compile &lt;template folder&gt; &gt; &lt;apptemplate&gt;.json
i.e. serviced template compile ./LogAnalyzer &gt; loganalyzer.json
</pre>
If you don’t see any errors, then you are ready to move on to deploying the application.

####Step 6 - Deploying an application with Control Center**

_Note: Make sure you have added a host to Control Center prior to starting the application_

This can part can be done via the CLI or UI:

####CLI workflow: 

1.serviced template add &lt;apptemplate.json&gt;

2.serviced deploy &lt;templateId&gt; &lt;Host:4979&gt; &lt;resource pool id&gt; &lt;deployment id&gt;

3.serviced service start &lt;serviceid&gt;

4.serviced service status

_Note: Resource pool id should be set to "default" unless you have created more than one. Deployment id should be a string value such as test or dev_

####UI workflow:

1.Add service template via CLI command: serviced template add &lt;apptemplate&gt;.json

2.Log into Control Center UI -> https://[HOSTIP]

3.Walkthrough deployment wizard

4.Click on "deploy and run" option, wait for application to come up. 

5.Check UI Logs, service health check status and individual service stdout log messages for any issues. 

6.Test the application in Control Center 

_Note: If you use the ELK stack template, Kibana has been configured to use TCP 8080. In addition, you will need to configure your local host file configuration if you want to use the reverse proxy feature in Control Center (VHost) for mult-instance deployment._ 

####Application topology view in Control Center
![image alt text](/images/applicationtopology.png)

####Host performance stats
![image alt text](/images/hosthealth.png)

####Deployed ELK stack in Control Center
![image alt text](/images/elkdeployed.png)

####Control Center Application Service Impact Model in Zenoss Resource Manager
![image alt text](/images/serviceimpact.png)

