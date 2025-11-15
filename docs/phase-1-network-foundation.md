**Phase I: Network Foundation (Building a Secure VPC):**
The first phase of our project is to build a private, segmented AWS network with proper routing and connectivity. We developed a two-tier network — public layer (internet-facing) and private layer (internal systems).

### Step 1: Create IAM User and Attach Policy

- Created the user "networkengineer".
- Created IAM policy and attached policy to the user. 
- Created access keys and configured the networkengineer AWS CLI profile.

### Step 2: Create a VPC

We created an isolated private network in AWS to enforce internal security policies and organize environment.

- Created a VPC ```aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ca-central-1 --profile networkengineer --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=network-hardening-vpc}]```

- Checked VPC status ```aws ec2 describe-vpcs --vpc-ids <vpc-ID> --profile networkengineer --region ca-central-1 --query "Vpcs[0].State" --output text```

Expected Output:
Status= "Available"

 ### Step 3: Create Public and Private Subnets

A public and a private subnet was created in different availability zones to ensure that sensitive resources are not exposed to the internet and stay isolated.

- Create a public subnet (10.0.1.0/24 in ca-central-1a):
```aws ec2 create-subnet --vpc-id <vpc-ID> --cidr-block 10.0.1.0/24 --availability-zone ca-central-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=network-hardening-public-subnet}]" --profile networkengineer --region ca-central-1```

- Enable automatic public IP assignment:
```aws ec2 modify-subnet-attribute --subnet-id <public-subnet-id> --map-public-ip-on-launch --profile networkengineer --region ca-central-1```

- Create a private subnet (10.0.2.0/24 in ca-central-1b): 
```aws ec2 create-subnet --vpc-id <vpc-ID> --cidr-block 10.0.2.0/24 --availability-zone ca-central-1b --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=network-hardening-private-subnet}]" --profile networkengineer --region ca-central-1```

### Step 4: Create Internet Gateway and attach to VPC

Created a dedicated IGW for VPC to allow public subnets to connect to the Internet.

- Create an Internet Gateway:
```aws ec2 create-internet-gateway --profile networkengineer --region ca-central-1 --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=network-hardening-igw}]"```
 
 Expected Output:
    "InternetGatewayId": "<igw ID>",
    
- Attach the IGW to VPC:
```for /f "delims=" %i in ('aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=network-hardening-igw" --profile networkengineer --region ca-central-1 --query "InternetGateways[0].InternetGatewayId" --output text') do @aws ec2 attach-internet-gateway --internet-gateway-id %i --vpc-id vpc----- --profile networkengineer --region ca-central-1 & echo Attached IGW=%i```

- Verify:
```aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=<vpc-ID>" --profile networkengineer --region ca-central-1 --query "InternetGateways[0].Attachments[0]" --output json```

Expected Output:
    "VpcId": "<vpcID>",
    "State": "attached"

### Step 5: Create Public Route Tables and routes

- Create a route table to control flow of traffic inbound and outbound traffic of public subnet:
```for /f "delims=" %i in ('aws ec2 create-route-table --vpc-id <VPC ID> --profile networkengineer --region ca-central-1 --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=network-hardening-public-rt}]" --query "RouteTable.RouteTableId" --output text') do @set RT=%i & echo Created RT=%RT%```

- Check for RT ID:
```for /f "delims=" %i in ('aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-<VPC ID>" "Name=tag:Name,Values=network-hardening-public-rt" --profile networkengineer --region ca-central-1 --query "RouteTables[0].RouteTableId" --output text') do @echo Found RT=%i```

- Create a default route to send all non-local traffic to the IGW, allowing the public subnet to access the internet:
```aws ec2 create-route --route-table-id <rtb-ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-ID> --profile networkengineer --region ca-central-1```

- Associate the route table with public subnet:
```aws ec2 associate-route-table --route-table-id <rtb-ID> --subnet-id <public-subnet-id> --profile networkengineer --region ca-central-1```

- Verify the route table now contains the 0.0.0.0/0 → IGW route and is associated with the public subnet:

```aws ec2 describe-route-tables --route-table-ids %RT% --profile networkengineer --region ca-central-1 --query "RouteTables[0].{RT:RouteTableId,Routes:Routes,Associations:Associations}" --output json```

### Step 6: Create NAT Gateway and Private Route Table

Allow instances in the private subnet to reach the internet securely for outbound traffic (e.g., updates or package downloads) while blocking all inbound access. This is done by creating a NAT Gateway in the public subnet and routing private subnet traffic through it.

- Allocate an Elastic IP for the NAT Gateway:

This Elastic IP will be assigned to the NAT Gateway so it can communicate with the internet via the Internet Gateway (IGW)

```for /f "delims=" %i in ('aws ec2 allocate-address --domain vpc --profile networkengineer --region ca-central-1 --query "AllocationId" --output text') do @set EIP=%i & echo EIP=%EIP%```

- Create NAT Gateway in Public Subnet to allow outbound- only internet access safely:

The NAT Gateway must be created in the public subnet (so it can use the IGW).

```for /f "delims=" %i in ('aws ec2 create-nat-gateway --subnet-id <subnet-ID> --allocation-id <eipalloc-ID> --profile networkengineer --region ca-central-1 --query "NatGateway.NatGatewayId" --output text') do (set NAT=%i & echo NAT=!NAT!)```

- Verify:
```aws ec2 describe-nat-gateways --nat-gateway-ids nat-<ID> --profile networkengineer --region ca-central-1 --query "NatGateways[0].State" --output text```

- Create Private Route Table for VPC:

This route table will manage traffic for the private subnet.

```cmd /v:on /c "for /f "delims=" %i in ('aws ec2 create-route-table --vpc-id <VPC-ID> --profile networkengineer --region ca-central-1 --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=network-hardening-private-rt}]" --query "RouteTable.RouteTableId" --output text') do (set PRT=%i & echo PRT=!PRT!)"```

- Add default route to the NAT Gateway:

This sends all outbound internet traffic from the private subnet to the NAT Gateway.

```aws ec2 create-route --route-table-id rtb-ID --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <nat-ID> --profile networkengineer --region ca-central-1```

- Associate the Private Route Table with the Private Subnet:

Link the route table to your private subnet so its routing rules apply.
```aws ec2 associate-route-table --route-table-id rtb-ID --subnet-id subnet-ID --profile networkengineer --region ca-central-1```

Expected Output:

    "AssociationId": "<rtbassoc ID>",
    "AssociationState": "State": "associated"

- Verify:

```aws ec2 describe-route-tables --route-table-ids <ID>--profile networkengineer --region ca-central-1 --query "RouteTables[0].Associations" --output json```

Expected Output includes:
            "DestinationCidrBlock": "10.0.0.0/16",
            "GatewayId": "local",
            "State": "active"
            "DestinationCidrBlock": "0.0.0.0/0",
            "NatGatewayId": "nat-ID",
            "Origin": "CreateRoute",
            "State": "active"
            "RouteTableAssociationId": "rtbassoc-ID",
            "RouteTableId": "rtb-ID",
            "SubnetId": "subnet-ID",
            "AssociationState":"associated"
   
### Step 7: Verify Deployed Resources

- VPC 10.0.0.0/16 — available
- Public Subnet 10.0.1.0/24 — IGW attached
- Private Subnet 10.0.2.0/24 — NAT Gateway route configured
- IGW + NAT Gateway verified operational