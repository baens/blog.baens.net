---

TODO:
[ ] Add links
[ ] Add code repository of the whole example
---

Here is the situation: I want a secret (let's say a database password) stored inside a Kubernetes secret. The infrastructure is controlled by Terraform. So, how do I go about keep those two things in sync?

Well, recently I found out a way! Its pretty neat, you can connect Terraform with both GCP infrastructure, and Kubernetes at the same time, and you can keep them in sync! How you say? Well, let me show you!

Let's look at the full blown example and then we will break the interesting bits down:

provider "google" {
}

data "google_client_config" "default" {
}

provider "kubernetes" {
  load_config_file = false
  host             = "https://${google_container_cluster.primary.endpoint}"
  token            = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(
    google_container_cluster.primary.master_auth[0].cluster_ca_certificate
  )
}

resource "kubernetes_namespace" "test" {
  metadata {
    name = "test"
  }

  depends_on = [google_container_node_pool.primary]
}

```

Ok, ok. This isn't a full blown example, but these are the most important pieces. The full example can be found in my example repository here. Here I am just highlighting the pieces you probably already want and need.

The first big piece you will need is the `google_client_config`. This piece is the one that actually led me down this road in the first place. This gives information back to Terraform on how to access the Google API and how to use other pieces. The most important piece here is the access token. This is a short lived token that will be used to communicate with the cluster with whatever privliege the gcloud command (or Terraform provider context) has.

As you may notice, the `google_container_cluster` give us the other pieces. It gives us the host, along with the CA certificate of the host. This then gives you full access to the kubernetes instance and allows you to start managing the instance with kubernetes!

Now the next bit that may trip people up is the `depends_on`. Because the provider doesn't naturally give you dependencies. Terraform can't figure out the dependency graph on its own, so it needs some help. We specically target the node pool because we want actual node workers in kubernetes for certain things to work (i.e. namespace is a good example that needs a worker node to work fully). If you don't have this, destorying and or other massive changes might leave you in a stuck state.
