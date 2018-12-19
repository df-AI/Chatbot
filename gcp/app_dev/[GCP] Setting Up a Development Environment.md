
# [GCP] Setting Up a Development Environment

@(GCP Coursera)[node.js, GCP 프로젝트 생성]

In this lab, you will provision a Google Compute Engine virtual machine and install software libraries for Node.js software development on Google Cloud Platform.

----------

[TOC]

## Task 1: Creating a Compute Engine Virtual Machine Instance

> In this section, you will use the Google Cloud Platform Console to provision a new Google Compute Engine virtual machine instance. 

1. In the Cloud Platform Console, on the Navigation menu, click Compute Engine.
2. On the VM Instances page, click Create.
3. On the Create an instance page, for Name type dev-instance, select us-central1 for region and us-central1-a for the zone.
> GCP Regions and Zones
> Google Cloud Platform offers products and services in multiple distinct geographic locations, called regions.
> Each region has multiple distinct zones. Each zone is isolated from other zones in terms of power and internet connectivity.
4. In the Identity and API access > Access Scopes section, select Allow full access to all Cloud APIs. 
5. In the Firewall section, enable Allow HTTP traffic.
6. Leave the remaining settings as their defaults, and click Create.
> It takes about 20 seconds for the virtual machine to be provisioned and started.
7. On the VM instances page, in the row for the dev-instance, click SSH (in the Connect column).
> This launches a browser-hosted SSH session. If you have a popup blocker, you may need to click twice.
> There's no need to configure or manage SSH keys.

## Task 2: Install software on the VM instance
1. In the SSH session, to update the Debian package list, execute the following command:
``` powershell
sudo apt-get update
```
2. To install Git, execute the following command:
``` powershell
sudo apt-get install git
```
If prompted, press Enter.
3. To download the Node.js setup script, execute the following command:
``` powershell
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
```
4. To install npm and Node.js, execute the following command:
``` powershell
sudo apt install nodejs
```

## Task 3: Configure the VM to Run Application Software
In this section, you will verify the software installation and run some sample codes.
1. To check the version of Node.js, execute the following command:
``` powershell
node -v
```
2. To clone the class repository, execute the following command:
``` powershell
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```
3. To change the working directory, execute the following command:
``` powershell
cd ~/training-data-analyst/courses/developingapps/nodejs/devenv/
```
4. To run a simple web server, execute the following command:
``` powershell
sudo node server/app.js
```
5. Return to the Cloud Console VM instances list, and click on the External IP address for the dev-instance. 
> You should see a Hello GCP dev! message from Node.js.
6. Return to the SSH window, and stop the application by pressing Ctrl+C.
7. To install the Node.js library for Compute Engine, execute the following command:
``` powershell
npm install
```
8. To run a simple Node.js application that lists Compute Engine instances, execute the following command:
``` powershell
node list-gce-instances.js
```
