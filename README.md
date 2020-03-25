# Jenkins X EKS Module

This repository contains a Terraform module for creating an EKS cluster and all the necessary infrastructure to install Jenkins X via `jx boot`.
The module generates for this purpose a  templated `jx-requirements.yml` which can be passed to `jx boot`.

The module makes use of the [Terraform EKS cluster Module](https://github.com/terraform-aws-modules/terraform-aws-eks).

## Assumptions

You want to create an EKS cluster for installation of Jenkins X.

Both `kubectl` (>=1.10) and `aws-iam-authenticator` are installed and on your shell's PATH.

## Usage example
This module works with a series of variables with default values, this will let you easily run it with a default configuration for easy prototyping by just providing the following required variables:

```sh
terraform init
terraform apply -var 'cluster_name=<cluster_name>' -var 'region=<your_aws_region>' -var 'account_id=<your_aws_account_id>'
```

Full customization of the EKS and Kubernetes modules through the use of this module is still not supported as this is still work in progress.

Several variables allow you to configure the resources for your EKS cluster.

### VPC configuration

The following variables allow you to configure the settings of the generated VPC:

```
    variable "vpc_name" {
      description  = "The name of the VPC to be created for the cluster"
      type         = string
      default      = "tf-vpc-eks"
    }

    variable "vpc_subnets" {
      description = "The subnet CIDR block to use in the created VPC"
      type        = list(string)
      default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
    }

    variable "vpc_cidr_block" {
      description = "The vpc CIDR block"
      type        = string
      default     = "10.0.0.0/16"
    }
```

### EKS Worker Nodes configuration

You can configure the EKS worker node pool with the following variables:

```
    variable "desired_number_of_nodes" {
      description = "The desired number of worker nodes to use for the cluster. Defaults to 3"
      type        = number
      default     = 3
    }

    variable "min_number_of_nodes" {
      description = "The minimum number of worker nodes to use for the cluster. Defaults to 3"
      type        = number
      default     = 3
    }

    variable "max_number_of_nodes" {
      description = "The maximum number of worker nodes to use for the cluster. Defaults to 5"
      type        = number
      default     = 5
    }

    variable "worker_nodes_instance_types" {
      description  = "The instance type to use for the cluster's worker nodes. Defaults to m5.large"
      type         = string
      default      = "m5.large"
    }
```

### Long Term Storage

You can choose whether to create S3 buckets for long term storage and enable them in the generated `jx-requirements.yml` file.

```
    variable "enable_logs_storage" {
      description = "Flag to enable or disable long term storage for logs"
      type        = bool
      default     = true
    }

    variable "enable_reports_storage" {
      description = "Flag to enable or disable long term storage for reports"
      type        = bool
      default     = true
    }

    variable "enable_repository_storage" {
      description = "Flag to enable or disable the repository bucket storage"
      type        = bool
      default     = true
    }
```

If these variables are `true`, after creating the necessary S3 buckets, it will configure the `jx-requirements.yml` file in the following section:

```yaml
    storage:
      logs:
        enabled: ${enable_logs_storage}
        url: s3://${logs_storage_bucket}
      reports:
        enabled: ${enable_reports_storage}
        url: s3://${reports_storage_bucket}
      repository:
        enabled: ${enable_repository_storage}
        url: s3://${repository_storage_bucket}
```

### Vault configuration

You can choose to create the Vault resources needed by Jenkins X by setting the following variables:

```
    variable "create_vault_resources" {
      description = "Flag to enable or disable the creation of Vault resources by Terraform"
      type        = bool
      default     = false
    }
```

If `create_vault_resources` is `true`, the `vault_user` variable will be required:

```
    variable "vault_user" {
      type    = string
      default = ""
    }
```

### External DNS and Cert Manager

#### External DNS

You can enable External DNS with the following variable:

```
    variable "enable_external_dns" {
      description = "Flag to enable or disable External DNS in the final `jx-requirements.yml` file"
      type        = bool
      default     = false
    }
```

If `enable_external_dns` is true, additional configuration will be required:

If you want to use a domain with an already existing Route 53 Hosted Zone, you can provide it through the following variable:

```
    variable "apex_domain" {
      description = "Flag to enable or disable long term storage for logs"
      type        = string
      default     = ""
    }
```    

This domain will be configured in the resulting `jx-requirements.yml` file in the following section:

```
    ingress:
      domain: ${domain}
      ignoreLoadBalancer: true
      externalDNS: ${enable_external_dns}
```

If you want to use a subdomain and have this script create and configure a new Hosted Zone with DNS delegation, you can provide the following variables:

```
    variable "subdomain" {
      description = "The subdomain to be used added to the apex domain. If subdomain is set, it will be appended to the apex domain in  `jx-requirements-eks.yml` file"
      type        = string
      default     = ""
    }

    variable "create_and_configure_subdomain" {
      description = "Flag to create an NS record ser for the subdomain in the apex domain's Hosted Zone"
      type        = bool
      default     = false
    }
```

By providing these variables, the script creates a new `Route 53` HostedZone that looks like `<subdomain>.<apex_domain>`, and it delegates the resolving of DNS to the apex domain.
This is done by creating a `NS` RecordSet in the apex domain's Hosted Zone with the subdomain's HostedZone nameservers.

This will make sure that the newly created HostedZone for the subdomain is instantly resolvable instead of having to wait for DNS propagation.

#### Cert Manager

You can enable Cert Manager to use TLS for your cluster through LetsEncrypt with the following variables:

```
    variable "enable_tls" {
      description = "Flag to enable TLS int he final `jx-requirements.yml` file"
      type        = bool
      default     = false
    }
```

LetsEncrypt has two environments, `staging` and `production`.
 If you use staging, you receive self-signed certificates, but you are not rate limited.
If you use the `production` environment, you receive certificates signed by LetsEncrypt, but you can be rate limited.

You can choose to use the `production` environment with the following variable:

```
    variable "production_letsencrypt" {
      description = "Flag to use the production environment of letsencrypt in the `jx-requirements.yml` file"
      type        = bool
      default     = false
    }
```

You need to provide a valid email to register your domain in LetsEncrypt:

```
    variable "tls_email" {
      description = "The email to register the LetsEncrypt certificate with. Added to the `jx-requirements.yml` file"
      type        = string
      default     = ""
    }
```

## Generation of jx-requirements.yml

The final output of running this module will not only be the creation of cloud resources but also the creation of a valid `jx-requirements.yml` file.
You can use this file to install Jenkins X by running:

```bash
 jx boot -r jx-requirements.yml
```

The template can be found [here](https://github.com/jenkins-x/terraform-aws-eks-jx/blob/master/jx/jx-requirements.yml.tpl)

## Conditional creation

Sometimes you need to have a way to create resources conditionally; however, Terraform does not allow to use `count` inside a module block.
There still isn't a solution for this in this repository, but we will be working to allow users to provide their own VPC, subnets etc.

## FAQ: Frequently Asked Questions

### IAM Roles for Service Accounts
This module sets up a series of IAM Policies and Roles. These roles will be annotated into a few Kubernetes Service accounts.

This allows us to make use of IAM Roles for Sercive Accounts to set fine-grained permissions on a pod per pod basis.

There still isn't a way to provide your roles or define other Service Accounts by variables, but you can always modify the `eks/terraform/jx/irsa.tf` Terraform file.

## Generated Documentation

The following tables are generated with `terraform-docs`:

### Providers

| Name | Version |
|------|---------|
| local | ~> 1.2 |

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:-----:|
| account\_id | n/a | `string` | n/a | yes |
| apex\_domain | Flag to enable or disable long term storage for logs | `string` | `""` | no |
| cluster\_name | n/a | `string` | n/a | yes |
| create\_and\_configure\_subdomain | Flag to create an NS record ser for the subdomain in the apex domain's Hosted Zone | `bool` | `false` | no |
| create\_vault\_resources | Flag to enable or disable the creation of Vault resources by Terraform | `bool` | `false` | no |
| desired\_number\_of\_nodes | The number of worker nodes to use for the cluster. Defaults to 3 | `number` | `3` | no |
| enable\_external\_dns | Flag to enable or disable External DNS in the final `jx-requirements.yml` file | `bool` | `false` | no |
| enable\_logs\_storage | Flag to enable or disable long term storage for logs | `bool` | `true` | no |
| enable\_reports\_storage | Flag to enable or disable long term storage for reports | `bool` | `true` | no |
| enable\_repository\_storage | Flag to enable or disable the repository bucket storage | `bool` | `true` | no |
| enable\_tls | Flag to enable TLS int he final `jx-requirements.yml` file | `bool` | `false` | no |
| manage\_aws\_auth | Whether to apply the aws-auth configmap file. | `bool` | `true` | no |
| max\_number\_of\_nodes | The maximum number of worker nodes to use for the cluster. Defaults to 5 | `number` | `5` | no |
| min\_number\_of\_nodes | The minimum number of worker nodes to use for the cluster. Defaults to 3 | `number` | `3` | no |
| production\_letsencrypt | Flag to use the production environment of letsencrypt in the `jx-requirements.yml` file | `bool` | `false` | no |
| region | n/a | `string` | `"us-east-1"` | no |
| subdomain | The subdomain to be used added to the apex domain. If subdomain is set, it will be appended to the apex domain in  `jx-requirements-eks.yml` file | `string` | `""` | no |
| tls\_email | The email to register the LetsEncrypt certificate with. Added to the `jx-requirements.yml` file | `string` | `""` | no |
| vault\_user | n/a | `string` | `""` | no |
| vpc\_cidr\_block | The vpc CIDR block | `string` | `"10.0.0.0/16"` | no |
| vpc\_name | The name of the VPC to be created for the cluster | `string` | `"tf-vpc-eks"` | no |
| vpc\_subnets | The subnet CIDR block to use in the created VPC | `list(string)` | <pre>[<br>  "10.0.1.0/24",<br>  "10.0.2.0/24",<br>  "10.0.3.0/24"<br>]</pre> | no |
| wait\_for\_cluster\_cmd | Custom local-exec command to execute for determining if the eks cluster is healthy. Cluster endpoint will be available as an environment variable called ENDPOINT | `string` | `"until curl -k -s $ENDPOINT/healthz \u003e/dev/null; do sleep 4; done"` | no |
| worker\_nodes\_instance\_types | The instance type to use for the cluster's worker nodes. Defaults to m5.large | `string` | `"m5.large"` | no |

### Outputs

| Name | Description |
|------|-------------|
| cert\_manager\_iam\_role | The IAM Role that the Cert Manager pod will assume to authenticate |
| cluster\_name | The name of the created cluster |
| cm\_cainjector\_iam\_role | The IAM Role that the CM CA Injector pod will assume to authenticate |
| controllerbuild\_iam\_role | The IAM Role that the ControllerBuild pod will assume to authenticate |
| external\_dns\_iam\_role | The IAM Role that the External DNS pod will assume to authenticate |
| jxui\_iam\_role | The IAM Role that the Jenkins X UI pod will assume to authenticate |
| lts\_logs\_bucket | The bucket where logs from builds will be stored |
| lts\_reports\_bucket | The bucket where test reports will be stored |
| lts\_repository\_bucket | The bucket that will serve as artifacts repository |
| tekton\_bot\_iam\_role | The IAM Role that the build pods will assume to authenticate |
| vault\_dynamodb\_table | The bucket that Vault will use as backend |
| vault\_kms\_unseal | The KMS Key that Vault will use for encryption |
| vault\_unseal\_bucket | The bucket that Vault will use for storage |
