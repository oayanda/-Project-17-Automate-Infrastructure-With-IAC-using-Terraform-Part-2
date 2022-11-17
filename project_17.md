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

Let's run `terraform plan` to see this in effect

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

The Nat gateway needs to be in a public subnet in order for it enable the private subnets communicate with public. 

Using the `element function` helps retrieves a single element from a list of public subnets which are configured dynamically.
element(aws_subnet.public.*.id, 0) - where 0 is the first index

```bash
# Elastic IP

resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.igw]

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
  depends_on    = [aws_internet_gateway.igw]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}
```
 ![variables](/images/nat.png)

## AWS ROUTES

Now let's create a file called route_tables.tf and use it to create routes for both public and private subnets, create the below resources and ensure they are properly tagged.

- aws_route_table
- aws_route
- aws_route_table_association

```bash
# create private route table
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

# associate all private subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private[*].id)
  subnet_id      = element(aws_subnet.private[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

# create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-Route-Table", var.name)
    },
  )
}

# create route for the public route table and attach the internet gateway
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

# associate all public subnets to the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public[*].id)
  subnet_id      = element(aws_subnet.public[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
}
```

## AWS Identity and Access Management

IaM and Roles
We want to pass an IAM role to our EC2 instances to give them access to some specific resources, so we need to do the following:

Create AssumeRole
Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.

```bash
resource "aws_iam_role" "ec2_instance_role" {
name = "ec2_instance_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  tags = merge(
    var.tags,
    {
      Name = "aws assume role"
    },
  )
}
```

In this code we are creating AssumeRole with AssumeRole policy. It grants to an entity, in our case it is an EC2, permissions to assume the role.

### Create IAM policy for this role

This is where we need to define a required policy (i.e., permissions) according to our requirements. For example, allowing an IAM role to perform action describe applied to EC2 instances

```bash

resource "aws_iam_policy" "policy" {
  name        = "ec2_instance_policy"
  description = "A test policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:Describe*",
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]

  })

  tags = merge(
    var.tags,
    {
      Name =  "aws assume policy"
    },
  )

}
```

### Attach the Policy to the IAM Role

This is where, we will be attaching the policy which we created above, to the role we created in the first step.

```bash
 resource "aws_iam_role_policy_attachment" "test-attach" {
        role       = aws_iam_role.ec2_instance_role.name
        policy_arn = aws_iam_policy.policy.arn
    }

```

### Create an Instance Profile and interpolate the IAM Role

```bash
 resource "aws_iam_instance_profile" "ip" {
        name = "aws_instance_profile_test"
        role =  aws_iam_role.ec2_instance_role.name
    }

```
 ![variables](/images/role.png)