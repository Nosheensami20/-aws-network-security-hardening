 # Phase II: Security Hardening (Lock Down Network Access)

Restrict who and what can access your VPC and its resources. Only necessary traffic is allowed; everything else is blocked.
Implements “least privilege network access.”

### Step 8: Security Groups Hardening and Auditing

### 8.1: Check the status of Existing SGs

Check the status of SGs that exist in our VPC for any overly permissive access like SSH or RDP from 0.0.0.0/0

```aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-<VPC-ID>" --profile networkengineer --region ca-central-1 --query "SecurityGroups[*].{Name:GroupName,ID:GroupId,Description:Description}" --output table```

Expected output:

                 DescribeSecurityGroups                      |
+-----------------------------+------------------------+----------+
|         Description         |          ID            |  Name    |
+-----------------------------+------------------------+----------+
|  default VPC security group |  sg-00000000000000000  |  default |
+-----------------------------+------------------------+----------+

The output shows only a default SG that exists in the VPC which has following configurations by default:

- All inbound traffic is allowed from the same SG (all instances within the same SG may talk freely) which is not ideal when we want to keep the instances isolated. 
- All outbound traffic is allowed 0.0.0.0/0, which is too broad as any resources inside the SG can reach to the entire internet. 

### 8.2: Create Restricted SG

SGs are created for both public and private resources with least priviliges and ensure security.

- Public SG ```aws ec2 create-security-group --group-name network-hardening-public-sg --description "Security group for public subnet resources (HTTP/HTTPS only)" --vpc-id vpc-ID --profile networkengineer --region ca-central-1```

Expected output:
    "GroupId": "sg-ID",
    "SecurityGroupArn": "arn:aws:ec2:ca-central-1:<AccountID>:security-group/sg-ID"

Next, we added rules to public SG:

- Allow HTTP (80) from anywhere: ```aws ec2 authorize-security-group-ingress --group-id sg-ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --profile networkengineer --region ca-central-1```
- Allowed HTTPS (443) from anywhere: ```aws ec2 authorize-security-group-ingress --group-id sg-ID --protocol tcp --port 443 --cidr 0.0.0.0/0 --profile networkengineer --region ca-central-1```
- Allow SSH (22) from admin IP only: ```aws ec2 authorize-security-group-ingress --group-id sg-ID --protocol tcp --port 22 --cidr <YOUR_PUBLIC_IP>/32 --profile networkengineer --region ca-central-1```

Verify: 
```aws ec2 describe-security-groups --group-ids sg-ID --profile networkengineer --region ca-central-1 --query "SecurityGroups[0].IpPermissions" --output json```

Expected Output includes:
      "IpProtocol": "tcp",
        "FromPort": 80,
        "ToPort": 80,
        "CidrIp": "0.0.0.0/0"
       "IpProtocol": "tcp",
        "FromPort": 22,
        "ToPort": 22,
       "CidrIp": "<Public IP>32"
       "IpProtocol": "tcp",
        "FromPort": 443,
        "ToPort": 443,
         "CidrIp": "0.0.0.0/0"
     

### 8.3: Create a private SG

```cmd /v:on /c "for /f "delims=" %i in ('aws ec2 create-security-group --group-name network-hardening-private-sg --description "Private SG for internal resources (restrict access to public SG)" --vpc-id <vpc-ID> --profile networkengineer --region ca-central-1 --query "GroupId" --output text') do (set PRIVSG=%i & echo PRIVSG=!PRIVSG!)"```

Expected output:
PRIVSG=sg-ID

- Check the private SG’s inbound rules:
```aws ec2 authorize-security-group-ingress --group-id sg-ID --protocol tcp --port 0-65535 --source-group <sg-ID> --profile networkengineer --region ca-central-1```

Expected output includes:

            "SecurityGroupRuleId": "sgr-ID",
            "IpProtocol": "tcp",
            "FromPort": 0,
            "ToPort": 65535,
      
- Allow the private SG to accept traffic from the public SG

For now we’ll allow all TCP from the public SG as an internal trust rule (you can and should tighten to specific ports later — e.g., 8080, 3306, etc.).

```aws ec2 authorize-security-group-ingress --group-id <PRIVSG_ID> --protocol tcp --port 0-65535 --source-group sg-ID --profile networkengineer --region ca-central-1```

Output:

```An error occurred (InvalidPermission.Duplicate) when calling the AuthorizeSecurityGroupIngress operation: the specified rule "peer: <sg-ID>, TCP, from port: 0, to port: 65535, ALLOW" already exists```
(This message confirms the rule already existed — no further action required.)

Verify: 
```aws ec2 describe-security-groups --group-ids sg-ID --profile networkengineer --region ca-central-1 --query "SecurityGroups[0].IpPermissions" --output json```
[
    {
        "IpProtocol": "tcp",
        "FromPort": 0,
        "ToPort": 65535,
        "UserIdGroupPairs": [
            {
                "UserId": "ID",
                "GroupId": "sg-ID"
            }
        ],
        "IpRanges": [],
        "Ipv6Ranges": [],
        "PrefixListIds": []
    }
]

### 8.4: Tightening rules for Private SG

The private SG currently allows all TCP (0-65535) from the public SG, but we replaced the broad rule with only specified required ports. 

- Revoke the broad rule:
```aws ec2 revoke-security-group-ingress --group-id sg-ID --protocol tcp --port 0-65535 --source-group <sg-ID> --profile networkengineer --region ca-central-1```

- Allow HTTP:
```aws ec2 authorize-security-group-ingress --group-id sg-ID --protocol tcp --port 8080 --source-group <sg-ID> --profile networkengineer --region ca-central-1```

- Allow MYSQL:

```aws ec2 authorize-security-group-ingress --group-id sg-ID --protocol tcp --port 3306 --source-group <sg-ID> --profile networkengineer --region ca-central-1``` 

Public SG is restricted to HTTP/HTTPS/SSH (admin IP only). Private SG allows only 8080 and 3306 from the public SG. Default SG unused — network access least-privilege enforced.
Final check:

Finally we checked if there is any instance that is still running under default SG.
``` aws ec2 describe-instances --filters "Name=instance.group-id,Values=<sg-ID>" --profile networkengineer --region ca-central-1 --query "Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,SGs:SecurityGroups}" --output json``` 

Output did not return any instance confirming all resources are under new SGs. 

### Step 9: Network ACL Audit & Hardening

The objective of this step is to review, audit, and harden all Network ACLs (NACLs) in our VPCs to ensure they follow least privilege and best practices for inbound and outbound traffic control.

### 9.1: List all NACLs
Listed all NACL that existed in our account and region to inspect the associations and find default/custom NACL, and subnets to which they apply. 

``` aws ec2 describe-network-acls --profile "networkengineer" --region ca-central-1 --query "NetworkAcls[].{NetworkAclId:NetworkAclId,VpcId:VpcId,IsDefault:IsDefault,SubnetIds:Associations[].SubnetId}" --output table``` 

From the output, check the VPC ID and type of associated NACL that was mentioned as default and subnets (private and public) that we created earlier we also associated. 

``` aws ec2 describe-network-acls --profile "networkengineer" --region ca-central-1 --query "NetworkAcls[].{NetworkAclId:NetworkAclId,Entries:Entries[].{RuleNumber:RuleNumber,Protocol:Protocol,RuleAction:RuleAction,Cidr:CidrBlock,Egress:Egress,PortRange:PortRange}}" --output table```

From the output, we checked for the applied rules with CIDR = 0.0.0.0/0:

- -1 Protocol: -1 = all protocols (TCP, UDP, ICMP, etc.)
- Allow 0.0.0.0/0 on rule 100 = This allows all traffic in and out from anywhere.
- Deny 0.0.0.0/0 on rule 32767 = This is the default “catch-all” deny rule that exists in every NACL.


### 9.2: Check risky NACLs and associated subnets

Next we need to check if this default NACL is associated with public subnet, if yes, it is a high risk because it opens up all ports and protocols to the world.

```aws ec2 describe-network-acls --profile "networkengineer" --region ca-central-1 --network-acl-ids acl-ID --query "NetworkAcls[].Associations[].SubnetId" --output table```

From the output, check for rules with `CidrBlock = 0.0.0.0/0`:

- Protocol -1 → all protocols (TCP, UDP, ICMP, etc.)
- Rule 100 Allow → allows all traffic from anywhere (risky for public subnets)
- Rule 32767 Deny → default catch-all deny rule

### 9.3: Hardening the Public Subnets

To implement least privilege network access, we need to allow only necessary traffic; everything else should be blocked.
To achieve this, we adopted a hardening strategy for public subnets:

**Inbound (Ingress)** 
Only allow traffic needed for the public subnet:
    - HTTP (80) and HTTPS (443) if it hosts a web server
    - SSH (22) only from trusted IPs (e.g., your office/public IP)
Deny everything else (catch-all deny is already rule 32767).

**Outbound (Egress)**
Allow outbound traffic needed for services:
    - HTTP/HTTPS (ports 80, 443)
    - DNS (53 UDP/TCP)
Deny everything else by default.

Therefore, we first deleted the all inbound and outbound permissions that exist by default:

**Delete Inbound** ```aws ec2 delete-network-acl-entry --profile "networkengineer" --region ca-central-1 --network-acl-id acl- ID--rule-number 100 --ingress```

**Delete Outbound** ```aws ec2 delete-network-acl-entry --profile "networkengineer" --region ca-central-1 --network-acl-id acl- ID--rule-number 100 --egress```

For hardened ingress, allowed SSH only from our IP, and HTTP and HTTPS from the internet.

- Created json policy files for SSH, HTTP, and HTTPS and created the nacl entry rules separately:

```aws ec2 create-network-acl-entry --profile networkengineer --region ca-central-1 --cli-input-json file://nacl-https.json```

For hardened egress, allowed HTTP, HTTPS, and DNS:

- Created json policy files for HTTP, HTTPS, and DNS and created nacl entries separately:
```aws ec2 create-network-acl-entry --profile networkengineer --region ca-central-1 --cli-input-json file://nacl-egress-http.json```

## NACL Hardening Verification:

Last step is to verify the rules have been applied correctly: 

```aws ec2 describe-network-acls --profile networkengineer --region ca-central-1 --network-acl-ids acl-ID --query "NetworkAcls[].Entries[]" --output table```

The table below shows the final NACL rules after hardening:

| RuleNumber | Egress | Protocol | PortRange | CidrBlock       | RuleAction |
|------------|--------|---------|-----------|----------------|------------|
| 100        | False  | 6       | 22        | IP/32          | allow      |
| 110        | False  | 6       | 80        | 0.0.0.0/0      | allow      |
| 120        | False  | 6       | 443       | 0.0.0.0/0      | allow      |
| 100        | True   | 6       | 80        | 0.0.0.0/0      | allow      |
| 110        | True   | 6       | 443       | 0.0.0.0/0      | allow      |
| 120        | True   | 17      | 53        | 0.0.0.0/0      | allow      |
| 32767      | False  | -1      | None      | 0.0.0.0/0      | deny       |
| 32767      | True   | -1      | None      | 0.0.0.0/0      | deny       |

We can see that:
- Public subnet NACL is hardened for ingress (SSH from our IP, HTTP/HTTPS from anywhere).
- Outbound (egress) rules allow only necessary traffic (HTTP, HTTPS, DNS).
- Default deny rules remain in place.
- Verification done via filtered CLI outputs.