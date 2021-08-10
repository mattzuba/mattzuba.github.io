---
title: Running Jira Server on AWS - Part 1
---

Last year, my development team started using Jira to start tracking our project work.  This year, we expanded its use to other teams to utilize for unit-testing and end-to-end testing of a new ERP implementation.  Next year, we're looking at potentially using the Service Desk feature.  Knowing last year that as we used Jira that it would be used more and more, I wanted to ensure it was setup right from the beginning with all three of the Jira products installed.

As a non-profit organization, we took advantage of [Atlassian's Community Licensing]{:n} program to run Jira Server.  This post will be the first in a series of how I've setup Jira to run on AWS.

# In This Post
{: .no_toc}

- TOC
{:toc}

# AWS VPC Configuration

In this first post, I'll highlight the configuration of the AWS VPC, the AWS resources used, and the installation of Jira.  If you're not familiar with creating VPCs (including the networking that goes with them), Application Load Balancers, configuring Amazon Linux EC2 instances, AWS RDS and AWS EFS, this will be challenging.  I strongly recommend not proceeding until you have a firm grasp on these concepts.

Below is a basic diagram of how the VPC I used for this setup as well as discuss the security groups created and their inbound rules.  _Note: as a non-profit organization, we leverage [AWS credits available through TechSoup]{:n} which allows us to run this infrastructure for very minimal cost.  Adjust what you implement based on your needs and available funds._

The setup of this VPC is your typical public/private subnets with a bastion host and NAT Gateway.  There are additional private subnets for other functions to keep things separated out.  If you're creating a VPC only for Jira, this may not be necessary.  We host other items in this VPC as well, so it's configured as such.  You can see [Amazon's VPC Tutorial]{:n} for the basics on setting this up if necessary.

![AWS Diagram](/images/jira-diagram.svg)

## AWS Resources

Let's start with the resources used for this setup.

* Public (DMZ) and private subnets in both AZs.
* A NAT Gateway in the DMZ.
* Application Load Balancer (ALB) - There is also a Target Group for this ALB that includes the Jira Server instance with a health check pointing to `/status` on the instance.
* A t3.nano Bastion host in the DMZ for accessing private subnet servers.
* An m5.xlarge (your needs may vary) EC2 instance for Jira.
  * Amazon Linux 2 will work if you can use it.
* A t2.medium MySQL 5.7 RDS instance (your needs may vary).
  * The database we're using also hosts Confluence, so I used Atlassian's [Confluence MySQL setup guide]{:n}.  If you'll only be hosting Jira, you can follow the [Jira MySQL setup guide]{:n}.
* An EFS file system for attachments and avatars.
  * Add a lifecycle to this; we use 14 days however that could change as time goes on.
  * I recommend using encryption at rest and in transit if possible.

## Security Groups

In order to secure communication between everything, a number of security groups are used:

* **ssh-bastion** - Attached to Bastion EC2 instance, allows port 22 inbound.
* **jira-load-balancer** - Attached to ALB, allows ports 80 and 443 inbound.
* **jira-server** - Attached to Jira EC2 instance, allows port 8080 from **jira-load-balancer**, port 22 from **ssh-bastion** and itself.
* **jira-database** - Attached to RDS instance, allows port 3306 from **jira-server** and **ssh-bastion** (so that I can access the database over the Bastion host).
* **jira-efs** - Attached to Jira EFS, allows port 2049 from **jira-server**.

As you can see, the only way into the ecosystem is through the bastion host and the load balancer which reside in the DMZ subnets, the only public subnets in the VPC.  Everything else has its access based on inbound rules from other Security Groups rather than IP addresses to ensure that only the resources that should access other resources can.

You may also notice that there while most of this setup is redundant, the lone Jira server is still a failure point.  Unless you're using Jira Data Center, [Jira does not support a load-balanced installation]{:n}.  It is not yet shown here as I haven't completed the setup, but I am in the process of setting up a cold spare in the other AZ that can be manually spun up in case something happens in the primary AZ.  The cold spare can be added to the Load Balancer's Target Group and started up when needed so long as it is kept up to date with the primary server.

## Some Other AWS Notes

Here are some things you may want think about doing for good measure.  Some of these drive up the costs of running this infrastructure, so do what you can.

* Add tags to all the resources you're creating so that you can easily track costs in billing if needed.
* Take regular snapshots of the Jira EC2 instance.
* Take regular snapshots of the RDS instance.
* Use AWS Backup to take regular backups of the EFS.
* Ideally you'll have a NAT Gateway and Bastion host in each public DMZ for full HA.

# Jira Installation

Once your VPC is configured and all of your resources created, you're ready to install Jira, configure the EFS, configure Okta (or your SSO solution of choice, if you're using one) and start playing with Jira.

## Prepare the EC2 instance

There are a few packages you're going to want to install to ensure things go easily (this assumes Amazon Linux 2, adjust as necessary for other OS):

* `yum-cron` can help ensure the server stays up to date with security updates.
* `amazon-efs-utils` will help with mounting the EFS volume.
* `diffutils`, `patch` and `patchutils` come in handy during upgrades to compare and update core files.
* `postfix` is installed by default on Amazon Linux 2; if you're using SES to send emails, follow Amazon's guide on [Integrating Amazon SES with Postfix]{:n}.  Since I have yum-cron enabled, I have this configured so that it can also send emails when necessary.  I've then set the SMTP mail server in Jira to `localhost` to send mail through the local Postfix daemon.

## Installation

Rather than reinvent the wheel, I'll simply point you to the basic Atlassian instructions for [installing Jira]{:n}.  If you're installing more than one of these, I recommend installing Core and then enabling the other product you'll want/need.  Not sure what you need?  Check out [Atlassian's docs] {:n} to help you decide.  Make sure to run this installation as `root` using `sudo`, and allow the installer to create a systemd service file so that Jira runs as a service on the server.  Once you go through the setup and have Jira running successfully, you'll want to stop it from running so that we can continue the configuration on the file system.

~~~ bash
$ systemctl stop jira
~~~

Now that Jira is installed, we'll do the additional setup on the command line to use our EFS, configure Okta, setup SSL to the RDS instance and some other minor tweaks.

### Configure EFS

First you'll need to switch to the `root` user so that we can access the Jira application data.  You'll then need to navigate to the Jira application-data folder and start making some adjustments.  First we're going to move our data folder (which holds attachments, assets and avatars) to a temporary _.bak_ folder.

~~~ bash
$ cd /var/atlassian/application-data/jira
$ mv data data.bak
$ mkdir data
~~~

Now we're going to mount our EFS.  Visit your EFS Console and note the file system id of the file system we'll be mounting here.  Remember to make sure that your Jira server has access to the EFS by security groups as described above in the AWS setup.  Modify your `/etc/fstab` file to add the EFS entry to the end of the file.  The entry below mounts the root of the file system to our newly created data directory over TLS.  Replace the file system id at the beginning of the line with your own.

~~~
fs-fb25f5cf:/	/var/atlassian/application-data/jira/data efs _netdev,tls 0 0
~~~

Now run `mount -a` to mount the file system to the folder.  You can verify the mount was successful by running the `mount` command and looking for the mount.

~~~ bash
$ mount | grep jira
127.0.0.1:/ on /var/atlassian/application-data/jira/data type nfs4 (rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,noresvport,proto=tcp,port=20283,timeo=600,retrans=2,sec=sys,clientaddr=127.0.0.1,local_lock=none,addr=127.0.0.1,_netdev)
~~~

Now with our new EFS mounted, we'll need to make some adjustments to the folder permissions and put the proper content into it.

~~~ bash
$ chown jira.jira data
$ chmod 700 data
$ mv data.bak/* data/.
$ rmdir data.bak
~~~

### Configure Okta

There are [a number]{:n} of SAML apps for Jira on the Atlassian Marketplace, but as we are using Okta, I decided to go with Okta's official SAML integration for Jira.  It uses the Jira API to provision users when they are assigned the app, and with some small tweaks it can support SSO on the Service Desk portal (if you're using Jira Service Desk).

Like the above instructions for installing Jira, I'll skip re-inventing the wheel and simply point you to [Okta's documentation]{:n}.  Once that is complete, we'll add some additional configuration if you're using Jira Service Desk.  This is a recreation of [my comment]{:n} on a Jira Bug Report.  Basically, Jira's Seraph authentication security framework (which Okta leverages) doesn't yet support SSO for Jira Service Desk's Customer Portal, only the agent login.  This is a small hack around it by enabling Tomcat's version of Apache Httpd's mod_rewrite and creating some rewrite rules for the Service Desk login page to use Okta.

1. Edit `/opt/atlassian/jira/conf/context.xml` and add the following line _above_ the line containing \</Context\>:
    ~~~ xml
    <Valve className="org.apache.catalina.valves.rewrite.RewriteValve" />
    ~~~    
2. Create `/opt/atlassian/jira/atlassian-jira/WEB-INF/rewrite.config` and add the following to it:
    ~~~
    RewriteCond %{REQUEST_URI} ^/servicedesk/customer(/.*)?/user/login [NC] 
    RewriteCond %{QUERY_STRING} ^destination=(.*)$ 
    RewriteRule .* /okta_login.jsp?RelayState=\%2Fservicedesk\%2Fcustomer\%2F%1 [NE,R=307,L] 
    
    RewriteCond %{REQUEST_URI} ^/servicedesk/customer(/.*)?/user/login [NC] 
    RewriteRule .* /okta_login.jsp?RelayState=\%2Fservicedesk\%2Fcustomer\%2Fportals [NE,R=307,L]
    ~~~

The first rewrite cond/rule pair captures requests that have a destination parameter and the second captures requests without.  Both redirect to the okta_login.jsp page with the RelayState set so that when okta_login.jsp does the POST over to Okta, it will carry the relay state and return the user to the proper page after the login.

### SSL to MySQL RDS

> Secure all the things!

In all honesty, anything in transport these days should be secured with SSL or TLS (and anything at rest should be encrypted when possible).  Unlike 20 years ago, encryption is computationally inexpensive, and it's just good practice.  That said, since the communication within this VPC is all private between the Jira instance and the database, secure communication isn't a necessity unless you have other things in this VPC and it's not the most secure.  I've setup SSL here as more of a learning experience, so you can do with it what you will.

{:n: target="_blank"}

[Atlassian's Community Licensing]: https://www.atlassian.com/software/views/community-license-request
[AWS credits available through TechSoup]: https://www.techsoup.org/Products/--G-50197--AWS
[Amazon's VPC Tutorial]: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenario2.html
[Confluence MySQL setup guide]: https://confluence.atlassian.com/display/DOC/Database+Setup+For+MySQL
[Jira MySQL setup guide]: https://confluence.atlassian.com/display/ADMINJIRASERVER/Connecting+Jira+applications+to+MySQL+5.7
[Jira does not support a load-balanced installation]: https://community.atlassian.com/t5/Jira-questions/JIRA-load-balancing-without-datacenter/qaq-p/442889
[Integrating Amazon SES with Postfix]: https://docs.aws.amazon.com/ses/latest/DeveloperGuide/postfix.html
[Installing Jira]: https://confluence.atlassian.com/display/ADMINJIRASERVER/Installing+Jira+applications+on+Linux
[Atlassian's docs]: https://confluence.atlassian.com/confeval/jira-software-evaluator-resources/jira-software-which-jira-do-i-need
[a number]: https://marketplace.atlassian.com/addons/app/jira/top-rated?query=saml
[Okta's documentation]: https://saml-doc.okta.com/Provisioning_Docs/Okta_Jira_Authenticator_Configuration_Guide.html
[my comment]: https://jira.atlassian.com/browse/JSDSERVER-630?focusedCommentId=2389890&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-2389890