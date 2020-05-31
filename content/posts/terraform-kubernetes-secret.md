Here is the situation: I want a secret (let's say a database password) stored inside a Kubernetes secret. The infrastructure is controlled by Terraform. So, how do I go about keep those two things in sync?

Well, recently I found out a way! Its pretty neat, you can connect Terraform with both GCP infrastructure, and Kubernetes at the same time, and you can keep them in sync! How you say? Well, let me show you!

Let's look at the full blown example and then we will break the interesting bits down:
