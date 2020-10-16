# Sample of Packer config to configure VMs for `machine`/`setup_remote_docker` to use Docker Hub pull through cache server

## Overview

Starting [November 1, 2020](https://www.docker.com/blog/scaling-docker-to-serve-millions-more-developers-network-egress/), Docker Hub will impose rate limits for anonymous pulls based on the originating IP. To avoid service disruption, it is recommended to authenticate Docker pulls with Docker Hub. You can authenticate using build config (see [Using Docker Authenticated Pulls](https://circleci.com/docs/2.0/private-images/)).

Alternatively, you can set up a Docker Hub pull through registry mirror pre-configured with Docker Hub account credentials. Using a pull through registry mirror is potentially simpler than making many build config modifications. It may also bring additional performance improvements since network roundtrips to Docker Hub are reduced.

Once you have already [set up pull through cache server for Docker Hub](https://circleci.com/docs/2.0/docker-hub-pull-through-mirror/), you also can configure VMs spun up for `machine` executors or VMs for jobs calling `setup_remote_docker` to use the pull through cache server as well. With this configuration, layer pulls ocurred on image builds _without_ authentication (i.e. `docker build` commands executed _before_ calling `docker login`) will also be covered by the pull through cache, as well as unauthenticated `docker pull` or `docker run`. As a result, you can mitigate impacts of the rate limit imposed to VMs dynamically created by CircleCI for jobs.

To achive this objective, you need to create a custom AMI that contains a configuration for Docker daemon (`dockerd`) to use your own pull through cache server. Namely, you have to:

1. create a custom AMI containing a custom configuration for `dockerd`, which is the main scope of this repository, and then
2. configure your CircleCI instance to use the AMI, following [the instruction in our official documentation](https://circleci.com/docs/2.0/vm-service/#customizing-and-creating-vm-service-images).

This repository provides a Packer config and peripheral scripts which help create custom AMIs for the purposes discussed above.

## How to create a custom AMI

This packer script will run on CircleCI, and add custom touches on the default AMI used by CircleCI. Here are the steps to follow:

1. Prerequisites:

    1. Make sure that you have **your own copy of this repository** on your GitHub organization.

    2. Make sure that you have **an access key** for an AWS IAM User that has sufficient permission to create AMIs using Packer. Refer to [documentation of Packer](https://www.packer.io/docs/builders/amazon#authentication) for details.

2. **Set up a CircleCI project**. You can use your own CircleCI Server on AWS for this.

3. **Configure environment variables**. Following 4 variables are required to make the entire parts functional.

    | Environment variable | Description |
    |-|-|
    | `AWS_ACCESS_KEY_ID` | Access key ID of your access key |
    | `AWS_SECRET_ACCESS_KEY` | Secret access key of your access key |
    | `AWS_DEFAULT_REGION` | Region where your CircleCI Server is hosted, e.g. `ap-noartheast-1` |
    | `DOCKER_MIRROR_HOSTNAME` | Hostname of your Docker Hub pull through cache, e.g. `http://192.0.2.0` |

4. **Run the workflow on CircleCI**. You can push a new commit, or just hit "Rerun workflow".

5. In the "Summarize results" step of the resulting job, you will see an AMI ID for the generated AMI. **Configure your server to use the newly created AMI by specifying the AMI ID**, as [illustrated here](https://circleci.com/docs/2.0/vm-service/#customizing-and-creating-vm-service-images). Note that you need to **fill both "Custom Linux VM AMI" and "Custom Remote Docker AMI"** with the new custom AMI ID.
