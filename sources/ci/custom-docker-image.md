page_main_title: Using a custom Docker image
main_section: CI
sub_section: Configuration
sub_sub_section: Advanced config
page_title: Using a custom Docker image
page_description: How to use a custom docker image to continuously build your project in Shippable.

#Using a custom Docker image

In most cases, your CI workflow should work fine with our official images. However, you might want to consider using a custom Docker image in the following situations:

- Your build has dependencies that take a long time to install if installed as part of your CI workflow.
- You want to build a project written in a language not officially supported by Shippable.
- You are using a combination of languages and tools not supported together in any official images.
- You want to run CI in your own Docker image to better simulate your production environment.

There are two available options to spin up your CI container with a custom image:

- [Pull an image](#pull-image) from a supported Docker registry and use it to spin up your CI container
- [Build an image](#build-image) and use it to spin up your CI container

##Minimum requirements

The Shippable platform officially supports custom images built on Ubuntu 14.04, Ubuntu 16.04, CentOS7, RHEL 7, and Windows Server 2016. However, many customers have successfully used custom images that are built on  unsupported platforms, as long as they satisfy minimum requirements mentioned below.

If your custom Docker image has `apt-get` pre-installed, we will install any additional packages required at runtime.

If your image does not have `apt-get` and you do not want it on your image for some reason, you should install the following packages:

* git
* ssh-agent or openssh
* sudo
* Python 2.7
* jq 1.5

In addition, it is recommended that you install any dependencies, test tools, language versions, CLIs, etc, in your custom image to avoid having to install these during each build. This will make your builds faster since installing them in your build scripts will add the installation time to each build.

We want to do our best to support your custom image, so please [open a support issue](https://github.com/Shippable/support/issues/new) if you're seeing errors while using your image with Shippable.

<a name="pull-image"></a>
##Pulling your custom image and using it for CI

The `pre_ci_boot` section lets you override the default CI image and use your own. You can pull images from both public and private repositories on any Docker registry.

###Pulling a public image

You can configure the `pre_ci_boot` section in your yml as shown below:

```
build:
  pre_ci_boot:
    image_name: my_registry_repo/my_image
    image_tag: latest
    pull: true
    env: FOO=bar
    options: "-e HOME=/root"

```
In the snippet above, replace the following:

* `image_name` value is in the format (docker-registry-username)/(docker-registry-image-repo). For GCR and ECR, you will to specify image_name in the right format:
    *  GCR: gcr.io/(docker-registry-username)/(docker-registry-image-repo)
    *  ECR: aws_account_id.dkr.ecr.us-east-1.amazonaws.com/repo-name
* `image_tag` is the tag for the image that you want to pull.  
* set `pull` to `true` if you want to pull this image from a docker registry and `false` if the image is already on the build machine and doesn't need to be pulled from a registry. If you built the image in the `pre_ci` section, the image should already be on the machine.
* In the `env` section, you can enter any environment variables you want to be set inside your CI container.
* In the `options` tag, enter any docker options you want to use in the `docker run` command. You also need to include the HOME environment variable as shown if it is not already set in your image.

###Pulling a private image

To pull a private image, you will first need to add an account integration for the registry you want to pull from. We will then authenticate against the registry and pull your image.

Follow the steps below to pull a private image:

- Create an account integration for the registry you want to pull from. For instructions, choose your registry below:
    - [Docker Registry](../platform/integration/dockerRegistryLogin/)
    - [Amazon ECR](../platform/integration/aws-keys/)    
    - [Google Cloud](../platform/integration/gcloudKey/)
    - [Quay](../platform/integration/quayLogin/)      

- Add the following to your `shippable.yml`


```
build:
  pre_ci_boot:
    image_name: my_registry_repo/my_image    ##replace with your repo and image name
    image_tag: latest
    pull: true
    env: FOO=bar
    options: "-e HOME=/root"

integrations:                 
  hub:
    - integrationName: my-docker-hub   ##replace with your integration name
      type: dockerRegistryLogin
```

* `integrationName` is the friendly name of the account integration you added. Please note that the name has to match exactly.
* `type` depends on the registry you are pulling from:
    * `dockerRegistryLogin` for Docker Hub and Docker Private/Trusted Registry
    * `ecr` for Amazon ECR.
        - Note: For running builds on docker version 17.06.0-ce and above, make sure that the `aws` cli installed in the image is 1.11.91 version or above.
        - Both the deprecated [ECR integration](/platform/integration/aws-ecr) and newer [AWS keys](/platform/integration/aws-keys) integration support the same YML structure.
    * `gcloudKey` for Google Container Registry (GCR) (If you're still using the deprecated [GCR integration](/platform/integration/gcr), set the `type` here to `gcr`. Both will work exactly the same)
    * `quayLogin` for Quay.io


<a name="build-image"></a>

##Building an image to use for CI

You can build a Docker image in the `pre_ci` section of your `shippable.yml` and then use it to spin up the CI container. Commands in the `pre_ci` section are run on the build machine, i.e. outside your CI container, so this gives you an opportunity to actually build your CI container as part of you workflow.  

Follow the steps below to build a Docker image and use it to spin up your CI container:

* Make sure the **Dockerfile** for the image you want to build is in your repository

* Include the following in your `shippable.yml`. This snippet builds a Docker image called **myImage** with the tag **latest** from a Dockerfile at the root of the repository. You can use your own image name and tag as needed:

```
build:
  pre_ci:
    - docker build -t my_registry_repo/my_image:latest .

  pre_ci_boot:
    image_name: my_registry_repo/my_image    ##replace with your repo and image name
    image_tag: latest
    pull: true
    env: FOO=bar
    options: "-e HOME=/root"

```

* If your Dockerfile contains a `FROM` command that needs a private image, you will also need to follow a couple of additional steps:
    * Create an account integration for the registry you want to pull from. For instructions, choose your registry below:
        - [Docker Registry](../platform/integration/dockerRegistryLogin/)
        - [Amazon ECR](../platform/integration/aws-keys/)    
        - [Google Cloud](../platform/integration/gcloudKey/)
        - [Quay](../platform/integration/quayLogin/)      
    *  Add the integration name to your `yml`:

```
integrations:                 
  hub:
    - integrationName: my-docker-hub   ##replace with your integration name
      type: dockerRegistryLogin
```

* `integrationName` is the friendly name of the account integration you added. Please note that the name has to match exactly.
* `type` depends on the registry you are pulling from:
    * `dockerRegistryLogin` for Docker Hub and Docker Private/Trusted Registry
    * `ecr` for Amazon ECR
    * `gcloudKey` for Google Container Registry (GCR) (If you're still using the deprecated [GCR integration](/platform/integration/gcr), set the `type` here to `gcr`. Both will work exactly the same)
    * `quayLogin` for Quay.io

This should configure your workflow to build `my_registry_repo/my_image` image and use it for your CI.
