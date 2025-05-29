# Terraform Associate Certification Preparation Course

## Introduction to Terraform and Infrastructure as Code (IaC)

Terraform is *an infrastructure as code tool* that lets you build, change, and version cloud and on-prem resources safely and efficiently.  Infrastructure as Code (IaC) means **capturing manual provisioning tasks in code** so you can automate them consistently. As HashiCorp’s co-founder Armon Dadgar explains, IaC codifies the steps you would normally click through in a GUI, so you can run them repeatedly: *“how do we take the things we were pointing and clicking to achieve and capture that in a codified way? ... I can automate that.  Every morning, I can hit a script that brings up a thousand machines”*. By storing this code in version control, you get *versioning and history* of your infrastructure: *“I’ve captured it as codified form, I can check it into version control and I get versioning.  Now I can see an incremental history of who changed what”*. This makes infrastructure transparent, reproducible, and auditable, reducing human error and speeding up provisioning.

**Examples:** Below are simple Terraform configurations (main.tf) defining a basic resource in AWS, Azure, or GCP, illustrating IaC for each provider.

* **AWS Example:** Define an EC2 instance in AWS using HCL.

  ```hcl
  provider "aws" {
    region = "us-east-1"
  }

  resource "aws_instance" "web" {
    ami           = "ami-0c55b159cbfafe1f0"
    instance_type = "t2.micro"
  }
  ```
* **Azure Example:** Define a Resource Group in Azure.

  ```hcl
  provider "azurerm" {
    features = {}
  }

  resource "azurerm_resource_group" "rg" {
    name     = "example-resources"
    location = "West US"
  }
  ```
* **GCP Example:** Define a VPC network in GCP.

  ```hcl
  provider "google" {
    project = "my-gcp-project-id"
    region  = "us-central1"
  }

  resource "google_compute_network" "vpc" {
    name                    = "example-network"
    auto_create_subnetworks = true
  }
  ```

**Tips:** Store your Terraform code in version control (e.g. Git) so you can review changes via pull requests and revert if needed. Use descriptive names for resources and modules. Keep variable values (secrets) out of code; use [Terraform Cloud/Enterprise](#terraform-cloud-and-enterprise) or environment variables for sensitive data.

**Quiz:**

1. **What does “Infrastructure as Code” (IaC) mean?**
   *Answer:* IaC means managing and provisioning infrastructure (servers, networks, etc.) through machine-readable code instead of manual steps. This allows automation and version control of infrastructure changes.
2. **Why is using code to define infrastructure beneficial?**
   *Answer:* Because it enables automation, repeatability, versioning (history of changes), and collaboration. Terraform’s code can be checked into a VCS to track who changed what, reducing human error.
3. **Name three popular cloud providers supported by Terraform.**
   *Answer:* AWS, Azure, and Google Cloud Platform (GCP) are major providers. Terraform has providers for these and many others.
4. **How does Terraform apply your configuration to real infrastructure?**
   *Answer:* Terraform generates an execution plan (showing create/update/delete steps) and then performs those actions via provider APIs when you run `terraform apply`. The workflow is *Write* (create config), *Plan*, and *Apply*.
5. **What is the default file name for Terraform’s state file?**
   *Answer:* By default Terraform writes state to `terraform.tfstate` in the working directory.

## Terraform Providers

Terraform *providers* are plugins that let Terraform interact with cloud platforms, services, and APIs. Providers define the available resource types for an infrastructure platform.  *“Terraform relies on plugins called providers to interact with cloud providers, SaaS providers, and other APIs”*.  Each resource type (e.g. `aws_instance` or `azurerm_resource_group`) is implemented by a provider, and without providers Terraform can manage no resources.

In your configuration, you **must declare required providers** so Terraform can install and use them. You also typically configure them (e.g. region, credentials). In Terraform 0.13+ it is best practice to lock provider versions in a [`terraform` block](https://developer.hashicorp.com/terraform/language/settings/terraform) to ensure reproducibility.

* **AWS Provider Configuration:**

  ```hcl
  terraform {
    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = "~> 4.0"
      }
    }
  }
  provider "aws" {
    region = "us-east-1"
  }
  ```
* **Azure Provider Configuration:**

  ```hcl
  terraform {
    required_providers {
      azurerm = {
        source  = "hashicorp/azurerm"
        version = "~> 3.0"
      }
    }
  }
  provider "azurerm" {
    features = {}
  }
  ```
* **GCP Provider Configuration:**

  ```hcl
  terraform {
    required_providers {
      google = {
        source  = "hashicorp/google"
        version = "~> 4.0"
      }
    }
  }
  provider "google" {
    project = "my-gcp-project-id"
    region  = "us-central1"
  }
  ```

**Best Practices:** Always pin provider versions using a `terraform { required_providers { ... } }` block to avoid unexpected upgrades. Store provider configs (like credentials or regions) in a shared variables or backend, not in code, for security. Initialize providers with `terraform init` to download them.

**Quiz:**

1. **What is a Terraform provider?**
   *Answer:* A provider is a plugin that knows how to manage resources of a certain type (e.g. AWS, Azure). Providers implement resource and data source types, and Terraform relies on them to interact with cloud APIs.
2. **How do you tell Terraform which provider to use?**
   *Answer:* By declaring it in the configuration. You use a `provider "NAME" { ... }` block (plus optionally a `required_providers` setting) to configure a provider. Terraform will then download and initialize that provider.
3. **Why should you constrain provider versions in Terraform?**
   *Answer:* To ensure consistent behavior. Locking versions prevents Terraform from installing a new incompatible provider version unexpectedly. You can do this with a `required_providers` block.
4. **Give an example of a provider meta-argument.**
   *Answer:* The `provider` meta-argument can be used to select a specific provider configuration for a resource. For example: `provider = aws.uswest2` if you aliased multiple AWS providers. (Other meta-arguments like `count`, `for_each` apply to resources as well.)
5. **Can Terraform manage resources without declaring any providers?**
   *Answer:* No. Terraform needs a provider for each resource type. Without declaring providers, you cannot manage cloud infrastructure with Terraform.

## Terraform Resources

Resources are the *building blocks* of Terraform configurations. Each `resource` block describes one or more infrastructure objects (servers, networks, databases, etc.). *“Resources are the most important element in the Terraform language. Each resource block describes one or more infrastructure objects, such as virtual networks, compute instances, or higher-level components…”*. A resource block has a **type** (like `aws_instance` or `azurerm_resource_group`) and a **name** (an arbitrary identifier) and includes configuration arguments specific to that resource.

* **AWS Example:** Define an EC2 instance.

  ```hcl
  resource "aws_instance" "web" {
    ami           = "ami-0c55b159cbfafe1f0"
    instance_type = "t2.micro"
    tags = {
      Name = "web-server"
    }
  }
  ```
* **Azure Example:** Define a Virtual Network (in addition to a resource group).

  ```hcl
  resource "azurerm_resource_group" "rg" {
    name     = "example-resources"
    location = "West US"
  }
  resource "azurerm_virtual_network" "vnet" {
    name                = "example-vnet"
    address_space       = ["10.0.0.0/16"]
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
  }
  ```
* **GCP Example:** Define a Compute Engine instance.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = "example-network"
    auto_create_subnetworks = true
  }
  resource "google_compute_instance" "vm" {
    name         = "example-vm"
    machine_type = "f1-micro"
    zone         = "us-central1-a"
    boot_disk {
      initialize_params {
        image = "debian-cloud/debian-11"
      }
    }
    network_interface {
      network = google_compute_network.vpc.id
    }
  }
  ```

**Notes:** Resource names (`"web"`, `"rg"`, etc.) are local to the module and serve as references. You can refer to another resource’s attributes via expressions (e.g., `azurerm_resource_group.rg.name`). Terraform will track these resources in the state file.

**Best Practices:** Give logical names to resources (e.g. `web_server` instead of `web`). Avoid hard-coding sensitive data in resource arguments. Use interpolation (e.g. `${}` or `resource.type.name.attribute`) to reference attributes of other resources. Keep related resources together and add comments if needed.

**Quiz:**

1. **What does each `resource` block in Terraform describe?**
   *Answer:* It describes one or more infrastructure objects (like a server, network, database). The block contains the resource type and a name, plus provider-specific arguments.
2. **In the resource block `resource "aws_instance" "web" {...}`, what are `"aws_instance"` and `"web"`?**
   *Answer:* `"aws_instance"` is the resource **type** (managed by the AWS provider), and `"web"` is the resource **name** (a local identifier within the module).
3. **Can a Terraform configuration manage resources across multiple providers?**
   *Answer:* Yes. Terraform can include resources from multiple providers in one configuration (e.g., some AWS and some Azure resources), as long as each provider is declared.
4. **How do you reference an attribute of another resource?**
   *Answer:* Use an expression of the form `resource_type.resource_name.attribute`. For example, `aws_instance.web.public_ip` to get the public IP of the `web` instance.
5. **What happens if two resources have the same type and name in the same module?**
   *Answer:* Terraform would report an error because resource type and name must be unique within a module (they uniquely identify the resource in state).

## Modules

A **module** is a container for multiple Terraform resources that are used together. Every Terraform configuration has at least one module (the *root* module). You can create *child modules* and call them from your root configuration. Modules help package and reuse infrastructure code. They can be local directories or sources from the Terraform Registry or Git.

* **AWS Example:** Call a VPC module from the Terraform Registry.

  ```hcl
  module "vpc" {
    source  = "terraform-aws-modules/vpc/aws"
    version = "3.0.0"
    name    = "example-vpc"
    cidr    = "10.0.0.0/16"
    azs     = ["us-east-1a","us-east-1b"]
  }
  ```
* **Azure Example:** Use a network module for Azure.

  ```hcl
  module "azure_network" {
    source  = "Azure/network/azurerm"
    version = "2.0.0"
    name    = "mymodule"
    address_space = ["10.1.0.0/16"]
  }
  ```
* **GCP Example:** Call a GCP VPC module.

  ```hcl
  module "gcp_vpc" {
    source      = "terraform-google-modules/network/google"
    version     = "5.0.0"
    project_id  = "my-gcp-project-id"
    network_name = "example-network"
    routing_mode = "GLOBAL"
  }
  ```

You can also create your own modules by organizing `.tf` files in a directory (with `variables.tf`, `outputs.tf`, etc.). For example, a `module "webserver"` could encapsulate creating EC2 instances and load balancers. To pass data into a module, use input variables in the module; to use outputs from a module, refer to them as `module.NAME.output_name`.

**Tip:** Use modules for logical grouping (e.g. “network”, “compute”) and reuse. Publish shared modules to a Registry or Git repo. Pin module versions (`version = "x.y.z"`) just like providers. Keep modules simple and parameterized via inputs.

**Quiz:**

1. **What is a Terraform module?**
   *Answer:* A module is a container (directory) of Terraform configuration (resources, variables, outputs) used together. It can be reused across configurations. The root module is the main config; child modules are additional, reusable configs.
2. **How do you use a module from the Terraform Registry?**
   *Answer:* Use a `module` block with `source = "registry/name"` and specify `version`. For example: `module "vpc" { source = "terraform-aws-modules/vpc/aws", version = "3.0.0", ... }`.
3. **How do you pass a value into a module?**
   *Answer:* By defining an input variable in the module (using a `variable` block) and setting it in the module call. For instance, if the module has `variable "region"`, you can use `module "x" { source = "...", region = "us-west-1" }`.
4. **How do you get an output value from a module?**
   *Answer:* Define an `output` in the module, and then reference it as `module.NAME.output_name`. For example, `module.vpc.vpc_id` to get the `vpc_id` output from a module named `vpc`.
5. **Can a module call another module?**
   *Answer:* Yes. A module can call other (child) modules using `module` blocks inside its config. These called modules are nested under the parent.

## Variables and Outputs

Terraform configurations use **input variables** and **output values** to parameterize and expose data. Input variables allow you to customize modules without changing the code, and outputs let you expose information after deployment.

As the docs state, *“Input Variables serve as parameters for a Terraform module, so users can customize behavior without editing the source. Output Values are like return values for a Terraform module.”*.

* **AWS Example (Variables & Outputs):**

  ```hcl
  variable "instance_count" {
    description = "Number of web instances"
    type        = number
    default     = 2
  }

  resource "aws_instance" "web" {
    count         = var.instance_count
    ami           = "ami-0c55b159cbfafe1f0"
    instance_type = "t2.micro"
  }

  output "web_ips" {
    description = "Public IPs of web instances"
    value       = aws_instance.web[*].public_ip
  }
  ```
* **Azure Example:**

  ```hcl
  variable "location" {
    description = "Azure region for all resources"
    type        = string
    default     = "East US"
  }

  resource "azurerm_storage_account" "storage" {
    name                     = "examplestorageacct"
    resource_group_name      = "example-resources"
    location                 = var.location
    account_tier             = "Standard"
    account_replication_type = "LRS"
  }

  output "storage_id" {
    value = azurerm_storage_account.storage.id
  }
  ```
* **GCP Example:**

  ```hcl
  variable "region" {
    type    = string
    default = "us-central1"
  }

  resource "google_compute_network" "vpc" {
    name                    = "example-network"
    auto_create_subnetworks = true
  }

  output "network_self_link" {
    value = google_compute_network.vpc.self_link
  }
  ```

**Notes:** Variables are declared with `variable` blocks, can have `type`, `default`, and `description`. Users can override defaults via `terraform.tfvars` or `-var`. Outputs are declared with `output` blocks and have `value` expressions. Output values are shown after `terraform apply` and can be used by other modules (via `terraform_remote_state` or module references).

**Quiz:**

1. **What is an input variable in Terraform?**
   *Answer:* An input variable is a parameter for a module, declared with a `variable` block. It allows customization without editing code.
2. **How do you specify a default value for a variable?**
   *Answer:* In the `variable` block, include a `default = ...` argument. If not set in tfvars or CLI, Terraform uses the default.
3. **How do you reference a variable inside the config?**
   *Answer:* Use the syntax `var.NAME`, e.g., `var.instance_count`.
4. **What is an output value?**
   *Answer:* An output value is like a return value of a module. Declared with an `output` block, it exposes information (e.g. IPs or IDs) after apply.
5. **Can outputs from one configuration be used in another?**
   *Answer:* Yes. You can use the `terraform_remote_state` data source or module outputs to consume outputs of one config in another.

## State Management

Terraform uses a **state file** to keep track of real infrastructure and map it to your configuration. This state is *the source of truth* for Terraform’s view of resources. By default, state is stored locally in `terraform.tfstate`. However, local state complicates team workflows (everyone must get the latest state file).

With **remote state**, Terraform stores state in a shared location (e.g. Terraform Cloud, S3, Azure Blob, GCS, etc.), allowing collaboration and locks. As HashiCorp docs note: *“With remote state, Terraform writes the state data to a remote data store, which can then be shared between all members of a team. Terraform supports storing state in \[…] Amazon S3, Azure Blob Storage, Google Cloud Storage, \[etc.]”*. For example, you can configure an S3 backend with DynamoDB for locking, or use Terraform Cloud’s integrated state storage.

* **AWS Example (S3 Backend):**

  ```hcl
  terraform {
    backend "s3" {
      bucket         = "my-terraform-state-bucket"
      key            = "project/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "terraform-locks"
    }
  }
  ```
* **Azure Example (Blob Storage Backend):**

  ```hcl
  terraform {
    backend "azurerm" {
      storage_account_name = "tfstateaccount"
      container_name       = "tfstate"
      key                  = "prod.terraform.tfstate"
    }
  }
  ```
* **GCP Example (GCS Backend):**

  ```hcl
  terraform {
    backend "gcs" {
      bucket = "my-terraform-state-bucket"
      prefix = "project1"
    }
  }
  ```

**Best Practices:** Use a remote backend in team/project settings. Enable state locking (e.g. DynamoDB with S3) to prevent concurrent writes. Ensure state files are protected (encrypted). Avoid sensitive data in state by marking variables as sensitive or storing secrets elsewhere. Regularly back up state if stored remotely.

**Quiz:**

1. **Why does Terraform need a state file?**
   *Answer:* Terraform needs state to map resource blocks in your code to real-world objects (with IDs, attributes). The state file is the *source of truth* for what Terraform has created.
2. **Where is Terraform state stored by default?**
   *Answer:* By default, in a local file named `terraform.tfstate` in the working directory.
3. **What is a Terraform backend?**
   *Answer:* A backend determines where state is stored. For example, the `backend "s3"` stores state in an AWS S3 bucket. Backends can be local or remote (cloud storage, Terraform Cloud, etc.).
4. **Name two remote storage options for Terraform state.**
   *Answer:* Amazon S3 (often with DynamoDB locking), Azure Blob Storage, Google Cloud Storage, HashiCorp Terraform Cloud, or Consul are common options.
5. **How can teams avoid state conflicts when working together?**
   *Answer:* By using a remote backend with locking (e.g. S3 + DynamoDB, or Terraform Cloud), so that one Terraform run locks the state until completion.

## Workspaces

**Workspaces** (legacy Terraform CLI workspaces) allow multiple states for a single configuration. This means you can deploy the same config into multiple environments (e.g. **dev**, **staging**, **prod**) without duplicating code. In essence, you get separate state files per workspace.

Some backends (like Terraform Cloud or Consul) support multiple named workspaces. Terraform documentation explains: *“Some backends support multiple named workspaces, allowing multiple states to be associated with a single configuration... you can deploy multiple distinct instances of that configuration without configuring a new backend”*.

Example usage (Terraform CLI):

```bash
terraform workspace new dev
terraform workspace select dev
terraform apply
# Switch to prod workspace
terraform workspace new prod
terraform workspace select prod
terraform apply
```

You can refer to the workspace name in configs as `terraform.workspace` if needed, for example to create environment-specific resources.

**Note:** Terraform CLI workspaces are best for simple environment segregation. For larger teams, it’s often better to use separate state backends or full configurations for each environment. (Don’t confuse these with Terraform Cloud’s concept of workspaces, which are slightly different.)

**Quiz:**

1. **What is a Terraform workspace (CLI)?**
   *Answer:* A workspace is an isolated state for a given configuration. Each workspace has its own state, allowing multiple environments (e.g. dev and prod) with the same code.
2. **What is the default workspace called?**
   *Answer:* The default workspace is named `default`.
3. **How do you create or switch workspaces?**
   *Answer:* Using Terraform CLI commands: `terraform workspace new <name>` to create, and `terraform workspace select <name>` to switch.
4. **Can resources from one workspace conflict with another?**
   *Answer:* No, each workspace has its own state, so resources in different workspaces are managed independently.
5. **When should you *not* use workspaces?**
   *Answer:* Don’t use CLI workspaces to manage completely separate environments requiring different credentials or access controls. In those cases, it’s better to use separate configurations or workspaces in Terraform Cloud. The docs warn that workspaces are not a full substitute for environment separation.

## Provisioners and Meta-Arguments

### Provisioners

**Provisioners** allow Terraform to execute scripts on a resource after (or before) it is created/destroyed. They are a “last resort” feature for bootstrapping. From the docs: *“Provisioners are used to execute scripts on a local or remote machine as part of resource creation or destruction. Provisioners can be used to bootstrap a resource… cleanup before destroy, run configuration management, etc.”*. However, Terraform recommends avoiding them if possible (use cloud-init/user-data or configuration management tools instead).

Examples of built-in provisioners: `local-exec`, `remote-exec`, `file`.

* **AWS Example (local-exec provisioner):** Run a local script after creating an EC2 instance.

  ```hcl
  resource "aws_instance" "web" {
    ami           = "ami-0c55b159cbfafe1f0"
    instance_type = "t2.micro"

    provisioner "local-exec" {
      command = "echo The server's IP is ${self.public_ip}"
    }
  }
  ```
* **Azure Example (remote-exec provisioner):** Run a command on an Azure VM over SSH/WinRM.

  ```hcl
  resource "azurerm_virtual_machine" "vm" {
    # VM config omitted for brevity
    ...
    provisioner "remote-exec" {
      inline = ["echo Hello, Azure VM"]
      connection {
        type        = "SSH"
        host        = azurerm_public_ip.myvm.ip_address
        user        = "azureuser"
        private_key = file("~/.ssh/id_rsa")
      }
    }
  }
  ```
* **GCP Example (remote-exec provisioner):**

  ```hcl
  resource "google_compute_instance" "vm" {
    # VM config omitted
    ...
    provisioner "remote-exec" {
      inline = ["echo Hello, GCP VM"]
      connection {
        type        = "ssh"
        host        = google_compute_instance.vm.network_interface[0].network_ip
        user        = "gcpuser"
        private_key = file("~/.ssh/gcp_key")
      }
    }
  }
  ```

**Meta-Arguments:** These are special settings within resource (or module) blocks that adjust behavior. Common meta-arguments include:

* `count` (create multiple instances of a resource using one block).
* `for_each` (iterate over a map or set to create multiple resources).
* `depends_on` (explicitly set resource dependencies).
* `lifecycle` (customize create/destroy behavior).
* `provider` (to select a provider alias).

For example, to create 3 copies of an AWS instance:

```hcl
resource "aws_instance" "app" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

And using `for_each` with a map:

```hcl
resource "aws_security_group_rule" "rule" {
  for_each   = { ssh = 22, http = 80 }
  type       = "ingress"
  from_port  = each.value
  to_port    = each.value
  protocol   = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

**Quiz:**

1. **What is the purpose of a provisioner?**
   *Answer:* A provisioner runs scripts on a resource during creation or destruction (e.g. to configure a VM). It’s a bootstrapping tool but should be used sparingly.
2. **Name two built-in provisioner types.**
   *Answer:* `local-exec` (runs a local command) and `remote-exec` (runs on the remote machine via SSH/WinRM). Also `file` to copy files.
3. **What does the `count` meta-argument do?**
   *Answer:* `count` creates multiple instances of a resource from one block. For example `count = 3` would create three identical resources with indexes 0..2.
4. **How is `for_each` different from `count`?**
   *Answer:* `for_each` iterates over a map or set, allowing you to create one resource per key/value. It provides a map of values (`each.key` and `each.value`) inside the block.
5. **When would you use `depends_on`?**
   *Answer:* When you need to force a dependency between resources that Terraform doesn’t automatically infer. For example, if resource A must be created after resource B even if no config reference is present, use `depends_on = [aws_instance.a]`.

## Lifecycle Rules

The `lifecycle` meta-argument lets you customize resource creation and destruction behavior. According to HashiCorp, lifecycle arguments *“help control the flow of your Terraform operations by creating custom rules for resource creation and destruction”*. Common lifecycle settings include:

* `create_before_destroy = true`: Create replacement resource *before* destroying the old one (avoids downtime on replacement).
* `prevent_destroy = true`: Forbid destroying this resource (Terraform will error if a destroy is planned).
* `ignore_changes`: A list of attributes Terraform should ignore changes to (useful if some settings are managed outside of Terraform).
* `replace_triggered_by`: (v1.2+) Replace a resource when a referenced resource changes.

Example usage:

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [ "tags" ]
  }
}
```

In this block, Terraform will spin up the new instance before deleting the old one, and it will refuse to delete the instance entirely. It will also ignore any changes to tags (allowing an external process to update tags without Terraform trying to revert them).

**Quiz:**

1. **What does `create_before_destroy = true` do?**
   *Answer:* It tells Terraform to create the new replacement resource first, and only destroy the old one after the new is ready.
2. **What is the effect of `prevent_destroy = true`?**
   *Answer:* Terraform will refuse to destroy this resource if ever planned (to protect critical resources from accidental deletion).
3. **How does `ignore_changes` work?**
   *Answer:* It tells Terraform to ignore changes to the specified attributes on a resource when planning. Terraform will still create/destroy, but if the ignored attribute differs, Terraform won’t try to update it.
4. **Give an example when you might use lifecycle rules.**
   *Answer:* Use `create_before_destroy` for a resource that cannot be down (e.g. database) to avoid downtime. Use `prevent_destroy` on a production DB to avoid accidental deletion. Use `ignore_changes` if, for example, a separate process updates tags that you want Terraform to ignore.
5. **What happens if a resource with `prevent_destroy = true` is removed from code?**
   *Answer:* The `prevent_destroy` setting is removed along with it, so Terraform will then allow the resource to be destroyed (since the protection flag isn’t there in config anymore).

## Remote Backends

A **backend** in Terraform configures where state is stored and how operations are executed. A *remote backend* writes state to a remote store, enabling teamwork. As HashiCorp explains: *“With remote state, Terraform writes the state data to a remote data store, which can then be shared between all members of a team”*. Common remote backends include:

* **Terraform Cloud (HCP Terraform):** Hosted state management (no YAML config needed).
* **Consul:** State in HashiCorp Consul KV store.
* **S3 (with DynamoDB lock):** For AWS.
* **Azure Blob Storage:** For Azure.
* **Google Cloud Storage:** For GCP.

**Examples:** (These are placed in a `terraform { backend "..." { ... } }` block in the root module.)

```hcl
# AWS S3 backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "env:/${terraform.workspace}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
  }
}

# Azure Blob backend
terraform {
  backend "azurerm" {
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod/terraform.tfstate"
  }
}

# GCP GCS backend
terraform {
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "prod"
  }
}
```

When you configure a backend, run `terraform init` to initialize it. After that, state is stored remotely. You can often configure locking (e.g. DynamoDB with S3) and encryption. The `terraform_remote_state` data source can then retrieve outputs from remote state if needed.

**Best Practices:** Always use a remote backend (not local state) for shared/team environments. Enable locking to prevent simultaneous writes. For Terraform Cloud, you simply select remote execution and state storage in the workspace. Avoid hard-coding backend credentials; use environment variables or workspace credentials.

**Quiz:**

1. **What is a Terraform backend?**
   *Answer:* A backend defines where Terraform's state is stored and how operations are executed. For example, `backend "s3"` stores state in AWS S3.
2. **Name three remote backends supported by Terraform.**
   *Answer:* Terraform Cloud (HCP Terraform), AWS S3 (with DynamoDB), Azure Blob Storage, Google Cloud Storage, and HashiCorp Consul are examples.
3. **How do you switch an existing config to use a remote backend?**
   *Answer:* Add a `terraform { backend "TYPE" { ... } }` block to your config and run `terraform init`. Terraform will prompt to migrate state to the new backend.
4. **Why use remote state in a team environment?**
   *Answer:* To share a single source of truth for state, prevent state drift, and allow locking. With local state, each user could be out of sync.
5. **Can Terraform Cloud be used as a backend?**
   *Answer:* Yes. Terraform Cloud provides a remote execution and state backend. You configure it via the workspace settings or using the `backend "remote"` block.

## Terraform Cloud and Enterprise

Terraform Cloud (now part of HashiCorp’s Infrastructure Cloud, formerly HCP) and Terraform Enterprise are SaaS and self-hosted offerings that add collaboration features on top of Terraform. They provide remote runs, secure remote state, VCS integration, private module registry, policy as code (Sentinel), team management, and more. Terraform Enterprise is essentially a self-hosted Terraform Cloud with enterprise security features (audit logs, SSO).

Key points:

* **Remote Runs:** Code is version-controlled (e.g. GitHub) and Terraform Cloud runs `plan` and `apply` automatically on its servers.
* **State Management:** Uses remote state with locking, no need for separate S3 or other backend.
* **VCS Integration:** Connect a Git repo; workspace triggers on pull request merges.
* **Modules & Registry:** You can share and version modules in a private registry.
* **Policy (Sentinel):** (Cloud/Enterprise only) Enforce rules on runs (like requiring tags, limiting instance types).
* **Team Collaboration:** Access controls, workspace permissions, notifications.
* **Enterprise Features:** SAML SSO, audit logging, private networking, performance seats, etc.

Example (Terraform v0.14+ remote backend usage):

```hcl
terraform {
  backend "remote" {
    hostname     = "app.terraform.io"
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}
```

This tells Terraform CLI to use Terraform Cloud (App.terraform.io) for state and runs.

**Tip:** For the exam, know that Terraform Cloud/Enterprise gives remote execution and state, not available in OSS CLI alone. Practice using `terraform login` and remote backend configuration. Understand the difference between **Terraform CLI/OSS** (runs locally) vs **Terraform Cloud (remote runs)**.

**Quiz:**

1. **What is Terraform Enterprise?**
   *Answer:* A self-hosted distribution of Terraform Cloud (HCP Terraform) with enterprise features like SAML SSO and audit logs.
2. **Name a feature exclusive to Terraform Cloud/Enterprise (not plain CLI).**
   *Answer:* Policy as Code (Sentinel) or private module registry. Also remote run orchestration with VCS integration.
3. **How do you configure Terraform to use Terraform Cloud as a backend?**
   *Answer:* Use a `backend "remote"` block (as shown above) or the newer Cloud UI integration. This sets `hostname`, `organization`, and workspace.
4. **Can Terraform OSS (CLI) use remote state in the cloud?**
   *Answer:* Yes, by configuring the remote backend (Terraform Cloud) as above or using other backends (S3/Azure/GCS). Plain CLI doesn’t have built-in state sharing.
5. **What is a Terraform Cloud workspace (in contrast to CLI workspaces)?**
   *Answer:* A Terraform Cloud workspace is a concept in Terraform Cloud that holds state and settings for one configuration (often mapping to a VCS repo or folder). It is distinct from the CLI’s workspace concept.

## Importing Resources

Terraform can **import** existing cloud resources into your state, so Terraform can manage them. The `terraform import` command links a real-world resource to a resource block in your configuration. Key points:

* You must **write the resource block first** (with the same type and name) in code, then run import. Terraform will *not* generate configuration for you.
* The command syntax is `terraform import <address> <id>`, where `<address>` is `resource_type.name` and `<id>` is the cloud resource ID.
* Importing only updates the state; it does not create the resource. After import, Terraform will manage the resource.

Example imports:

```bash
terraform import aws_instance.web i-0abc123def4567890
terraform import azurerm_resource_group.rg /subscriptions/0000-1111-2222-3333/resourceGroups/myResourceGroup
terraform import google_compute_instance.vm my-project/us-central1-a/example-vm
```

After running import, you should run `terraform plan` to verify that your code matches the existing resource and make any configuration adjustments.

**Quiz:**

1. **What does `terraform import` do?**
   *Answer:* It attaches an existing infrastructure object (like an AWS instance) to a resource in your Terraform state, so Terraform can manage it.
2. **Does `terraform import` create Terraform code for you?**
   *Answer:* No. Import only adds the resource to state. You must manually have a matching `resource` block in your code before importing.
3. **What is the general syntax of the `terraform import` command?**
   *Answer:* `terraform import [options] ADDRESS ID`. Example: `terraform import aws_instance.web i-1234567890abcdef0`.
4. **Can you import multiple resources with one command?**
   *Answer:* The CLI `import` command imports one resource at a time. (Terraform v0.14+ introduced an `import` block in config for multiple imports.)
5. **After importing a resource, what should you do?**
   *Answer:* Run `terraform plan` to ensure the config matches the imported resource. Adjust the config or state if needed, then apply to bring Terraform and the real world in sync.

## Debugging and Logging

When troubleshooting Terraform, detailed logs can help. Terraform supports environment variables to control logging verbosity:

* **TF\_LOG**: Set `TF_LOG` to one of `TRACE`, `DEBUG`, `INFO`, `WARN`, or `ERROR` to enable logging at that level. For example:

  ```bash
  export TF_LOG=DEBUG
  terraform apply
  ```

  This will print detailed logs to stderr.
* **TF\_LOG\_PATH**: You can set `TF_LOG_PATH` to have Terraform write logs to a file. For example:

  ```bash
  export TF_LOG_PATH=terraform.log
  ```
* **TF\_LOG\_CORE / TF\_LOG\_PROVIDER**: Control logging for Terraform core vs providers. E.g. `TF_LOG_PROVIDER=DEBUG` logs only provider plugins, `TF_LOG_CORE` for Terraform core.
* **Terraform Console and Output**: Use `terraform console` to evaluate expressions, or add `output` statements with sensitive values (be careful not to log secrets).

Setting `TF_LOG` to `TRACE` or `DEBUG` gives a very verbose output (with sensitive info!), mainly for debugging. Remember to unset or lower the log level afterward. There is no cost to enabling it except performance and risk of leaking secrets in logs.

**Quiz:**

1. **How do you enable detailed Terraform debugging logs?**
   *Answer:* Set the environment variable `TF_LOG` to `DEBUG` (or `TRACE`) before running Terraform. For example: `export TF_LOG=DEBUG`.
2. **Where do Terraform logs go when `TF_LOG` is set?**
   *Answer:* They are printed to stderr by default. If `TF_LOG_PATH` is set, logs are written (appended) to that file.
3. **What is the difference between `TF_LOG_CORE` and `TF_LOG_PROVIDER`?**
   *Answer:* They filter logging to core Terraform or provider plugins respectively. E.g. `TF_LOG_PROVIDER=DEBUG` will only log the provider’s internal operations.
4. **Why should you be careful with TF\_LOG?**
   *Answer:* Because debug logs may contain sensitive data. Don’t leave high verbosity logs enabled in production and avoid logging secrets.
5. **How can you debug a Terraform expression or data source?**
   *Answer:* Use `terraform console` to interactively evaluate expressions, or use `output` to print values. For CLI errors, use `TF_LOG=DEBUG` for more context.

## Tips and Best Practices

* **Use Version Control:** Put all Terraform `.tf` files in a Git repository. Review changes with pull requests and commit history.
* **Keep Code DRY:** Use modules and variables to avoid duplication. Reuse modules for common patterns.
* **Secure State:** Never commit `terraform.tfstate` to VCS. Use remote state backends with encryption and locking.
* **Pin Versions:** Pin your Terraform version (`required_version`) and provider versions (`required_providers`) to avoid surprises.
* **Automate Checks:** Run `terraform fmt` and `terraform validate` to catch syntax issues. Review plans before applying (`terraform plan`).
* **Separation of Environments:** For dev/staging/prod, use either separate backends or Terraform Cloud workspaces, rather than just CLI workspaces for production workloads.
* **Least Privilege:** Use separate cloud credentials for Terraform with only the necessary permissions. Don’t run Terraform as root/admin.
* **Refresh State Appropriately:** Use `terraform refresh` or targeted refresh if state is out-of-sync. (Most tasks auto-refresh on plan.)
* **Stay Current:** Read HashiCorp’s release notes, as Terraform adds exam-relevant features in new versions (for example, `replace_triggered_by` was added in v1.2).
* **Practice in a Sandbox:** Before the exam, practice creating, modifying, and destroying resources in AWS, Azure, or GCP.  Familiarize yourself with common CLI commands (`init`, `plan`, `apply`, `destroy`, `import`, etc.).

**Final Notes:** This course has covered the core topics for the HashiCorp Certified Terraform Associate exam, including all required concepts and practical code examples. Use the quiz questions to test your understanding. Practice hands-on with Terraform and refer to the official documentation for any details. Good luck with your exam preparation!
