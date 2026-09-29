# AWS: Questions Only

Questions without answers, for self-testing. Model answers are in [aws.md](aws.md).

## 1. Global infrastructure and networking

- 1.1 Regions, Availability Zones and edge locations?
- 1.2 Describe a typical VPC design for a web service.
- 1.3 Security groups vs NACLs?
- 1.4 How do you connect VPCs and on-prem networks?
- 1.5 Gateway vs interface VPC endpoints? Why use them?

## 2. Identity and security

- 2.1 IAM users vs roles vs policies?
- 2.2 How is an IAM request evaluated?
- 2.3 How do you give CI/CD pipelines access to AWS without long-lived keys?
- 2.4 How do you manage secrets on AWS?
- 2.5 What is KMS envelope encryption?
- 2.6 How do you structure AWS accounts?

## 3. Compute

- 3.1 EC2 vs ECS vs EKS vs Lambda vs Fargate: how do you choose?
- 3.2 How does Rust perform on Lambda?
- 3.3 What are Lambda cold starts and how do you mitigate them?
- 3.4 What is Lambda concurrency and why can it hurt?
- 3.5 Auto Scaling Groups: which scaling policies exist?
- 3.6 When should you use Spot instances?
- 3.7 Why consider Graviton (arm64)?

## 4. Storage and databases

- 4.1 S3 consistency, storage classes and performance?
- 4.2 RDS vs Aurora?
- 4.3 How does DynamoDB partitioning work and how do you design keys?
- 4.4 DynamoDB on-demand vs provisioned capacity? What are its consistency options?
- 4.5 ElastiCache: Redis/Valkey vs Memcached?
- 4.6 EBS vs EFS vs instance store?

## 5. Messaging and integration

- 5.1 SQS standard vs FIFO?
- 5.2 What is the SQS visibility timeout and what goes wrong with it?
- 5.3 SNS vs SQS vs EventBridge vs Kinesis vs MSK?
- 5.4 What is Step Functions used for?

## 6. Load balancing, DNS and edge

- 6.1 ALB vs NLB vs GWLB?
- 6.2 What does connection draining (deregistration delay) do?
- 6.3 Which Route 53 routing policies exist?
- 6.4 What does CloudFront provide beyond caching?
- 6.5 What is API Gateway and when would you use it over an ALB?

## 7. Observability

- 7.1 How do you monitor a service on AWS?
- 7.2 What is CloudTrail vs CloudWatch vs AWS Config?

## 8. Reliability, scaling and cost

- 8.1 What are the six pillars of the Well-Architected Framework?
- 8.2 Describe disaster recovery strategies.
- 8.3 How would you design a multi-region active-active service?
- 8.4 What are common AWS service limits and throttling issues?
- 8.5 How do you optimize AWS costs?
- 8.6 Which data transfer costs catch teams by surprise?

## 9. Infrastructure as code

- 9.1 Terraform vs CloudFormation vs CDK?
- 9.2 How do you manage Terraform state and environments safely?
