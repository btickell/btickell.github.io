---
layout: post
title: AWS Tutorial
date: 2020-02-23 20:13 -0500
---

# Setting up AWS
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

# Running with Doodad

Doodad is a python package that allows you to quickly launch requests for spot instances with a specified price which is often much cheaper than on demand instances created through the console. With doodad, you can spawn many seeds of a given experiment and log results to reliable AWS storage or an EC2 experiment server.


## Install doodad

```bash
git pull https://github.com/justinjfu/doodad.git
cd doodad/
pip install -e .
```

Once installed run the EC2 setup script in `scripts/`

```bash
python scripts/ec2_setup.py 
```

## Launching a script
Lets say we have an example script `doodad_test.py` which takes a single argument and prints it to the command line:
``` python
import argparse
parser = argparse.ArgumentParser()
parser.add_argument('--input', type=str)
args = parser.parse_args()
print('Hello, my input argument is {}'.format(args.input))
```

To run this script with doodad, make sure you have created a bucket on s3 (through the EC2 console) and run the following code:

``` python
import doodad as dd
mode = dd.mode.EC2AutoConfig(
			s3_bucket=S3_BUCKET_NAME,
			s3_log_prefix='storage',
			s3_log_path='',
			ami_image=AMI_NAME,
			aws_key_name=KEY_NAME,
			instance_type="t2.nano",
			region='us-central1-a',
			terminate_on_end=True,
			spot_price=PRICE_PER_HOUR # Check this on the ec2 spot request calculator (set slightly above min price)
)
arg_string '--input doodad'
run_python('doodad_test.py', cli_args=arg_string, mode=mode_obj, docker_image=DOCKER_IMAGE)
```
Where `PROJECT_NAME`, `S3_BUCKET_NAME`, `AMI_NAME`, and `DOCKER_IMAGE` are specific to the user.

This code does a few important things:

1) Creates a doodad mode, specifying all the parameters of the instance(s) being created.

2) Specifies that the instance should terminate when the target script is done executing.

3) Designates that all log files saved by doodad should be saved to s3 storage, located at path `s3://BUCKET_NAME/storage/doodad_test`

5) Constructs input argument string to take `doodad` as input.

6) Launches `doodad_test.py` to run on the described instance, using docker image `DOCKER_IMAGE`
	

For a comprehensive list of instance types and their current spot pricing see [https://aws.amazon.com/ec2/spot/pricing/](https://aws.amazon.com/ec2/spot/pricing/)

## Mounting Files (AWS)
For the above example, our script was simple and didn't require any additional code or data to run. Now lets say we have two directories, `utils/` and `data/`

Our script `compute_metrics.py` computes the mean and standard deviation of the data:
``` python
from utils import get_metrics
import numpy as np

x = np.load('data/example_dat.npy')
mean, sd = get_metrics(x)
print("My mean is {} and my sd is {}!".format(mean, sd)
``` 

In order for this code to run correctly, we need to *mount* our code from doodad when launching our instances. To do this we modify the original code block:

``` python
import doodad as dd
OUTPUT_DIR = '/outputs'
PROJECT_ROOT = 'path/to/your/project/'
code_mount = mount.MountLocal(mount_point='utils', local_dir=PROJECT_ROOT+'utils/', pythonpath=True)
output_mount = mount.MountS3(s3_path='results/', mount_point=OUTPUT_DIR+'results/' , output=True)
mounts = [code_mount, output_mount]
```

Then when we run our code as before, we pass the mounts list to 
```python
run_python('doodad_test.py', cli_args=arg_string, mode=mode_obj, mounts=mounts docker_image=DOCKER_IMAGE)
```
This code is very similar to before but has two mounts with important arguments.
1) The code mount specifies `pythonpath=True`, which will automatically append the location of the mounted directory to your python path on your spawned instances.
2) The output mount is a designated location where your script can store things that will be synced to storage by doodad. The argument `output=True` specifies that the mount is an empty directory to be created by doodad, and that any files you write into the `mount_point` directory, will be saved to storage.

And we are done. You should now be able to launch arbitrary code to a wide array of dynamic compute instances depending on what you require.

