---
title: "Terraform: How I am setting up the Architecture"
date: 2020-03-06
---

I have been using, what I think at least, is a novel approach to structure my Terraform architecture. So far its been proven to at least work and help with the common day to day problems, and can scale to most needs for small to medium size projects.

My approach has been battle tested on Google's Cloud, so this approach may not be useful if you aren't there. I hope to one day test it out on Azure and AWS, but time, and my current focus, just hasn't let me to do that quite yet.

The first thing that we need to explore is where to store the Terraform state files. For simplicity sakes, I keep everything stored in GCP. This way everything associted with my project is in one place. There is a very good Hashicorp offering to stores these in that cloud. Which is a great option, but I just haven't been pushed enough to incorperate that into this workflow.


# The Overlord

Let me try and visually represent where things will be going first:

#TODO: Overview picture

Let me introduce the the Overlord project. This project houses the main service accounts that will be used by the pipelines, house any type of artifacts that will be produced and deploy into the projects, and then house the Terraform state files. The whole idea behind this project is that it is used as storage for things that are global. We want to make sure we have a central place for these things. This is the first project that gets created through manual steps, and then everything is turned over to have bots run these after that.

This project has a ramp up period where it is initially worked on to get the basic things setup. Then after that, it doesn't get touched much.

There are three service accounts that get used from this project: a Terraform service account, a Artifact pushing service account, and a Kubernetes deployment service account. The terraform service account is used to deploy terraform changes. And is used in any of the Terraform steps. The artifact pushing service account is used when there is a build artifact that needs to be used across the projects. This account has access to push that artifact (mostly docker images into GCR) into the representive repositories. The Kubernetes deployment service account is used to deploy to the Kubernetes environment.

# Environments

Now from there, I have a folder setup for each environment. And what I mean by environment is a development, staging, and production environments. I contain each environment in a folder so I can have global rules and permissions that I can have one place to set them on. Each environment folder can be made up of multiple projects for whatever the overall system or organization needs.

# Projects

Now for the meat of your infrastructure, the actual project space. A project is where you host an actual GCP project and put what ever infrastructure you need. This keeps all of the networking, instances, databases, etc in one logical group. I use a single definition that gets deploy over each of the different environments.

Each project in the environment folder has its own Terraform state file in the overlord repository. This keeps each statefile focused on one project and one environment. That way, if things go wrong. You aren't messing up other's statefiles and can keep things seperated and isolate. This is an attempt in trying to keep the blast radius of a bad deploy to the smallest possible footprint.

# Deploying

The pipelines look like any other build/deploy pipeline. They push into each of the environments in sequence and it has to go through those areas before it can get to production. So if you have a development/stage/production, it goes through each one before proceeding to the next one.

# Conclusion

There you have it, my current setup for Terraform. Again, this is working for me, but it might not for you. This is a very opinionated approach of how to do things and it probably won't suite everyones needs. However, I have sucessfully been using this now for a enough time that I have to be very persuaded to change much of what is going on.
