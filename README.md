# wordpresshosting
hosted wordpress on aws
## What is a virtual private cloud (VPC)?

<img src = screenshots/0.jfif>
 <img src = screenshots/1.jfif>

### A virtual private cloud (VPC) is a public cloud offering that lets an enterprise establish its own private cloud-like computing environment on shared public cloud infrastructure. A VPC gives an enterprise the ability to define and control a virtual network that is logically isolated from all other public cloud tenants, creating a private, secure place on the public cloud.

### Imagine that a cloud provider’s infrastructure is a residential apartment building with multiple families living inside. Being a public cloud tenant is akin to sharing an apartment with a few roommates. In contrast, having a VPC is like having your own private condominium—no one else has the key, and no one can enter the space without your permission.A VPC’s logical isolation is implemented using virtual network functions and security features that give an enterprise customer granular control over which IP addresses or applications can access particular resources. It is analogous to the “friends-only” or “public/private” controls on social media accounts used to restrict who can or can’t see your otherwise public posts.

## The following are the key concepts for VPCs:

Virtual private cloud (VPC) — A virtual network dedicated to your AWS account.
Subnet — A range of IP addresses in your VPC.
Route table — A set of rules, called routes, that are used to determine where network traffic is directed.
Internet gateway — A gateway that you attach to your VPC to enable communication between resources in your VPC and the internet.
VPC endpoint — Enables you to privately connect your VPC to supported AWS services and VPC endpoint services powered by PrivateLink without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection. Instances in your VPC do not require public IP addresses to communicate with resources in the service. Traffic between your VPC and the other service does not leave the Amazon network.

## TASK DESCRIPTION:
Statement: We have to create a web portal for our company with all the security as much as possible.
So, we use WordPress software with dedicated database server.
Database should not be accessible from the outside world for security purposes.
We only need to public the WordPress to clients.
So here are the steps for proper understanding!

Steps:

## 1) Write a Infrastructure as code using terraform, which automatically create a VPC.

## 2) In that VPC we have to create 2 subnets:

### a) public subnet [ Accessible for Public World! ]

### b) private subnet [ Restricted for Public World! ]

## 3) Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.

## 4) Create a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.

## 5) Launch an ec2 instance which has WordPress setup already having the security group allowing port 80 so that our client can connect to our wordpress site.

Also attach the key to instance for further login into it.

## 6) Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our wordpress vm can connect with the same.

## STEP-BY-STEP GUIDE:

Basically, to do all the steps mentioned above -I will write a teraform code. Using this terraform code, the whole setup will be created automatically since terraform is an open-source automation tool which can be easily used for infrastructure deployment in AWS.

## 1.Configure aws

        provider "aws" {
            region ="ap-south-1"
            profile = "lekhika"

        }
## 2.Create VPC

        resource "aws_vpc" "myvpc1" {
          cidr_block       = "192.168.0.0/16"
          instance_tenancy = "default"

          tags = {
            Name = "myvpc"
          }
        }
## 3.In this VPC, create one public subnet and one private subnet

        resource "aws_subnet" "public-subnet" {
          vpc_id     = "${aws_vpc.my-vpc.id}"
          cidr_block = "192.168.1.0/24"
          map_public_ip_on_launch = "true"
          availability_zone = "ap-south-1b"
          tags = {
            Name = "lekhikasub1"  
          }
        }
        resource "aws_subnet" "private-subnet" {
          vpc_id     = "${aws_vpc.my-vpc.id}"
          cidr_block = "192.168.0.0/24"
          availability_zone = "ap-south-1a"
          tags = {
            Name = "lekhikasub2"
          }
        }
## 4.Create a public facing internet gateway

    resource "aws_internet_gateway" "internet-gateway" {
      vpc_id = "${aws_vpc.my-vpc.id}"
      tags = {
        Name = "lekhikagw"
      }  
    }
## 5.Create a route table for Internet Gateway

    resource "aws_route_table" "r" {
      vpc_id = aws_vpc.myvpc1.id

       tags = {
        Name = "route_table"
      }
    }

    resource "aws_route_table_association" "a" {
      subnet_id      = aws_subnet.subnet1.id
      route_table_id = aws_route_table.r.id
    }
## 6. Add some Route so that everyone can connect to instance using Internet Gateway

    resource "aws_route" "b" {
      route_table_id = aws_route_table.r.id
      destination_cidr_block ="0.0.0.0/0"
      gateway_id     = aws_internet_gateway.gw.id
    }
## 7. Launch an ec2 instance which has WordPress setup already having the security group allowing port 80 so that our client can connect to our WordPress site

    //Creating key and security group

    resource "tls_private_key" "mykey"{
     algorithm = "RSA"
    }

    module "key_pair"{
     source ="terraform-aws-modules/key-pair/aws"

     key_name = "new_key"
     public_key = tls_private_key.mykey.public_key_openssh
    }

    resource "aws_security_group" "new_sg" {
      name        = "allow_tls"
      description = "Allow TLS inbound traffic"
      vpc_id      = aws_vpc.myvpc1.id

      ingress {
        description = "ssh"
        from_port   = 0
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

      ingress {
        description = "http"
        from_port   = 0
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }

      tags = {
        Name ="sgforWordPress"
      }
    }

    //Launching Instance
    resource "aws_instance" "myweb" {
      ami           = "ami-004a955bfb611bf13"
      instance_type = "t2.micro"
      subnet_id = "${aws_subnet.public-subnet.id}"
      vpc_security_group_ids = ["${aws_security_group.lekhikaSG1.id}"]
      key_name = "lekhika"
      tags = {
        Name = "lekhikaWebserver"
      }

    }
## 8. Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our WordPress vm can connect with the same.

    //Security Group

    resource "aws_security_group" "new_sg2" {
      name        = "sg_mysql"
      description = "Allow MYSQL"
      vpc_id      = aws_vpc.myvpc1.id


      ingress {
        description = "SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["${aws_subnet.public-subnet.cidr_block}"]
      }
      ingress {
        description = "TLS from VPC"
        from_port   = 3306
        to_port     = 3306
        protocol    = "tcp"
        cidr_blocks = ["${aws_subnet.public-subnet.cidr_block}"]
      }
      ingress {
        description = "ICMP - IPv4"
        from_port = -1
        to_port	= -1
        protocol	= "icmp"
        cidr_blocks = ["${aws_subnet.public-subnet.cidr_block}"]
      }
      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
      tags = {
        Name = "mysqlSG"
      } 
    }

    //Launching Instance
    resource "aws_instance" "mysql" {
      ami           = "ami-08706cb5f68222d09"
      instance_type = "t2.micro"
      subnet_id = "${aws_subnet.private-subnet.id}"
      vpc_security_group_ids = ["${aws_security_group.lekhikaSG2.id}"]
      key_name = "lekhika"
      tags = {
        Name = "MySQL"
      }
    }
Run the following commands accordingly after adding the above steps one-by-one in your code-

terraform init ( for installing the plugins)
terraform validate
terraform apply -auto-approve
## OUTPUTS:


<img src = screenshots/3.png>

<img src = screenshots/4.png>

<img src = screenshots/5.png>

<img src = screenshots/6.png>

<img src = screenshots/7.png>

<img src = screenshots/8.png>

<img src = screenshots/9.png>

<img src = screenshots/10.png>

<img src = screenshots/11.png>

# THANKYOU!!!
