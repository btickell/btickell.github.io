---
layout: post
title: AWS Tutorial
date: 2020-02-23 20:13 -0500
---

## Introduction
AWS provides users with a wide range of tools for launching and monitoring distributed compute jobs with varying stability and pricing needs. This makes it a great option for researchers who need to rapidly test and iterate on experimental ideas and who need to test the effects of hyperparameters at scale. This tutorial will show you how to create a simple AWS image and how to boot an instance from this image in order to run your code.

## Log in to your AWS account
Go to [AWS console login page](https://aws.amazon.com/console/) and sign in with your AWS account.

## Launch instance (with images for example)
1) Click the "Launch Instance" button.

<img src="/figures/aws_tutorial/launch_instance.png" width="1000" height="600" />

2) Select ubuntu 18.04 as your boot image.

<img src="/figures/aws_tutorial/ami_selection.png" width="800" height="450" />

3) Select a free tier eligable instance (we are only using this instance to install requirements so no need for large compute.)

<img src="/figures/aws_tutorial/instance_type_selection.png" width="1000" height="600" />


4) Download .pem file

Click the "Download Key Pair" button shown below.

<img src="/figures/aws_tutorial/keypair_creation.png" width="600" height="450" />

This will download your .pem file which will serve to identify you to amazon when using the command line API or connecting to your instances remotely. Copy this file to a known location.

This key will be used to identify you over ssh or when using AWS with command line.

```bash
mkdir ~/.aws_keys
mv /path/to/downloaded/.pem ~/.aws_keys/nameofkey.pem
```

## Connect to your instance

Once your instance is running (denoted by the green status symbol in the instance state column), you can connect to your machine and make changes. 

In the AWS console, you will find a public IP address for your instance (as pictured below):

<img src="/figures/aws_tutorial/ip_info.png" width="1100" height="700" />

```bash
ssh -i /path/to/your/.pem ubuntu@yourpublicip
```

Once connected you can install any software, packages, or docker images as well as store any files you would like present in all of your experiments. For persistent file storage, it is preferable to store files on a file server or on AWS s3 storage for ease of access.

It is beneficial to do this now, as you will create a snapshot of this instance that you can boot from in the future.

## Make a snapshot. 

We now have a single instance with our desired software and configuration. In order to launch instances using this configuration, we need to create a **snapshot** which will create an image (**AMI**). This creates an AMI based on a target instance that can be booted from many times.

To do this, right click on your instance in the AWS console and select create snapshot:

<img src="/figures/aws_tutorial/snapshot_img.png" width="1100" height="700" />
<img src="/figures/aws_tutorial/snapshot_creation.png" width="1100" height="700" />


You are now able to launch as many instances from your created snapshot as you want to run experiments as needed.

## Important Info:
If you lose your .pem file for a keypair, you will need to recreate any instances using that security key with a new keypair.

Spot instances vs On demand instances

price vs reliability 

Region: 

affects price and pool of machines available

How can you launch many jobs at once? (Doodad :D)

## Running with Doodad

Assume you have:
1) A target AMI
2) A docker image to pull / already on the AMI
3) Code to run

1) Install doodad

```bash
git pull https://github.com/justinjfu/doodad.git
cd doodad/
pip install -e .
```

## INSERT WIKI INFO HERE