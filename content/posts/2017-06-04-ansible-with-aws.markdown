---
layout: post
slug: ansible-with-aws
title: "Ansible dynamic inventory with AWS EC2"
date: 2017-06-03 21:47:50 +0800
comments: true
tags: [aws, ansible, devops]
---
By using [ansible dynamic inventory](http://docs.ansible.com/ansible/intro_dynamic_inventory.html#example-aws-ec2-external-inventory-script), we can run ansible based on instances' tags, such as `app=backend,env=staging` etc.
Here will talk about how to make use of defined tags on AWS EC2 and run ansible-playbook scripts onto them.

#### 1. Define a few tags on target EC2 instances

Define some tags on created EC2, such as `App=backend`, `Environment=staging`, `Usage=clock-worker`

![Example of creating tags on EC2](/img/ec2-tags-demo.png)

To assign tags on the multiple instances, you can use with [AWS Resource Tag editor](https://resources.console.aws.amazon.com/r/tags).

#### 2. Prepare ansible ec2.py and ec2.ini files

Download [ec2.py](https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.py) and [ec2.ini](https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini), put the files on `./inventory` from your playbook directory.
If you already have the files, remember to set `group_by_tag_keys = True` in your ec2.ini.

#### 3. Run in ansible / ansible-playbook
{{< highlight shell >}}
ansible -i ./inventory/ec2.py --limit "tag_App_backend:&tag_Environment_staging:&tag_Usage_clock_worker" -m ping all
ansible-playbook -i ./inventory/ec2.py --limit "tag_App_backend:&tag_Environment_staging:&tag_Usage_clock_worker" deploy.yml
{{< / highlight >}}


#### Reference
[Ansible - Dynamic Inventory](http://docs.ansible.com/ansible/intro_dynamic_inventory.html#example-aws-ec2-external-inventory-script)
