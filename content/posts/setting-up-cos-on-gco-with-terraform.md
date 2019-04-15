---
title: "Setting up a Docker container on a Container-Optimized OS running on the Google Cloud with Terraform"
date: 2019-04-15
---

This blog post will expand on a previous blog post that sets up a Container-Optimized OS in Google Cloud. But instead of doing all the commands manually, here is what it would take to do this with Terraform. I won't go into exactly what Terraform is or the benefits here. But I would highly recommend checking it out and using it to manage your cloud resources. This blog will focus on the Terraform file that would replicate the previous blog post functionality.

Let's see what the terraform file would look like:

{{< highlight yml "linenos=table">}}
provider "google" {
    version = "2.3.0"
}

resource "google_compute_firewall" "http-traffic" {
    name = "allow-http-traffic"
    network = "default"

    allow {
        protocol = "tcp"
        ports = ["80"]
    }

    source_tags = ["http-traffic"]
}

data "template_file" "cloud-init" {
    template = "${file("cloud-init.yml.tmpl")}"

    vars = {
        registry = "${ }"
        username = "${ }"
        password = "${ }"
        image = "${ }"
    }
}

resource "google_compute_instance" "cos-instance" {
    name = "cos-instance"
    machine_type = "n1-standard-1"

    tags = ["http-traffic"]

    boot_disk {
        initialize_params {
            image = "cos-cloud/cos-stable-72-11316-136-0"
        }
    }

    network_interface {
        network = "default"
        access_config {
            // this empty block creates a public IP address
        }
    }

    metadata {
        "user-data" = "${data.template_file.cloud-init.rendered}"
    }
}
{{< / highlight >}}
