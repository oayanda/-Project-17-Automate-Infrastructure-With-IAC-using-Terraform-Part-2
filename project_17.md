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

## Dynamic Tagging of  Resources
Tags or tagging are very important because it enables the effective management or investigate any aspect of AWS environment.

Create two variables `name` and `tags` in the variable.tf file.

```bash
variable "name" {
  type = string
  default = "oayanda"
  
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```

![variables](/images/var.png)

 Assign values for the `tags` variable in the `terraform.tfvars` file

```bash
tags = {
  Enviroment      = "production" 
  Owner-Email     = "oayanda@oayanda.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

![variables](/images/tags.png)

Instantiate the `tags` in the Public and Private subnet blocks of the `main.tf` file. The `tags` here is an argument which make use another variable called `tags`.

`Merge()` takes an arbitrary number of maps or objects, and returns a single map or object that contains a merged set of elements from all arguments.
We use `merge funtion` to combine two of the variables listed eariler and a `format function` to display the dynamic tag accordingly for the subnet resource or any other resources.

The `format function` produces a string by formatting a number of other values according to a specification string.

```bash
tags = merge(
    var.tags,
    {
      Name = format("%s-PublicSub-%s", var.name, count.index)
    }
  )
  ```
From the above snippet, `var.name` becomes the prefix and `count.index` which is loop becomes the suffix for each iteration.

  ![variables](/images/pu.png)

Let's run `terraform plan` to see the effect

 ![variables](/images/t.png)

## Internet Gateway

To create the internet gateway, declare the `aws_internet_gateway` and declare the `vcp_id` which makes sure the internet gateway is attached automatically.

```bash
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-%s", aws_vpc.main.id,"IGW")
    } 
  )
}
```

 ![variables](/images/igw.png)

 ## Create 1 NAT Gateways and 1 Elastic IP (EIP) addresses

```bash
# Elastic IP

resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP", var.name)
    },
  )
}

# Nat gateway and elastic Ip attachement
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}
```


