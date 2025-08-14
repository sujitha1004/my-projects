# my-projects
#vpc creation
resource "aws_vpc" "vcube" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "MyVPC"
  }
}
# Two Public Subnets (different AZs)
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.vcube.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"

  tags = {
    Name = "public-subnet-a"
  }
}
resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.vcube.id
  cidr_block              = "10.0.3.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1b"

  tags = {
    Name = "public-subnet-b"
  }
}

#igw

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vcube.id

  tags = {
    Name = "MainIGW"
  }
}

#pubrt
# Public Route Table

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.vcube.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-rt"
  }
}

# Association for Public Subnet A

resource "aws_route_table_association" "public_a_assoc" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

# Association for Public Subnet B

resource "aws_route_table_association" "public_b_assoc" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

# Security Group for ALB

resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow HTTP traffic"
  vpc_id      = aws_vpc.vcube.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
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
    Name = "alb-sg"
  }
}
# Application Load Balancer

resource "aws_lb" "app_lb" {
  name               = "my-app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [
    aws_subnet.public_a.id,
    aws_subnet.public_b.id
  ]

  tags = {
    Name = "my-app-lb"
  }
}
# Target Group

resource "aws_lb_target_group" "app_tg" {
  name     = "my-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.vcube.id

  health_check {
    path                = "/"
    protocol            = "HTTP"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
  tags = {
    Name = "my-target-group"
  }
}

# Listener

resource "aws_lb_listener" "app_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}
# Example EC2 in public subnet A
resource "aws_instance" "web_a" {
  ami           = "ami-084a7d336e816906b" # Replace with a valid AMI ID for your region
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_a.id
  security_groups = [aws_security_group.instance_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello from EC2 in subnet A" > /var/www/html/index.html
              yum install -y httpd
              systemctl start httpd
              EOF

  tags = {
    Name = "web-a"
  }
}

# Security group for EC2 instances
resource "aws_security_group" "instance_sg" {
  name        = "instance-sg"
  description = "Allow HTTP from ALB"
  vpc_id      = aws_vpc.vcube.id

  ingress {
    description = "HTTP from ALB"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    security_groups = [aws_security_group.alb_sg.id] # Only ALB can talk to EC2
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "instance-sg"
  }
}

# Data source to fetch latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["amazon"] # Amazon's official AMI owner ID
}

# Example EC2 instance using the fetched AMI
resource "aws_instance" "example" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.instance_sg.id]

  tags = {
    Name = "example-instance"
  }
}

# Attach EC2 to target group
resource "aws_lb_target_group_attachment" "app_tg_attach_a" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.web_a.id
  port             = 80
}

# -------------------------------
# Launch Template (EC2 config)
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"  # <-- Replace with your desired AMI ID
  instance_type = "t2.micro"

  tags = {
    Name = "example-instance"
  }
}

# -------------------------------
resource "aws_launch_template" "web_lt" {
  name_prefix   = "web-lt-"
  image_id      = "ami-084a7d336e816906b" # Replace with a valid Amazon Linux 2 AMI ID for your region
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.instance_sg.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              echo "Hello from AutoScaling instance" > /var/www/html/index.html
              systemctl start httpd
              systemctl enable httpd
              EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "ASG-Web-Instance"
    }
  }
}

# -------------------------------
# Auto Scaling Group
# -------------------------------
resource "aws_autoscaling_group" "web_asg" {
  desired_capacity     = 2
  max_size             = 3
  min_size             = 1
  vpc_zone_identifier  = [aws_subnet.public_a.id, aws_subnet.public_b.id] # Place in both public subnets
  target_group_arns    = [aws_lb_target_group.app_tg.arn]

  launch_template {
    id      = aws_launch_template.web_lt.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "ASG-Web"
    propagate_at_launch = true
  }
}

# -------------------------------
# Scaling Policies
# -------------------------------
resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out-policy"
  autoscaling_group_name = aws_autoscaling_group.web_asg.name
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
}

resource "aws_autoscaling_policy" "scale_in" {
  name                   = "scale-in-policy"
  autoscaling_group_name = aws_autoscaling_group.web_asg.name
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
}
