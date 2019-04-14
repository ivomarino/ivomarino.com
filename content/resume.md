+++
comments = false 
date = "2019-04-13T00:00:00-00:00"
draft = false
share = false
image = "img/head-03.jpg"
slug = "resume"
title = "Resume"
+++

Passionate about all things related to Unix like systems, I&#x27;ve started using Linux since 1998 starting with Slackware 3.x then moving along with Debian and derivates mostly.

I help customers run large-scale, reliable applications in the Cloud by working with engineers and architects to design, build, optimize, and operate infrastructure. My specialties are systems automation, security, and migrating workloads to container-based deployments. Interested in Continuous Integration, Containers, Docker, Kubernetes and similar technologies. 12Factor Evangelist, proponent of Infrastructure as Code, Continuous Delivery, and Self-Documenting systems

I speak native German, Italian and fluent English. Actually I'm only interested in remote working jobs.

---

## Consulting

### TechTalk managed OpenShift, [Swisscom AG](https://swisscom.ch)

In order to launch their new OpenShift based PaaS platform Swisscom introduced the concept [#makeswisscomnerdyagain](https://www.linkedin.com/feed/hashtag/?keywords=%23makeswisscomnerdyagain) to attract a new generation of developers, I had the pleasure to support them in a [series of tech talks](#swisscom-techtalk-managed-openshift) which lead to great feedback from the socials.

### Building a managed WordPress hosting platform, [switchplus ag](https://switchplus.ch)

I supported the developers during the initial lift of their Managed Wordpress Hosting platform called [PRESENCE](https://switchplus.ch/en/products/presencebasic/). The specifications required to build a high available WordPress hosting infrastructure in the Azure Cloud. [Capistrano](https://capistranorb.com) has beed introduced as remote server automation and deployment tool, including rollbacks.

### Docker based Node.js deployments via Wercker, [ginetta AG](https://ginetta.net)

As web architects ginetta develops applications based on [Node.js](https://nodejs.org) using a self-hosted headless and api-driven CMS called [Cockpit](https://getcockpit.com). The goal was to come up with a Docker based solution for easy local development and `git push` based deployment scenarios. To achieve this a [StrongLoop](https://github.com/strongloop/strongloop) Docker image has been [built on Docker Hub](https://hub.docker.com/r/ttssdev/strongloop) which integrates easily with [Wercker](https://app.wercker.com) and allows easy branch based deployments to the cloud. For service discovery and registration on HAProxy another Docker container has been built called [strong-confd-parser](https://hub.docker.com/r/ttssdev/strong-confd-parser).

### Automation &amp; deployments, [mms solutions ag](https://mmssolutions.io)

[ns.publish](https://nspublish.io) is a tried-and-tested publishing system for financial and corporate publications which required a high-available hosting infrastructure on Azure cloud, I helped to build up the whole architecture including the deployment pipeline. The major need was to provision easily new web projects across all environments, from dev to production. I introduced Ansible to fully automate the whole provisioning process so developers were able to setup new customer projects for the whole team in a centralized way.

### CoreOS in production, [Moneyhouse AG](https://moneyhouse.ch)

Moneyhouse links different data, coming from the Swiss business world, intelligently and makes it useful for everyone. Over 13 million facts about everything that is important for business updated on daily basis. In order to handle this huge amount of data Moneyhouse was one of the first to adopt a Docker based solution not only for development but also in production. I worked side by side with the developers to build up the new infrastructure based on several [CoreOS](https://coreos.com) clusters. This project was a big success not only from a technical point of view but also regarding the introduction of the DevOps philosophy to the whole team.

### HA WordPress hosting on Azure, [Pro Helvetia](https://prohelvetia.ch)

When the Swiss Arts Council needed to relaunch their web presence I joined the development team as consultant to architect the new infrastructure. The goal was to deploy several HA nodes for hosting a WordPress ecosystem which delivers content across different nations. Automation was the key to success here, so we provisioned several nodes using Ansible providing everything from load balancers to [Let's Encrypt](https://letsencrypt.org) SSL certificates. I've also introduced a unified deployment pipeline which allows zero-downtime deployments on different projects.

### HA WordPress hosting, [Swisscom Magazine](https://mag.swisscom.ch)

The Swisscom Magazine is a social oriented blog about Digital Transformation, one key aspect in the project was to deliver fast performance throughput and stability in case of high user load. The key to succeed in this scenario was to put [Varnish](https://varnish-cache.org) into the content delivery chain, this way pages which have already been processed by the backend where directly served from cache. The system response was extremely stable also during media campaigns where video down and uploads have been involved.

### Hardened HA WordPress, [blog.migrosbank.ch](https://blog.migrosbank.ch)

Security was an important aspect from the beginning when MigrosBank decided to launch their new blog platform. I've planed the provisioning of the system using separate nodes and load balancing via HAProxy for gaining best stability and performance. The frontend load balancers where hardened and tuned to get the best SSL ciphers for an A+ grade. The whole infrastructure is completely redundant and allows failover in case a single nodes goes down or needs maintenance.

### Twelve-Factor WordPress App, [required gmbh](https://required.com)

I collaborate quite often with the guys from Required, a remote agency from Switzerland & Germany. When we first meet they were looking for a clean layout for WordPress projects so I showed them [Bedrock WordPress boilerplate](https://roots.io/bedrock/) which became the de-fact standard for all future projects. This way it was super easy to introduce concepts like the [Twelve-Factor WordPress App](https://roots.io/twelve-factor-wordpress/) and define a complete deployment pipeline based on Capistrano.

---

## Experiences

### Senior DevOps Engineer, [1plusX AG](https://1plusx.com)
##### 2019 - present

1plusX uses state of the art and innovative Machine Learning approaches to  generate competitive and leading edge technology, builds and operates systems that use different types of data from the web, mobile or TV networks to calculate valuable and meaningful information for a broad spectrum of customers.

All production systems were deployed to EC2 and used S3, Couchbase, ElasticSearch and Kafka for data processing. New features were adopted as they were released by Amazon, including ELB and EBS. Auto Scaling was used easily since it was released, and A/B tests have been implemented long before they were en vogue. One of my roles is to setup and build processes that are easy to use for development and production, optimize Jenkins CI/CD, facilitate testing of code and improve deployment processes based on Docker, ECS, and Kubernetes.

I work side by side with more then 20 engineers expanding usage of Terraform and Ansible, containerizing workloads, and iterating on DevOps best practices to improve margins.

### DevOps, [watson](https://watson.ch)
##### 2017 - 2018

The goal to achieve at watson was to build up and launch their new German website [watson.de](https://watson.de) in a quite short timeframe. The platform should be analog to the original Swiss website, speaking of a technological point of view, but abel to stand a higher user load.

In order to get this up and running we have chosen to deploy the new infrastructure in a complete automated way using Ansible -- "Infrastructure as Code" has been one of the main topics in the list of priorities. Involved technologies were HAProxy, Varnish, nginx, memcached, HHVM, Thumbor, Serf, Sphinx and MySQL.

Another important aspect was to introduce main concepts of the DevOps philosophy, especially when it comes to deployments and A/B testing. Here the aspect of automated deployments, Slack notification integration for notifying the whole dev team and rollback features have been added to the stack.

### Senior Linux Engineer, [ricardo.ch AG](https://ricardo.ch)
##### 2013 - 2016

At Ricardo, a swiss based online auction platform, I directly joined the SRE team as Linux Engineer. The team was responsible for keeping the Linux based production environment up and running.

The infrastructure was mostly CentOS based and Puppet was used as provisioning tool, after some time I led the effort to embrace configuration management, choosing Ansible as the tool to deploy configurations in the infrastructure.

This allowed to provision and manage in an automated way technologies like on-premise Elastic Search clusters (ELK) for aggregating customer related data which was important for data science and big data analysis.

I&#x27;ve actively contributed to build up Ricardo Shops deploying and maintaining infrastructures based on Hybris and backend technologies like Varnish, HAProxy, MySQL and PostgreSQL.

### IT Manager, [Goethe-Institut Rom](https://goethe.de)
##### 2009 - 2012

After completing my Computer Science studies I directly joined the German Goethe-Institut in Rome as IT Manager responsible for all institutes in Italy.

My experience there based the layout for initial automated task executions both locally as on-site, I had change to to learn how government institutions organize their IT when it comes to deploy the infrastructure worldwide.

---

## Open Source projects

#### [AppFlow](https://github.com/ttssdev/appflow)

This tool allows to provision consistent environments over the entire infrastructure. Automated processes with traceability and reduction of error rates with possibility of reproduction and rollback functions. It's Open Source and independent from the physical environment (On-Premise, Cloud, Bare Metal). AppFlow works in both public and private clouds, as well as bare metal. Multitenant ready and config files are encrypted and maintained inside a private Git-Repo.

### [mentors.debian.net](https://mentors.debian.net)

This platform allows new Debian package maintainers to get in contact with official Debian developers (sponsors) for their packages. The system allows upload of new created packages to a special repository where they can be reviewed from DDs. The service gained big success in the Debian GNU/Linux community and has been also mentioned on Linux Magazine. More infos are on [https://wiki.debian.org/DebianMentorsNet](https://wiki.debian.org/DebianMentorsNet).

### [Debian](https://debian.org)

I have been a Debian Package Maintainer for several years and managed packages like `libaudio-flac-header-perl`, `libaudio-flac-perl`, `libcvs-perl` and `mp3roaster`.

---

## Speaks and Talks

### Swisscom TechTalk managed OpenShift
[Swisscom Business Campus](https://www.linkedin.com/feed/update/urn:li:activity:6521424888991928320), 2019  
[Impact Hub Bern](https://www.linkedin.com/feed/update/urn:li:activity:6518499967622004736), 2019  
[Swisscom Business Campus](https://www.linkedin.com/feed/update/urn:li:ugcPost:6521297710937624576), 2019 

### [Provisioning via Docker and AppFlow](https://www.meetup.com/Ansible-Zurich/events/238716282/)
5th Ansible Zurich Meetup, 2017

### [Vagrant/Ansible workflow and Percona XtraDB Cluster (Galera) with HAProxy](https://www.meetup.com/WordPress-Zurich/events/219612359/)
WordPress Zurich Meetup #8, 2015

---

## Education

### Universit&agrave; di Roma “La Sapienza”
### Computer Science Department, Information Technology, B.Sc. IT
##### 2003 - 2009

---

## Skills

* Configuration Management: Lots of Ansible
* Kubernetes, OpenShift, Rancher, CoreOS
* Infrastructure Automation (AWS CloudFormation, Google Cloud Platform Deployment Manager)
* Hashicorp Tools as Terraform, Consul, Serf, Packer, Vagrant
* Cloud Platforms: Amazon AWS, Azure
* Amazon Web Services (VPC, IAM, EC2, S3, Elastic Beanstalk)
* Containers: Docker, LXC, BSD jails
* CI/CD: Jenkins, Wercker, Drone
* Virtualization: VMware vSphere, Proxmox, KVM, OpenStack / bhyve
* Monitoring: Icinga2, Nagios, Zabbix
* Logging: Elasticsearch, Logstash, Kibana, Graphite, Grafana, collectd, InfluxDB, Telegraf
* MySQL Galera cluster, PostgreSQL
* Git, Gitea, GitLab
* GlusterFS, Ceph, Minion, ZFS
* HAProxy, Apache2, nginx, Varnish, Let's Encrpyt deployments
* GNU/Linux (Debian, Ubuntu, SuSE, Fedora, CentOS)
* FreeBSD (NetBSD, OpenBSD)
* Perl, OO-Perl, ANSI C, Python, PHP7
* Shell (sh/bash, dash, csh, zsh)
* Deployments: Capistrano, StrongLoop nodejs
* Excellent troubleshooting
