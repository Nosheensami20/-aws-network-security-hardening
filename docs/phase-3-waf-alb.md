# Phase III: WAF Protection & ALB Hardening

In this phase, we deployed an Application Load Balancer (ALB) and protected it using AWS WAF.
We configured managed rule groups to block common web exploits, enabled CloudWatch logging, and confirmed that malicious requests were successfully blocked with 403 Forbidden.
Security Groups were hardened to ensure the EC2 instance is only reachable through the ALB.

# Step 10: Create an Application Load Balancer (ALB)

- Created an internet-facing ALB in public subnets of VPC only allowing 80 from 0.0.0.0/0.

```aws elbv2 create-load-balancer --name network-harden-alb --subnets <public-subnet-1> <public-subnet-2> --security-groups <alb-sg> --profile networkengineer```

- Created a target group:
Target group for the backend EC2 instance, ALB will forward traffic to this group.

```aws elbv2 create-target-group --name network-harden-tg --protocol HTTP --port 80 --vpc-id <your-vpc-id> --target-type instance --region ca-central-1 --profile networkengineer```

- Create ALB Listener for Port-80.
 To map incoming requests to the target group.

```aws elbv2 create-listener --load-balancer-arn <alb-arn> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<tg-arn> --profile networkengineer```

### 10.1: Create and Configure WAF WebACL

- We created WebACL name "network-harden-waf-webacl" and attached Managed rules:
- AWSManagedRulesCommonRuleSet
- AWSManagedRulesAnonymousIpList
- AWSManagedRulesKnownBadInputsRuleSet

```aws wafv2 create-web-acl --name network-harden-waf-webacl --scope REGIONAL --default-action Allow={} --description "Web ACL for ALB" --rules file://rules.json --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=network-harden-waf --region ca-central-1 --profile networkengineer```

- Associate WAF WebACL with the ALB.

```aws wafv2 associate-web-acl --web-acl-arn <webacl-arn> --resource-arn <alb-arn> --region ca-central-1 --profile networkengineer```

-  Enable WAF Logging to CloudWatch.

Created a dedicated log group:

```aws logs create-log-group --log-group-name "/aws/waf/network-harden-logs" --profile networkengineer```

Enabled logging: 

```aws wafv2 put-logging-configuration --logging-configuration file://waf-logging.json --region ca-central-1 --profile networkengineer```

### 10.2: Test the WAF Using Malicious Inputs

**Simulate SQL Injection Attack**

- Curl to simulate SQL Injection test ```curl -i "http://<ALB-DNS>/?id=' OR '1'='1"```

Output: 
403 Forbidden → WAF successfully blocked the attempt.

**Verified Logs in CloudWatch to confirm the logs were created**

```aws logs get-log-events --log-group-name "/aws/waf/network-harden-logs" --log-stream-name <stream-name> --profile networkengineer```

Output showed the Blocked requests, Rule that blocked, Source IP & country,URI and method, 

Example of real malicious traffic WAF blocked:

uri = "/.env"
terminatingRule = ExploitablePaths_URIPATH
action = BLOCK
clientIp = 146.103.3.7

### 10.3: Harden ALB Security Group

Restricted ALB SG rules and removed unnecessary inbound rules.

Inbound: 80 from 0.0.0.0/0  
Outbound: 0.0.0.0/0 (default)

### 10.4: Harden EC2 Instance Security Group

To ensure the EC2 instance is not publicly accessible, we restricted inbound traffic only from the ALB SG
Inbound:   HTTP 80 → Source: sg-alb
Outbound:  Allow all (default)

**Achievements:**
- WAF successfully blocking real attacks (403 responses)
- CloudWatch logging capturing threat intelligence
- EC2 instances secured behind ALB
- Defense-in-depth architecture complete

**Security Posture:**
- Application layer protected by WAF with managed rules
- All traffic filtered before reaching backend
- Comprehensive logging for audit and compliance
- Zero direct internet access to application servers
