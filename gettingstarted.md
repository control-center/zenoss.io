---
layout: page
title: "Getting Started"
description: ""
---
{% include JB/setup %}
Getting Started
Installation


 You’ll need an Ubuntu 14.04 host to perform the installation. The first step after OS installation is installing docker. The quickest way to install docker is the following:
wget -O - http://get.docker.io | sudo su
The install script will add the docker repository to apt and install docker. After it is installed, you must either run docker commands as root or add your user to the “docker” group. You can perform this task using:

sudo usermod -aG docker $USER

… with $USER being the username you want to add to the docker group. You’ll need to log out all sessions of the user before the change will take effect. 

Next, you will need to install serviced.

sudo sh -c 'echo "deb [ arch=amd64 ] http://apt.zendev.org/apt/ubuntu trusty universe multiverse" \
  > /etc/apt/sources.list.d/zenoss.list'
sudo apt-get update
sudo apt-get install serviced
Serviced should now be installed in /opt/serviced. There may be an initial pause before serviced starts up as it loads its internal services images into the running docker instance. Once that is complete you should see the Control Center serving up an AngularJS UI at https://localhost/. You will need to log in using a user that is in the wheel or sudoers group. Authentication is performed using PAM; you may need to configure PAM appropriately if you use Kerberos or LDAP for authentication.

If in doubt about whether Control Center has started properly, check the log ->/var/log/upstart/serviced.log and also check if the host is listening on TCP 443. 
Deploying Applications
Next, you will need to deploy an application. Applications are deployed from templates.

Below is the basic process on how you can take a Docker app and managed it via Control Center, the tutorial will use Logstash, Elasticsearch and Kibana as the target application stack:

Step 1 - Preparation

For this how-to we will be using a popular ELK (Elasticsearch, Logstash and Kibana) Docker image (qnib/elk).

If you are adventurous, you may want to try one of the Solr images available or any other application of your own choosing. 

Docker Hub hosts many prebuilt ready to run applications -> (https://hub.docker.com/) OR build your own image using Docker make, Packer or another Docker make tool. 

Download target image via Docker: docker pull qnib/elk 

Otherwise you may use Control Center to do all of the heavy lifting, including downloading the image. The caveat is that you will require a complete service template ahead of time. You can find one here, ready to run: https://gist.github.com/mmaloney/5b990225d614ca413d33

Step 2 - Test Target Application

Test Application in Docker first in interactive mode to make sure it runs as expected.

Sometimes you may need to connect to the running container to see what is really going on. A popular method is described below, nsinit is the preferred Docker method but requires additional dependencies. 

Nsenter example command: 
PID=$(docker inspect --format '{{.State.Pid}}' my_container_id)
nsenter --target $PID --mount --uts --ipc --net --pid
Also, you can run a shell command on start to access the container directly. This may be required for any customization of configuration files, though Control Center takes care of this for you by injecting any configuration at application runtime.

Example: docker run -i -t <image> bash

Step 3 - Creating your first template

It’s now time to build a Control Center template. Below is a simple walk-through of how to create an ELK (Elasticsearch, Logstash and Kibana) template in Control Center. 

First you will need to setup a suitable application template folder structure on your Control Center host. 

Example:
Root Application Template Folder
~:/LogAnalyzer  
Reference sample template: https://gist.github.com/mmaloney/c4e4c653300515e8b679

Elasticsearch Service Template Folder
~:/LogAnalyzer/elasticsearch
Reference sample template: https://gist.github.com/mmaloney/637a0b8f5252ea057c5b

Logstash Service Template Folder
~:/LogAnalyzer/logstash
https://gist.github.com/mmaloney/1392be0332e3e8089adf

KIbana Search Template Folder
~:/LogAnalyzer/kibana
https://gist.github.com/mmaloney/de150ef7130a8e05b85c

Create any required configuration files for the target applications. Control Center, injects configuration at runtime as opposed to mutating one or more configuration files within the container. This means that you will need to use the special configuration folder for each service and use a relative path. 

Simply copy the requisite config files to the appropriate service/config folder using the configuration file path as seen from within the container and you should be good to go.

Note: “-CONFIGS-” is a special folder name used only for housing application configuration files, based on the relative path of the configuration files. Another folder called “-FILTERS-” is used for application log filters used by the Control Center version of Logstash.

Example:
Elasticsearch Service Config Files
~:/LogAnalyzer/elasticsearch/-CONFIGS-/elasticsearch/config/elasticsearch.yml
https://gist.github.com/mmaloney/5e1da5daa58b70a3a671

~:/LogAnalyzer/elasticsearch/-CONFIGS-/elasticsearch/config/logging.yml
https://gist.github.com/mmaloney/0f44115aa742361d39de

Logstash Service Config Files
~:/LogAnalyzer/logstash/-CONFIGS-/etc/logstash/conf.d/syslog.conf
https://gist.github.com/mmaloney/90fd5047156108b944b9

Kibana Service Config Files
~:/LogAnalyzer/lkibana/-CONFIGS-/etc/nginx/nginx.conf
https://gist.github.com/mmaloney/3931911b551837132ff8

Note: You do not have to use the configuration file feature but it does make it much easier to quickly iterate on an application whether it be to modify port mappings or sharding the application service etc.

The next step will be to create several basic template files. Examples can be found here: 
Logstash example
https://gist.github.com/mmaloney/c88fcbb6fd03e3066684

Zenoss application example
https://gist.github.com/mmaloney/6e6df605d55fe0d40491

Memcached example
https://gist.github.com/mmaloney/0681871a89df24fd2615

Rabbitmq example
https://gist.github.com/mmaloney/d2083f87f8f534c3216e

Step 4 - How to author application templates
Create target folder structure on master Control Center host with the following schema:
~:/<Application Name>/
~:/<Application Name>/<Service Group Name>/
~:/<Application Name>/<Service Group Name>/<Service Name>/
~:/<Application Name>/<Service Group Name>/<Service Name>/-CONFIGS-/

Note: You will need to specify the PWD in order to create the ”-CONFIGS-” folder i.e. mkdir 
./-CONFIGS-
~:/<Application Name>/<Service Group Name>/<Service Name>/-CONFIGS-/<APPLICATION CONFIG PATHS>/

Next copy any configuration files required for the application into the expected application config path you created earlier. 

Create a service template called “service.json” in each service folder. Below is the basic anatomy of a service template, it is not meant to be exhaustive but enough to get you going.
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

Note: If you have more than one configuration file in your application, you must explicitly define it in the service.json at the service child level. If you do not, your application will not start correctly. 

Step 5 - How to compile and deploy an application template

One you are satisfied with your application templates and have tested the application manually, you are now ready to compile all of your service definitions into an application template.

Run the following command:
serviced template compile <template folder> > <filename>.json
i.e. serviced template compile ./LogAnalyzer > loganalyzer.json

If you don’t see any errors, then you are ready to move on to deploying the application.

Step 6 - Deploying an application with Control Center

Note: Make sure you have added a host to Control Center prior to starting the application

This can part can be done via the CLI or UI. For the CLI method, follow these steps:
1. serviced template add <template.json>
2. serviced deploy <templateId> <cchost:4979> <resource pool id> <deployment id>
3. serviced service start <serviceid>
4. serviced service status

The UI workflow is even simpler:
1. Add service template via CLI command: serviced template add <template.json>
2. Log into Control Center UI -> https://[HOSTIP]
3. Walkthrough deployment wizard
4. Click on “deploy and run” option, wait for application to come up. 
5. Check UI Logs, service health check status and individual service stdout log messages for any issues. 
