---
title: "Terraform: How I am setting up the Architecture"
date: 2020-03-06
---

I have been using, what I think at least, is a novel approach to structure my Terraform architecture. So far its been proven to at least work and help with the common day to day problems, and can scale to most needs for small to medium size projects.

My approach has been battle tested on Google's Cloud, so this approach may not be useful if you aren't there. I hope to one day test it out on Azure and AWS, but time, and my current focus, just hasn't let me have a project to do that quite yet.

The first thing that we need to explore is where to store the Terraform state files. I know there cloud solutions from Hashicorp that can probably do this better. But I haven't had time to fully explore it, and I really don't want another invoice to process. This means that my architecture will fully rely on GCP and store the state files in GCP buckets.

# The Overlord

Let me introduce the storage location called the Overlord project. This project houses the main service accounts that will be used by the pipelines, house any type of artifacts that will be produced and deploy into the projects, and then house the Terraform state files. The whole idea behind this project is that is used as storage for things that are global. We want to make sure we have a central place for these things. This is the first project that gets created through manual steps, and then everything is turned over to have bots run these after that.

# Environments

Now from there, I have a folder setup for each environment. And what I mean by environment is a development, staging, and production environments. I contain each environment in a folder so I can have global rules and permissions that I can have one place to set them on. Each environment folder can be made up of multiple projects for whatever the overall system or organization needs.

Each project in the environment folder has its own Terraform state file in the overlord repository. This keeps each statefile focused on one project and one environment. That way, if things go wrong. You aren't messing up other's statefiles and can keep things seperated and isolate. This is an attempt in trying to keep the blast radius of a bad deploy to the smallest possible footprint.

# Deploying
