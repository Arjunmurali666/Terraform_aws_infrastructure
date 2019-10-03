# Terraform_aws_infrastructure

## Terraform - This code is written in terraform to setup an infrastructure in aws. Let me just explain the work which are going to complete using this code. 

## First it will create a new VPC in aws and one public subnet and one private subnet using the new VPC. Next creating an Internet gateway which will be assigned to the public subnet and a Nat gateway which will be assigned to the private subnet.

## Next step, we need to create an elastic IP for the nat gateway and associate it with the created nat gateway. Then we will create 2 ec2 instance. First one will be the webserver which will be created under the public subnet.

## The next one will be database server which will create under private subnet. So that these two instances can communicate each other but no other servers can communicate with the database from outside since there won't be any public IP assigned to the database server.

## Webserver can communicate with the Database server using the private IP

## Now create a directory and put this code inside the directory with a file name ending with ".tf"

## Then initialize terraform "$terraform init" and run the code "$terraform apply"


```tf
provider "aws" {
    access_key = [put_your_access_key]
    secret_key = [put_your_secret_key]
    region = "us-east-2"
}

#=========================================================================
# Variable Declaration
#=========================================================================

variable "ami" { 
    default = "ami-00c03f7f7f2ec15c3"
}

variable "vpc" {
    type = "map"
    default = {
        "name" = "blog"
        "cidr" = "172.16.0.0/16"
    }
}

variable "public1" {
    type = "map"
    default = {
        "name" = "blog-public-1"
        "cidr" = "172.16.0.0/20"
        "az" = "us-east-2a"
    }
}

variable "public2" {
    type = "map"
    default = {
        "name" = "blog-public-2"
        "cidr" = "172.16.16.0/20"
        "az" = "us-east-2b"
    }   
}

variable "private1" {
    type = "map"
    default = {
        "name" = "blog-private-1"
        "cidr" = "172.16.32.0/20"
        "az" = "us-east-2c"
    }   
}


variable "igw" {
    default = "blog-igw"
}

#=========================================================================
# Creating vpc
#=========================================================================

resource "aws_vpc" "vpc" {
  cidr_block = "${var.vpc.cidr}"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
        Name = "${var.vpc.name}"
  }
}

#=========================================================================
# Creating subnet public 1
#=========================================================================

resource "aws_subnet" "public1" {
  vpc_id = "${aws_vpc.vpc.id}"
  cidr_block = "${var.public1.cidr}"
  map_public_ip_on_launch = "true"
  availability_zone = "${var.public1.az}"
  tags = {
    Name = "${var.public1.name}"
  }
}

#=========================================================================
# Creating subnet public 2
#=========================================================================

resource "aws_subnet" "public2" {
  vpc_id = "${aws_vpc.vpc.id}"
  cidr_block = "${var.public2.cidr}"
  map_public_ip_on_launch = "true"
  availability_zone = "${var.public2.az}"
  tags = {
    Name = "${var.public2.name}"
  }
}


#=========================================================================
# Creating subnet private 1
#=========================================================================

resource "aws_subnet" "private1" {
  vpc_id = "${aws_vpc.vpc.id}"
  cidr_block = "${var.private1.cidr}"
  map_public_ip_on_launch = "false"
  availability_zone = "${var.private1.az}"
  tags = {
    Name = "${var.private1.name}"
  }
}
#=========================================================================
# Creating Internet-Gate-Way
#=========================================================================

resource "aws_internet_gateway" "igw" {
  vpc_id = "${aws_vpc.vpc.id}"
  tags = {
        Name = "${var.igw}"
  }
}

#=========================================================================
# Creating Elastic Ip for the nat instance
#=========================================================================
resource "aws_eip" "nat" {
    vpc      = true
}

#=========================================================================
# Creating Nat GateWay
#=========================================================================

resource "aws_nat_gateway" "nat" {
  allocation_id = "${aws_eip.nat.id}"
  subnet_id     = "${aws_subnet.public2.id}"
}

#=========================================================================
# Creating RoteTable for the public subnets
#=========================================================================
resource "aws_route_table" "rtb-public" {
  
    vpc_id = "${aws_vpc.vpc.id}"
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = "${aws_internet_gateway.igw.id}"
    }
    
    tags = {
        Name = "blog-rtb-public"
    }
}

#=========================================================================
# public1 association
#=========================================================================

resource "aws_route_table_association" "public1-association" {
  subnet_id      = "${aws_subnet.public1.id}"
  route_table_id = "${aws_route_table.rtb-public.id}"
}

#=========================================================================
# public2 association
#=========================================================================

resource "aws_route_table_association" "public2-association" {
  subnet_id      = "${aws_subnet.public2.id}"
  route_table_id = "${aws_route_table.rtb-public.id}"
}

#=========================================================================
# Creating RoteTable for the private subnets
#=========================================================================
resource "aws_route_table" "rtb-private" {
  
    vpc_id = "${aws_vpc.vpc.id}"
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = "${aws_nat_gateway.nat.id}"
    }
    
    tags = {
        Name = "blog-rtb-private"
    }
}

#=========================================================================
# private1 association
#=========================================================================

resource "aws_route_table_association" "private1-association" {
  subnet_id      = "${aws_subnet.private1.id}"
  route_table_id = "${aws_route_table.rtb-private.id}"
}

#=========================================================================
# security group
#=========================================================================
resource "aws_security_group" "blog-sec" {

    name         = "blog-sec"
    description  = "allows all"
    vpc_id =  "${aws_vpc.vpc.id}"
    ingress {
        cidr_blocks = ["0.0.0.0/0"]  
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
      }

    egress {
        from_port       = 0
        to_port         = 0
        protocol        = "-1"
        cidr_blocks     = ["0.0.0.0/0"]
    }

    tags = {
        Name = "blog-sec"
       
    }
}

#=========================================================================
# webserver instance creation
#=========================================================================

resource "aws_instance" "webserver" {
    ami = "${var.ami}" 
    instance_type = "t2.micro"
    key_name = "ansible"
    subnet_id  = "${aws_subnet.public1.id}"
    associate_public_ip_address = true
    vpc_security_group_ids = ["${aws_security_group.blog-sec.id}"]
    tags = {
        Name = "Webserver"
  }
}

#=========================================================================
# database instance creation
#=========================================================================

resource "aws_instance" "database" {
    ami = "${var.ami}" 
    instance_type = "t2.micro"
    key_name = "ansible"
    subnet_id  = "${aws_subnet.private1.id}"
    vpc_security_group_ids = ["${aws_security_group.blog-sec.id}"]
    tags = {
        Name = "Database"
  }
}
```
