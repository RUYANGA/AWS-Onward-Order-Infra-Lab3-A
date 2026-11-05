# Discovery report: delivering order system updates safely

Client: Onward Logistics. Module: Compute & Delivery on AWS, week 3.
Copy these sections into the Discovery Report Template.

## Part 1. What we found

### The problem today

The order system is one containerised full-stack application. Releases are done by hand: an engineer builds an image on a laptop, uploads it and updates the running service. This makes every release slow and dependent on one person's machine and memory, risky (no consistent tests, no record of exactly what was deployed, no quick way back), and something the team avoids on peak days, so fixes and features wait.

### What happens with no solution

Releases stay infrequent and large, so each one carries more change and more risk. A bad release reaches every customer at once and recovery means another manual build under pressure. Laptop builds can include unreviewed or uncommitted code. Access keys used for manual uploads tend to live on laptops and in scripts, where they leak. With no automatic scaling the service is either over-provisioned all year or under-provisioned on peak days. Knowledge of how releases work sits with a few people, so new engineers depend on guided tours.

### Common solutions for this kind of problem

| Approach | How it works | Fit for Onward |
|---|---|---|
| Manual releases with a checklist | Same as today, written down | Cheapest to start, keeps every risk above |
| ECS rolling deployments via CI | CI builds the image and calls `UpdateService`; ECS replaces tasks gradually | Simple, but old and new versions mix and rolling back is another deployment |
| ECS blue/green with CodeDeploy | New version starts beside the old one; traffic shifts at the load balancer, with test traffic, canary and alarm-based rollback | Meets zero-downtime and fast-rollback directly |
| AWS App Runner | Fully managed container service that deploys on image push | Very little to manage, but no VPC-private-only design matching Security's rules or CodeDeploy-style traffic control |
| Amazon EKS with GitOps (Argo CD / Flux) and Argo Rollouts | Kubernetes with progressive delivery | Powerful, but a cluster fee and a large learning curve for one service |
| Elastic Beanstalk | Managed platform with immutable or blue/green (CNAME swap) deployments | Less control over networking; DNS-based swaps are slower to reverse |
| Serverless (Lambda + API Gateway) with aliases | Weighted alias traffic shifting | Would require re-architecting the application |

For outbound access from private subnets, the two common patterns are a NAT gateway (simple, but gives a route to the internet and charges per GB) and VPC endpoints (traffic to AWS services stays on AWS's network, no internet route). For CI credentials the two patterns are stored IAM access keys (long-lived) and OIDC federation (short-lived, no stored secret).

## Part 2. Business value

### Chosen approach

Run the application on **Amazon ECS with AWS Fargate** in private subnets across two Availability Zones, behind one internet-facing **Application Load Balancer**. Engineers merge a pull request; **GitHub Actions** tests and builds the image and pushes it to **Amazon ECR** using **OIDC** short-lived credentials. The push raises an **Amazon EventBridge** event that starts **AWS CodePipeline**, which hands the release to **AWS CodeDeploy** for a **blue/green** deployment: the new version runs alongside the old, receives 10% of customers for five minutes, then all traffic, while CloudWatch alarms roll it back automatically if errors appear. The previous version stays available for 15 minutes for one-click rollback. Tasks reach ECR, S3 and CloudWatch Logs through **VPC endpoints** instead of a NAT gateway. All infrastructure is **AWS CloudFormation**, deployed by **CloudFormation GitSync** from its own repository.

### Services and why

| Service | Why it was chosen |
|---|---|
| Amazon ECS on AWS Fargate | Runs containers without servers to patch; spreads tasks across AZs; replaces failed tasks automatically |
| Application Load Balancer | The single public entry point; health checks; two target groups make blue/green possible |
| Application Auto Scaling | Adds tasks on busy days and removes them afterwards without anyone adjusting capacity |
| VPC endpoints (ECR API, ECR Docker, CloudWatch Logs; S3 gateway) | Private path to AWS services with no internet route; cheaper than NAT gateways across two AZs |
| GitHub Actions + IAM OIDC | Builds and tests every change; no stored AWS keys |
| Amazon ECR | Private image registry with scan-on-push and a retention rule that keeps 30 releases for rollback |
| Amazon EventBridge | Starts a release the moment a new image is published |
| AWS CodePipeline + AWS CodeDeploy | Auditable release history; blue/green with canary, test listener and automatic rollback |
| Amazon CloudWatch | Logs, Container Insights and the alarms that guard every release |
| AWS CloudFormation + GitSync | Every resource defined in reviewed code; the whole platform can be rebuilt from the repositories |

### Who benefits

| Stakeholder | Benefit |
|---|---|
| Chief Technology Officer | Merging is the only manual step; no outages from releases; rollback in minutes; layered, commented repositories a new engineer can follow |
| Head of Security | No long-lived credentials anywhere; one public entry point; containers have no internet route; least-privilege, managed-policy IAM |
| Head of Operations | Two AZs, at least two tasks, self-healing, CPU-based scaling from 2 to 6 tasks with no manual changes |
| Finance Lead | Pay-per-use compute that shrinks after peaks; endpoint costs explained and lower than NAT; dev can scale to zero off-hours or be deleted and rebuilt |
| Merchants and customers | A service that stays up during releases and peak days, with fixes delivered sooner |
| Engineers | Small, frequent, low-stress releases; no laptop builds |
