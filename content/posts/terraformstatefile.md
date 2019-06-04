---
title: "Decomposing your Terraform state file"
date: 2019-03-08
tags: ["terraform", "statefile", "refactor"]
draft: true
---

### Decomposing terraform state

#### Contents

* [Overview](#overview)
* [State file](#statefile)
  * [Remote state]
  * [Locking State](#locking state)
    * [Dynamodb table]
* [Resources and Data Sources](#resources_and_data_source)
* [State as Data](#state_as_data)
* [Decomposing environments into components](Decomposing_environments_into_components)
* [Migrating resources between state files](#migrating)
  * state command


#### Overview

After running terraform for awhile now things gotten to the point where it’s time to refactor.Initially I had read Charity Majors’ now [canonical post](https://charity.wtf/2016/03/30/terraform-vpc-and-why-you-want-a-tfstate-file-per-env/) on implementing terraform with multiple state files.  Following along I organized my terraform directory with separate environments for dev, stage, and prod.

        terraform/
        ├── env-dev
        ├── env-iam
        ├── env-prod
        ├── env-stage
        ├── env-test
        └── modules

As things go, we’ve added more and more resources with terraform  and I’ve grown uncomfortable with the thought of potentially corrupting the state file.  I’ve got the S3 backend set up in a versioned bucket, and I know I can dump a local copy of the state file any time I’m making major changes, still the blast radius of a failed state file is more than I want to deal with on any given day.

Thanks to the suggestions of [Alexander Savchuk](https://github.com/PacktPublishing/Hands-on-Infrastructure-Automation-with-Terraform-on-AWS) / [(endofcake)](https://github.com/endofcake), and the good people at [terragrunt](https://blog.gruntwork.io/how-to-manage-terraform-state-28f5697e68fa), I'll be refactoring terraform to accommodate component level state files.

Terraform initially introduced environments in v0.9, and then in v0.10 changed environements to workspaces.  According to the documentation this was to alleviate a certain amount of confusion that had become associated with the notion of environment, but for me, at any rate, I found this transformation to be the source of my confusion.  How is a workspace supposed to be any different than an environment when it exhibits exactly the same behavior?  I still don’t think I can answer that last question, but after having read … and … I’m much more comfortable with decomposing my environment level state files into component/service level workspaces.

Thanks to the good people … I’m going to decompose my environment level state files into component/service level workspaces.



#### State File

In it's minimal form the terraform state file is initialized from the terraform block in your configuration and placed in ./.terraform/terraform.tfstate.

    terraform {
      required_version = ">= 0.11.11"
    }

Interpolations are not active in the terraform block, and all values have to be literal, since [the terraform block runs before interpolations have been loaded](https://www.terraform.io/docs/configuration/terraform.html).  

Adding a backend to the terraform block moves all of state data from the local state files (./.terraform/terraform.tfstate) to the designated backend.  There are several [types of backends](https://www.terraform.io/docs/backends/types/index.html) available, and with most of our resources being in AWS, S3 was like the most direct choice.  After configuring a versioned bucket the terraform block looks like this,

    terraform {
      required_version = ">= 0.11.11"
      backend "s3" {
        bucket         = "mybucket"
        key            = "path/to/object/key"
        region         = "aws_region"
      }
    }

With the backend configured, running `terraform init` will migrate the current state data from the local state file to the configured backend.

Having recently added a new team member I'm no longer the only one working on terraform, so locking the state file has taken on some urgency.  Introduced in v0.9, state locking uses a dynamodb table to lock the backend terraform state file, ensuring that if multiple `terraform apply` commands are running, only one will have access to the state file at any given time.  To get started we need to configure the dynamodb table to hold the state lock,

    resource "aws_dynamodb_table" "tf_workspace_lock_table" {
      name           = "tf-workspace-lock-table"
      hash_key       = "LockID"
      read_capacity  = 10
      write_capacity = 10

      attributes {
        name = "LockID"
        type = "S"
      }
    }

Note: [the hash_key needs to be defined both in the index and as an attribute](https://www.terraform.io/docs/providers/aws/r/dynamodb_table.html#hash_key)

We can also set up a policy to provide access to the lock table,

    resource "aws_iam_policy" "compoenent_example_state_lock_table_policy" {
      name        = "component-example-state-lock-table-policy"
      path        = "/envDev/"
      description = "Access policy for env-dev state lock table"

      policy = <<EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "dynamodb:GetItem",
            "dynamodb:PutItem",
            "dynamodb:DeleteItem"
          ],
          "Resource": "arn:aws:s3:::mybucket/path/to/my/key"
        }
      ]
    }
    EOF
    }

With the lock table in order we can add the dynamo_table attribute to the terraform block,

    terraform {
      required_version = ">= 0.11.11"
      backend "s3" {
        bucket         = "mybucket"
        dynamodb_table = "tf_workspace_lock"
        key            = "path/to/object/key"
        region         = "us-east-1"
      }
    }

Now, whenever `terraform apply` is run the lock table will provide exclusive access to the workspace's state file in the backend bucket.


#### Resource and Data Sources

Terraform provides two primary mechanisms for managing cloud based infrastructure: resources and data sources.  As the documentaiton points out, "resources are a compoenent of your infrastructure."(https://www.terraform.io/docs/configuration/resources.html)  The terraform 'resource' key word provides full CRUD (Create, Read, Update, Delete) operations on theinfrastructure elements that you manage with terraform.  Addiitonally, the documentation defines data sources a way to bring external, or existing, infrastructure into your terraform configuration. "[D]ata sources allow a Terraform configuration to build on information defined outside of Terraform"  And while this is true, maybe a more significant notion is that data sources provide read only access to existing infrastructure, and, once it's been instantiated, any resource can be a data source. That is, even resources that have been defined internally by terraform can be read as data sources.

This is significant where the state file is concerned in that the data source for terraform_remote_state provides a non-volatile, read-only mechanism for interacting with state data from your current workspace, or from any other workspace in your configuration.  Given the consequences of a corrupted state file, this is a pretty cool feature.

#### State as Data

If you've already set up a back end this bit may already be familiar.  After initially setting up your terraform provider,

  provider "aws" {
    region = "${var.aws_region}"
  }

you probably set up a data source for the back end that looks very similar to your terraform block.  In my case, having set up separate environments for dev, stage, and prod, I had the following data source for my dev environment

    data "terraform_remote_state" "dev" {
      backend = "s3"
      config {
        bucket = "terraform-state-dev"
        key    = "dev/terraform.tfstate"
        region = "us-east-1"
      }
    }

At this point, any output that's been configured in your terraform state file can be read directly from the remote state data source, using the standard terraform interpolation rules.  That is, we can get the value for the output 'foo' with the following interpolation,

    "${data.terraform_remote_state.env-dev.foo}"

Now that we can read data from the state file, the state file table policy can be written to read the table key from the state file,

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "dynamodb:GetItem",
            "dynamodb:PutItem",
            "dynamodb:DeleteItem"
          ],
          "Resource": "${data.terraform_remote_state.self.state_lock_table_arn}"
        }
      ]
    }

#### Decomposing environments into components

Now we're ready to start moving the logical components of our infrastructure out of the environment level state file, and into component or service level state files.  When we're done the terraform hierarchy is going to look something like this,

    terraform/
    ├── .terraform
    ├── provider.tf
    ├── dev
    │   ├── services
    │   │   ├── app-server
    │   │   ├── image-server
    │   │   ├── service-example
    │   │   └── web-server
    │   └── vpc
    └── stage
    |   ├── services
    |   │   ├── app-server
    |   │   ├── image-server
    |   │   └── web-server
    |   └── vpc
    ├── prod
    │   ├── services
    │   │   ├── app-server
    │   │   ├── image-server
    │   │   └── web-server
    │   └── vpc
    └── site
        ├── iam
        └── route53

To migrate service-example to it's own workspace we'll start by setting the terraform block,

    terraform {
       required_version = ">= 0.11.11"

       backend "s3" {
         bucket         = "mybucket"
         key            = "dev/component_example/terraform.tfstate"
         dynamodb_table = "dev_component_example_state_lock_table"
         region         = "us-east-1"
       }
     }

and giving it a provider,

    provider "aws" {
      region = "${var.aws_region}"
    }

Now we can set the terraform_remote_state data, but we'll proceed slightly differently than we did with the flat hierarchy of a single state environment.  First we'll set the data source for the new workspace,

    data "terraform_remote_state" "self" {
      backend   = "s3"
      workspace = "component_example"

      config {
        bucket = "${data.aws_s3_bucket.state_bucket.id}"
        key    = "dev/${terraform.workspace}/terraform.tfstate"
        region = "${data.aws_s3_bucket.state_bucket.region}"
      }
    }

and we'll follow up by setting the terraform_remote_state data for the parent workspace as well,

    data "terraform_remote_state" "parent" {
      backend   = "s3"
      workspace = "dev"

      config {
        bucket = "${data.aws_s3_bucket.state_bucket.id}"
        key    = "dev/terraform.tfstate"
        region = "${data.aws_s3_bucket.state_bucket.region}"
      }
    }

Provided that each workspace exports all infrastructure attributes as outputes, which should include both resources and data sources, then self and parent states will be sufficient for managing an N level hierarchy.  That is, regardless of how deep parent is in the hierarchy, self will be able to access all values above it, since higher level attributes with be exported as data sources from the parent state file.    

####  Migrating resources between state files

With both workspaces configured we're ready to move our current infrastructure into it's own state file.  First we'll need to move the service-example module and outputs into the new workspace directory,

  $ pwd
  teraform/dev  

  $ mv service-example.tf service-example

The terraform command is not able to migrate resources directly between backend state files, so we'll need make a local copy of both state files.  Migrate the resource from one state file to the other, and then push both state files back to their respective locations.

We'll start by pulling local copies of both state files from the backend,

    $ pwd
    terraform/dev

    $ terraform state pull > dev.tfstate

    $ cd service-example

    $ terraform state pull > service-example.tfstate

Since terraform reads the location of the backend from ./.terraform/terraform.tfstat the pull command must be issued from the directory where the workspace is configured

Now we can migrate the service-examples module to it's new location,

  $ terraform state mv -state=../dev.tfstate \
    -state-out=./service-example.tfstate module.service-example module.service-example

  $ terraform state push ./service-example.tfstate

You can verify that the resources have been moved to the new service-example backend,

    $ terraform state list | grep service-example

And finally we need to push the parent state to the backend as well,

    $ cd ..
    $ terraform state push ./dev.tfstate


A cauitonary note on pushing a local copy of the state file to the backend. You could think of pushing the state file end as kind of like upgrading the firmware of your terraform configuration.  That is, it's a non-atomic operation that, if things go wrong, could brick your configuration.  Well, it's not quite as bad as all that since we've got the versioned bucket to fall back on, but if something goes wrong with the push operation it's going to be a pain because you'll have to fall back on the previous version.  So proceed with caution.
