

Download terraform [here](https://learn.hashicorp.com/tutorials/terraform/install-cli)

Start by setting up a Provider 

```terraform
provider "aws" {
  region  = "us-east-2"
  profile = "default"
}
```

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

```terraform
 resource "aws_security_group" "allow_web" {
  name        = "allow_web"
  description = "Allow Web inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description      = "HTTPS"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  ingress {
    description      = "HTTP"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }


  ingress {
    description      = "SSH"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }


  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_web"
  }
}
```

7. Create a network interface with an ip in the subnet that was created in step 4

```terraform
resource "aws_network_interface" "web-server-nic" {
  subnet_id       = aws_subnet.subnet-1.id
  private_ips     = ["10.0.1.50"]
  security_groups = [aws_security_group.allow_web.id]

}
```

8. Assign an elastic IP to the network interface create in step 7

```terraform
resource "aws_eip" "one" {
  vpc                       = true
  network_interface         = aws_network_interface.web-server-nic.id
  associate_with_private_ip = "10.0.1.50"
  depends_on = [
    aws_internet_gateway.gw
  ]
}
```

9. Create aws server using EC2 instance

```terraform
resource "aws_instance" "my_first_tf_server" {
  ami               = "ami-0568773882d492fc8"
  instance_type     = "t2.micro"
  availability_zone = "us-east-2a"
  key_name   = "main-key"


  network_interface {
    network_interface_id = aws_network_interface.web-server-nic.id
    device_index         = 0


  }

  user_data = <<-EOF
            #!/bin/bash
            sudo yum update -y
            sudo yum install -y httpd
            sudo systemctl enable httpd
            sudo systemctl start httpd

            MYINSTANCE=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
            MYADDRESS=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
            EC2_AVAIL_ZONE=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

            echo "<h1>Hello World from $MYADDRESS  </h1>" >> /var/www/html/index.html
            echo "<h1>Chilling in AZ $EC2_AVAIL_ZONE</h1>" >> /var/www/html/index.html
            echo "<h1>Instance ID $MYINSTANCE </h1>" >> /var/www/html/index.html
            EOF

  tags = {
    "Name" = "web-server"
  }

  #Option to use a sep. script variable

  # user_data = file("script.sh")
  # tags = {
  #   Name = "web-server"
  # }


}
```
