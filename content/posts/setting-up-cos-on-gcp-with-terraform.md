---
title: "Setting up a Docker container on a Container-Optimized OS running on the Google Cloud with Terraform"
date: 2019-07-01
---

This blog post will expand on a previous blog post that sets up a [Container-Optimized OS in Google Cloud](https://blog.baens.net/posts/setting-up-cos-on-gcp/). But instead of doing all the commands manually, here is what it would take to do this with [Terraform](https://www.terraform.io). I won't go into exactly what Terraform is or the benefits here. But I would highly recommend checking it out and using it to manage your cloud resources. This blog will focus on the Terraform file that would replicate the previous blog post functionality.

Let's see what the terraform file would look like:

{{< highlight yml "linenos=table">}}
provider "google" {
  version = "2.9.1"
  project = "${var.project}"
  zone    = "${var.zone}"
}

resource "google_compute_firewall" "http-traffic" {
  name    = "allow-http"
  network = "default"

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  target_tags = ["http-traffic"]
}

resource "google_compute_firewall" "http-ssh" {
  name    = "allow-ssh"
  network = "default"

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  target_tags = ["ssh-traffic"]
}

data "template_file" "cloud-init" {
  template = "${file("cloud-init.yml.tmpl")}"

  vars = {
    registry = "${var.registry}"
    username = "${var.username}"
    password = "${var.password}"
    image    = "${var.image}"
  }
}

resource "google_compute_instance" "cos-instnace" {
  name         = "cos-instance-${substr(md5(file("cloud-init.yml.tmpl")),0,10)}"
  machine_type = "n1-standard-1"

  tags = ["http-traffic", "ssh-traffic"]

  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-73-11647-217-0"
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

output "address" {
  value = "${google_compute_instance.cos-instnace.network_interface.0.access_config.0.nat_ip}"
}

{{< / highlight >}}

There isn't too much magic in there and a fairly straight forward GCP terraform configuration. As you may notice, I concentrated on just the compute instance and the firewall rules. This doesn't actually create the Google Cloud project, and actually assumes you already have one up and running.

The magic happens in blocks at line 19 and line 50. Line 19 slurps up the template file and does string replacements with the `vars` defined in the corresponding block. That rendered template file is then attached to the correct metadata `user-data` variable.

Now that we have all of this, how do we run this? Grab the [code in this repository](https://github.com/baens/code-examples-blog.baens.net/tree/cos-gcp-with-terraform) to help you run it. After that, make sure to use Terraform 0.11 for this post (sorry, haven't upgraded things to 0.12 yet). Once that is done, go ahead and run `terraform init` and make sure things install ok. 

Now that is done, we could go ahead and try and use `terraform plan`. But that is going to get annoying fast to keep getting asked all the different variables. Let's go ahead and take the `terraform.tfvars.tmpl`, copy that over to `terraform.tfvars` and fill out each variable. Then go ahead and run `terraform plan` and check that things are going to work out. If that plan looks out, go ahead and run `terraform apply`. With that, things should be created and then you can view your instance at the outputted IP address. 