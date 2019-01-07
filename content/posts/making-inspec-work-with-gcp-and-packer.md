---
title: "Making Inspec with SSH work with Packer on the Google Cloud Platform"
date: 2019-01-07
---

# Background

This blog post aims to help anyone trying to make a [Packer]() image, testing with [Inspec](), on the [Google Cloud Platform](). I have recently been using the these tools for my day-to-day and had come across an interesting problem. Hopefully this helps someone out there to make this work.

When I first set this up, I had to figure out a way to get Inspec to talk to the newly created VM through the provisioning steps of Packer. I also didn't want to install anything extra on the new VM because then I would have to clean that up. This proved to be more difficult then I had realized but in the end I have at least the basic pipeline working. 

Really quick, if you haven't used any of the tools let me explain what I'm trying todo. I want to create an image of a VM that I can just spin up with everything I need. 

So enough yap, let's see what the code looks like. 

# The problems I faced

So let me walk through the problems I encountered along this journey that I will outline and show answers to.

Problem #1: The first problem I hit was that I needed Inspec to talk to the new VM, but packer doesn't provide those variables very easily. The first thing I had todo was get the IP address somehow to the Inspec command to tell where to go looking for the VM.

Problem #2: Once I had the VM IP, I had to connect the Inspec process somehow to that VM. The main method of choice is SSH. This meant I needed to establish a key to login and tell the GCP VM of that key. Again, one would think this would happen out of the box, but that wasn't the case. I believe this is a bug in Packer but we can go through that later. 

# Solving Problem 1: Getting the IP of the Packer build in GCP

This sounds easy enough but is surprising difficult: how do I get the IP of the currently running VM where all I can do is execute CLI commands? One approach might ping some remote web site that tell you your public IP, but that seems ridiculous to introduce another layer outside of your control. My search stumbled upon a number of examples of how to do this on AWS but very little on GCP. However, that did give me a framework of what todo.

What people showed what to use was called the Metadata server. The Metadata server is a service for every VM running on [AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) (and [GCP](https://cloud.google.com/compute/docs/storing-retrieving-metadata)) that can tell you about the instance you are running on. This was the thing I needed, I found the [documentation for GCP](https://cloud.google.com/compute/docs/storing-retrieving-metadata) and sure enough, it had information on all of the exposed IP addresses. The particular address I needed was http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip.

Now that I have the IP address, how would I exactly retrieve that information from the command line. Thankfully, curl was already installed so I crafted this command:

```
curl  -H \"Metadata-Flavor: Google\" metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip
```

And executed that and viola, the IP address I needed. 

Now, something to be pointed out. That HTTP header we added (`Metadata-Flavor: Google`) is required. This is because the service only responds to commands with this header because otherwise these might be erroneous requests that are pointed to this service by accident. 

Now that I have my IP address, how do I use it? Again, long searches didn't show a way to pipeline variables from one Packer step to another so what was I going todo? I'm not 100% sure how I stumbled upon but I found a very neat trick of downloading the file after you created it. Here is what that will look like in Packer's provisioning steps:

```json
"provisioners": [
    {
        "type": "shell",
        "inline": ["curl -H \"Metadata-Flavor: Google\" metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip > /tmp/ip"]
    },
    {
        "type": "file",
        "direction": "download",
        "source": "/tmp/ip",
        "destination": "./host"
    }
]
```

Execute the `curl` command in the remote shell, then use the `file` provisioner to download that file. This will then create a file with the contents of the IP address that you can use for later steps.

# Solving Problem 2: Communicating over SSH

Now that I had the IP address, I need to communicate from Inpsec to that IP. Inspec has several ways of communicating with the machine. It can do it locally or even execute on a rmeote machine. I wanted to execute locally but communicate on the remote machine so that I didn't have to install anything on the remote box. This lead me down the path that I needed to use SSH to communicate with that remote box. By default SSH requires a username and password, and since I was going to be using this in an automated build environment. Having something prompt for a username and password wasn't going to fly, nevermind the fact that isn't very secure either. So I need to setup a SSH key somehow in the automated process to get logged into the remote box. 

* keygen
* GCP metadata

# The resulting Packer file

{{< highlight json "linenos=table" >}}
{
    "builders": [
        {
            "type": "googlecompute",
            "project_id": "{{user `gcp-project`}}",
            "source_image_family": "ubuntu-1804-lts",
            "zone": "us-central1-a",
            "image_description": "Image demo for SSH keys",
            "ssh_username": "packer",
            "tags": "packer",
            "image_name": "test-image-{{isotime | clean_image_name}}",
            "metadata": {
                "enable-oslogin": "false",
                "ssh-keys": "packer:{{user `inspec-key`}}"
            }
        }
    ],
    "provisioners": [
        {
            "type": "shell",
            "inline": ["curl metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip -H \"Metadata-Flavor: Google\" > /tmp/ip"]
        },
        {
            "type": "file",
            "direction": "download",
            "source": "/tmp/ip",
            "destination": "./host"
        },
        {
            "type": "shell-local",
            "command": "docker run --rm -v $(pwd):/workspace -w /workspace chef/inspec:3.2.7 detect . -t ssh://packer@$( cat host ) -i inspec-key"
        }
    ]
}
{{< / highlight >}}

Let me walk through these line by line:


