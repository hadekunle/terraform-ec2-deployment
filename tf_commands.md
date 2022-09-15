

This is  [website](https://www.youtube.com/watch?v=SLB_c_ayRMo&ab_channel=freeCodeCamp.org) to the full video on this set up



Start by setting up a Provider 

provider "aws" {
  region  = "us-east-2"
  profile = "default"
}


1. Create VPC

```terraform
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "prod"
  }
}
```

2. Create Internet Gateway
```terraform
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "production"
  }
}
```

3. Create Custom Route Table

```terraform
resource "aws_route_table" "prod_route_table" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  route {
    ipv6_cidr_block        = "::/0"
    egress_only_gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "example"
  }
```
4. Create a subnet
```terraform
resource "aws_subnet" "subnet-1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
 availability_zone = "us-east-2a"
  tags = {
    Name = "prod-subnet"
  }
}
```

5. Associate subnet with Route Table

```terraform
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet-1.id
  route_table_id = aws_route_table.prod_route_table.id
}
```

6. Create Security Group to allow port 22,80,443

7. Create a network interface with an ip in the subnet that was created in step 4


8. Assign an elastic IP to the network interface create in step 7


9. Create aws server and install/enable apache2