---
title: "Making Inspec with SSH work with Packer on the Google Cloud Platform"
date: 2019-01-11
---

# Background

This blog post aims to help anyone trying to make a [Packer](https://www.packer.io/) image, tested with [Inspec](https://www.inspec.io/), on the [Google Cloud Platform](https://cloud.google.com/). I have recently been using the these tools for my day-to-day and I wanted to make these three tools work together, which was surprisingly difficult. This blog post will hopefully help anyone out there that may be trying todo the same.

What I want from these tools is have Packer spin up a new VM, run all the build steps, then test that VM with Inspec. Also, I didn't want to clutter up the VM with anything more installed then it really needed, so I didn't want Inspec running with all of its tools installed on the new VM. That means I wanted Inspec to run from my box then connect through SSH to the new box. Sounds simple right? Well, there were enough gotchas and lack of search resources out there that I felt I needed to write this up.

So enough yap, let's see what the code looks like. If you want to see everything put together, [code repository here](https://github.com/baens/code-examples-blog.baens.net/tree/packer-gcp-ssh-key).

# The problems I faced

So let me walk through the problems I encountered along this journey that I will try and demonstrate answers to.

Problem #1: The first problem I hit was that I needed Inspec to talk to the new VM, but Packer doesn't provide those variables very easily. What I needed was the host IP of the VM running in the cloud. Since Packer didn't provide that out of the box, I had to figure out a way to get around that.

Problem #2: Once I did establish a connection, I then needed to authenticate Inspec with the VM. Inspec could natively connect over SSH, so creating and establishing a SSH key to login with was the obvious choice. However, Packer again didn't provide a convenient way of doing this so I need to establish that key myself and set a few manual things to make that work.

# Solving Problem 1: Getting the IP of the Packer VM in GCP

This sounds easy enough but is surprising difficult: how do I get the IP of the currently running VM that Packer is communicating with? One approach might ping some remote web site that tell you your public IP, but that seems ridiculous to introduce another layer outside of your control. My search stumbled upon a number of examples of how to do this on AWS but very little on GCP. However, people using AWS did give me an idea of what todo, or at the very least what to search for.

What people use on AWS often is called the Metadata server. The Metadata server is a service for every VM running on [AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) (and [GCP](https://cloud.google.com/compute/docs/storing-retrieving-metadata)) that can tell you about the instance you are running on. This was the thing I needed, I found the [documentation for GCP](https://cloud.google.com/compute/docs/storing-retrieving-metadata) and sure enough, it had information on all of the exposed IP addresses. The particular address I needed was located at this URL: `http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip`. And whats even better was that this was generic across instances so I didn't have to do some fancy logic depending on the box. 

Now that I have the HTTP address of where I can get the external IP address, how would I exactly retrieve that information from the command line. Thankfully, curl was already installed so I crafted this command:

```
curl  -H \"Metadata-Flavor: Google\" metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip
```

And executed that and viola, the IP address I needed. 

Now, something to be pointed out. That HTTP header we added (`Metadata-Flavor: Google`) is required. This is a safe guard to protect you incase you accidentally point something wrong to this location. 

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

And that solved the problem. Just have to create a file with data from a special internal endpoint, then download that file for use in next steps. Easy! Or at least easy once you work it out. Go figure!

# Solving Problem 2: Communicating over SSH

Now that I had the IP address, I need to communicate from Inpsec to that IP. Inspec has several ways of communicating with the machine. It can do it locally or even execute on a remote machine. I wanted to execute locally but communicate on the remote machine so that I didn't have to install anything on the remote box. This lead me down the path that I needed to use SSH to communicate with that remote box. By default SSH requires a username and password, and since I was going to be using this in an automated build environment. Having something prompt for a username and password wasn't going to fly, never mind the fact that isn't very secure either. So I need to setup a SSH key somehow in the automated process to get logged into the remote box.

[Google cloud provides a way for you to provide a SSH key](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys#edit-ssh-metadata) that will get installed on the box for you to use. So first let's generate the key. The way Google parses the key information is it looks for comments inside of the key for which user is associated with this. Here is the command I use to generate the key:

```bash
ssh-keygen -f inspec-key -C packer -N '' -m PEM
```

The command will output a set of SSH key files called `inspec-key` with a comment of `packer` and the `-N` flag is the password to be used (blank in this case). You need to specify the PEM format because some of the Ruby modules that will be loaded can't parse newer OpenSSH key formats. 

Alright, I now have my SSH key, now I need to place that SSH key for Packer to use when it creates the VM instance. For that you need to modify the builder with a few parameters. Here is what the builder turns into:

```json
"builders": [
        {
            "type": "googlecompute",
            ...
            "metadata": {
                "ssh-keys": "packer:{{user `inspec-key`}}"
            }
        }
    ],
```

And here is what the command would look like to run Packer:

```bash
packer build -var inspec-key=$(cat inspec-key.pub) image.json
```

The `metadata/ssh-keys` is the important part. You add the metadata with the username, then the contents of the key. This tripped me up at first because I thought the key had already been setup with all of the relevant information, but trial and error found the right format. And just to be clear, the format is `<username>:<ssh public key data>`. 

Alright, now that we have our new instance up and running, how do we connect Inspec? Well, again, I didn't really want to have to install another tool so let's use docker to run the actual tool. The docker command will look like this:

```bash
docker run --rm -v $(pwd):/workspace -w /workspace chef/inspec:3.2.7 detect -t ssh://packer@$( cat host ) -i inspec-key
```

Few things to explain in this:

* `-v $(pwd):/workspace -w /workspace` - Take your current directory and mount it at `/workspace` and make the current directory when the docker container is running inside of that directory.
* `chef/inspec:3.2.7 detect` - The docker image and the command to run. For the demo I used `detect` just to show you it connects. When you actually want to run tests you run `exec`.
* `-t ssh://packer@$( cat host ) -i inspec-key` - Here are a few of the magic bits. Tell Inspect to connect through `ssh` with the data from the `host` file we downloaded in the previous step. Next, use the `inspec-key` file we created earlier. 

# The resulting Packer file and the test run

Alright, all together this will look like this:

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
            "command": "docker run --rm -v $(pwd):/workspace -w /workspace chef/inspec:3.2.7 detect -t ssh://packer@$( cat host ) -i inspec-key"
        }
    ]
}
{{< / highlight >}}

The builder is setup with our keys for us to connect. The tool then downloads the IP of the server, and we use that with Inspec to connect to the host. It should look something like this when it runs:

![Demo of the build](/images/cloud-build.gif)

[All source code can be found on github if you want to run this yourself.](https://github.com/baens/code-examples-blog.baens.net/tree/packer-gcp-ssh-key)


