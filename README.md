The generate.yml just runs the role that generates the terraform to create
the terraform files.  

Basically followed the blog: https://spacelift.io/blog/terraform-aws-vpc 

The variables are in the role itself under the roles/genterraform/vars/main.yml. This is
what the directory structure looks like:

```shell

roles/                                                                                                                                                                                                                  
└── genterraform                                                                                                                                                                                                        
    ├── defaults                                                                                                                                                                                                        
    │   └── main.yml                                                                                                                                                                                                    
    ├── files                                                                                                                                                                                                           
    ├── handlers                                                                                                                                                                                                        
    │   └── main.yml                                                                                                                                                                                                    
    ├── meta                                                                                                                                                                                                            
    │   └── main.yml                                                                                                                                                                                                    
    ├── README.md                                                                                                                                                                                                       
    ├── tasks                                                                                                                                                                                                           
    │   └── main.yml                                                                                                                                                                                                    
    ├── templates                                                                                                                                                                                                       
    │   ├── provider.tf.j2                                                                                                                                                                                              
    │   ├── subnet.tf.j2                                                                                                                                                                                                
    │   └── vpc.tf.j2                                                                                                                                                                                                   
    ├── tests                                                                                                                                                                                                           
    │   ├── inventory                                                                                                                                                                                                   
    │   └── test.yml                                                                                                                                                                                                    
    └── vars                                                                                                                                                                                                            
        └── main.yml

```

## VARS file format under main.yml (Ansible)
The vars file has the following format:
```yaml

---
# vars file for genterraform

business_units:
  - name: bu1
    provider: 
      cloud: aws 
      region: us-west-2
    vpc:
      name: bu1_vpc
      cidr: 10.0.0.0/16
      tag_name: "BU1 VPC on AWS"
      subnets:
        - name: bu1_public_subnet
          description: "Public Subnet CIDRS for BU1"
          cidrs: ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
          availability_zones: [ "us-east-1", "us-west-1", "us-west-2"]
        - name: bu1_private_subnet
          description: "Private Subnet CIDRS for BU1"
          cidrs: ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"] 
          availability_zones: [ "us-east-1", "us-west-1", "us-west-2"]

```
## Description of variables
Under the business_units section you can have a list of business units.  Each one will have the following variables defined:

```yaml

- name: Name of the BU
  provider: Section that describes the cloud provider.
    name: Name of the providers. aws, azure etc.
    region: Region used to create the resources
  vpc: Section that describes the VPC resource
    name: Name of the VPC
    cidr: Cidr for the VPC
    tag_name: Tag name for the VPC
    subnets: Section used to defines the list of subnets
      - name: Name of the subnet
        description: Description of the subnet resource. Used to create a TAG in AWS 
        cidrs: List of CIDRS for the subnet
        availability_zones: List of availability zones for the subnet

```

## Running the playbook

```shell

ansible-playbook ./generate.yml
claudiol@claudiol-thinkpadp16vgen1:gen-terraform
$ ansible-playbook ./generate.yml 

PLAY [localhost] *******************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************************************
ok: [claudiol-thinkpadp16vgen1.rmtusco.csb]

TASK [genterraform : Generate Terraform files] *************************************************************************************************************************************************************************
ok: [claudiol-thinkpadp16vgen1.rmtusco.csb] => (item=provider.tf.j2)
ok: [claudiol-thinkpadp16vgen1.rmtusco.csb] => (item=vpc.tf.j2)
ok: [claudiol-thinkpadp16vgen1.rmtusco.csb] => (item=subnet.tf.j2)

PLAY RECAP *************************************************************************************************************************************************************************************************************
claudiol-thinkpadp16vgen1.rmtusco.csb : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

## What gets generated
roles/genterraform/v
The following files get generated using the Jinja2 templates under roles/genterraform/templates directory.

```shell

claudiol@claudiol-thinkpadp16vgen1:gen-terraform
$ tree /tmp/bu
/tmp/bu
├── provider.tf
├── subnet.tf
└── vpc.tf

```

### vpc.tf file

```
resource "aws_vpc" "bu1_vpc" {
 cidr_block = "10.0.0.0/16"
 
 tags = {
   generated: "VPC Terraform resource bu1_vpc generated by Ansible"
   Name = "bu1_vpc"
 }
}

```

### provider.tf

```

provider "aws" {
  region = "us-west-2"
  tags = {
    generated: "VPC Terraform resource bu1_vpc generated by Ansible"
    Name = "bu1_vpc"
  }
}

```


### subnet.tf file

```

resource "aws_subnet" "bu1_public_subnet" {
  count = length( 3 )
  vpc_id     = "bu1_vpc.id"
  cidr_block = element( ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"], count.index )
  availability_zone = element( ["us-east-1", "us-west-1", "us-west-2"], count.index )

  tags = {
    generated: "VPC Terraform resource bu1_public_subnet generated by Ansible"
    Name = "Public Subnet CIDRS for BU1 ${count.index + 1}"
  }
}
resource "aws_subnet" "bu1_private_subnet" {
  count = length( 3 )
  vpc_id     = "bu1_vpc.id"
  cidr_block = element( ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"], count.index )
  availability_zone = element( ["us-east-1", "us-west-1", "us-west-2"], count.index )

  tags = {
    generated: "VPC Terraform resource bu1_private_subnet generated by Ansible"
    Name = "Private Subnet CIDRS for BU1 ${count.index + 1}"
  }
} 

```

## Running terraform to initialize

```shell

claudiol@claudiol-thinkpadp16vgen1:bu
$ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v5.76.0...
- Installed hashicorp/aws v5.76.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

```

## Run terraform plan
You then run terraform plan to see what resources this will create

```shell

claudiol@claudiol-thinkpadp16vgen1:bu
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_subnet.bu1_private_subnet[0] will be created
  + resource "aws_subnet" "bu1_private_subnet" {
      + arn                                            = (known after apply)
      + assign_ipv6_address_on_creation                = false
      + availability_zone                              = "us-east-1"
      + availability_zone_id                           = (known after apply)
      + cidr_block                                     = "10.0.4.0/24"
      + enable_dns64                                   = false
      + enable_resource_name_dns_a_record_on_launch    = false
      + enable_resource_name_dns_aaaa_record_on_launch = false
      + id                                             = (known after apply)
      + ipv6_cidr_block_association_id                 = (known after apply)
      + ipv6_native                                    = false
      + map_public_ip_on_launch                        = false
      + owner_id                                       = (known after apply)
      + private_dns_hostname_type_on_launch            = (known after apply)
      + tags                                           = {
          + "Name"      = "Private Subnet CIDRS for BU1 1"
          + "generated" = "VPC Terraform resource bu1_private_subnet generated by Ansible"
        }
      + tags_all                                       = {
          + "Name"      = "Private Subnet CIDRS for BU1 1"
          + "generated" = "VPC Terraform resource bu1_private_subnet generated by Ansible"
        }
      + vpc_id                                         = "bu1_vpc.id"
    }

  # aws_subnet.bu1_public_subnet[0] will be created
  + resource "aws_subnet" "bu1_public_subnet" {
      + arn                                            = (known after apply)
      + assign_ipv6_address_on_creation                = false
      + availability_zone                              = "us-east-1"
      + availability_zone_id                           = (known after apply)
      + cidr_block                                     = "10.0.1.0/24"
      + enable_dns64                                   = false
      + enable_resource_name_dns_a_record_on_launch    = false
      + enable_resource_name_dns_aaaa_record_on_launch = false
      + id                                             = (known after apply)
      + ipv6_cidr_block_association_id                 = (known after apply)
      + ipv6_native                                    = false
      + map_public_ip_on_launch                        = false
      + owner_id                                       = (known after apply)
      + private_dns_hostname_type_on_launch            = (known after apply)
      + tags                                           = {
          + "Name"      = "Public Subnet CIDRS for BU1 1"
          + "generated" = "VPC Terraform resource bu1_public_subnet generated by Ansible"
        }
      + tags_all                                       = {
          + "Name"      = "Public Subnet CIDRS for BU1 1"
          + "generated" = "VPC Terraform resource bu1_public_subnet generated by Ansible"
        }
      + vpc_id                                         = "bu1_vpc.id"
    }

  # aws_vpc.bu1_vpc will be created
  + resource "aws_vpc" "bu1_vpc" {
      + arn                                  = (known after apply)
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = (known after apply)
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags                                 = {
          + "Name"      = "bu1_vpc"
          + "generated" = "VPC Terraform resource bu1_vpc generated by Ansible"
        }
      + tags_all                             = {
          + "Name"      = "bu1_vpc"
          + "generated" = "VPC Terraform resource bu1_vpc generated by Ansible"
        }
    }

Plan: 3 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

Hopefully this gives you an idea of the power of code generation.  
