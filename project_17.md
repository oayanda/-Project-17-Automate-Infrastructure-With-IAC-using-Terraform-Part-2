# Automating AWS Infrastructure with Terraform

The VPC and the 2 required public subnets were created in the previous project.
Let's continue by creating the required 4 private subnets

## Create 4 Private Subnets

```bash
# Create 4 private subnets
resource "aws_subnet" "private" {
  count                   = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 2)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "oayanda_prisubnet_${count.index + 1}"

  }
}

```

![private subnet](/images/private.png)
![private subnet](/images/p2.png)

Dynamic Tagging of  Resources


