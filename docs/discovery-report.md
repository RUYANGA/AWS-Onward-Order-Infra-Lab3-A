# Discovery Report: Delivering Order System Updates Safely for Onward Logistics

| | |
|---|---|
| **Lab** | Delivering Order System Updates Safely for Onward Logistics (Compute & Delivery on AWS, week 3) |
| **Name** | Merci RUYANGA |
| **Date** | 16 November 2026 |
| **Client** | Onward Logistics (see the Client Profile) |

## 1. What I found

### The client's need

Onward Logistics needs to put fixes and new features in front of merchants and customers often and without fear: no downtime, no surprises, and a quick way back when a release goes wrong.

Today one engineer builds the order system on a laptop, uploads it and swaps the running service by hand. Releases are slow, depend on one person, leave no record of exactly what went live, and are avoided on peak days.

### Solutions that exist

| Solution | How it works | Trade-off for this client |
|---|---|---|
| **1. Rolling updates from CI** (e.g. GitHub Actions calling ECS `UpdateService`) | The pipeline builds the image and ECS replaces old tasks with new ones a few at a time. | **Strength:** simple and cheap, nothing extra to run. **Weakness:** old and new versions serve customers at the same time, and rolling back means running another full deployment while customers are affected. |
| **2. Blue/green on Amazon ECS with AWS CodeDeploy** | The new version starts beside the old one, is checked on a private test port, then the load balancer moves traffic across. The old version stays on standby so traffic can be moved back. | **Strength:** no downtime, and rollback is a traffic switch measured in seconds, triggered automatically by alarms. **Weakness:** more moving parts (two target groups, test listener, pipeline), and two copies of the service run for a few minutes during each release. |
| **3. Kubernetes (Amazon EKS) with GitOps and progressive delivery** (Argo CD / Argo Rollouts) | A cluster pulls the desired version from Git and shifts traffic in steps. | **Strength:** very flexible, and the industry standard for large platforms. **Weakness:** a cluster fee plus a steep learning curve and ongoing upgrades. Far too heavy for one service and a small team. |

I also considered two other choices.
- **Network:** a NAT gateway gives private containers internet access but adds hourly and per-GB charges in each AZ. VPC endpoints keep traffic to AWS services private and need no internet route.
- **CI login:** stored IAM access keys live forever and can leak. OIDC federation issues credentials that expire within an hour and leaves no secret to steal.

### What happens with no solution

- **Large, rare releases carry more risk.** Every release bundles many changes. When one fails, it fails for every customer at once, and recovery is another hand-built release under pressure. In 2012 Knight Capital lost about **USD 440 million in 45 minutes** because a manual deployment missed one of eight servers and old code ran in production (SEC order). In July 2024 a faulty CrowdStrike update was sent to all customers at once instead of in stages. It crashed about 8.5 million Windows machines and grounded flights worldwide.
- **Lost orders and lost trust.** For Onward, a broken release means merchants can't book deliveries. That is direct lost revenue, and merchants may move to a competitor. The risk is highest on peak days, when releases are already avoided.
- **Leaked credentials.** Upload keys kept on laptops and in scripts can leak. In the 2021 Codecov breach, attackers changed a CI script and collected secrets stored in customers' CI environments. That kind of leak creates legal and data-protection exposure.
- **No audit trail.** Nobody can prove which code was running when an incident happened.

### Sources

1. SEC, *In the Matter of Knight Capital Americas LLC* (2013): https://www.sec.gov/litigation/admin/2013/34-70694.pdf
2. CrowdStrike, *Channel File 291 Incident Root Cause Analysis* (2024): https://www.crowdstrike.com/wp-content/uploads/2024/08/Channel-File-291-Incident-Root-Cause-Analysis-08.06.2024.pdf
3. Codecov, *Bash Uploader Security Update* (2021): https://about.codecov.io/security-update/
4. AWS, *Amazon ECS blue/green deployments with CodeDeploy*: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html
5. AWS Builders' Library, *Automating safe, hands-off deployments*: https://aws.amazon.com/builders-library/automating-safe-hands-off-deployments/
6. GitHub Docs, *Configuring OpenID Connect in Amazon Web Services*: https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
7. AWS, *CloudFormation Git sync*: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/git-sync.html
8. AWS pricing pages for Fargate, PrivateLink and NAT gateway: https://aws.amazon.com/fargate/pricing/, https://aws.amazon.com/privatelink/pricing/, https://aws.amazon.com/vpc/pricing/
9. DORA, *Software delivery performance metrics*: https://dora.dev/guides/dora-metrics-four-keys/
10. AWS Pricing Calculator estimate for this design: https://calculator.aws/#/addService

## 2. The business value of my solution

### My chosen approach

I chose **blue/green releases on Amazon ECS with AWS Fargate, run by AWS CodePipeline and AWS CodeDeploy**, with every piece of infrastructure written as code.

In plain terms, an engineer merges a reviewed change and the rest is automatic:
1. The change is tested and packaged.
2. The release owner gets an email and approves the release with one click.
3. The new version starts **next to** the old one and is health-checked before any customer reaches it.
4. Customers are switched across in one step.
5. The old version stays ready for **5 minutes**, and any rollback goes to an image already stored in the registry. If errors or unhealthy tasks appear, alarms switch customers back automatically, with no rebuild.

This suits Onward better than the alternatives.
- **Better than rolling updates:** customers never see a half-old, half-new service, and going back is a switch, not a redeploy.
- **Better than Kubernetes:** Onward gets the same safety without running or paying for a cluster.
- **Fargate:** there are no servers for the small team to patch.
- **Infrastructure as code:** the whole platform can be rebuilt from the repositories.

### Services I will use

| Service | The job it does | Why this service, used this way |
|---|---|---|
| **Amazon ECS on AWS Fargate** | Runs the order app as containers: 2 to 6 tasks of 0.25 vCPU and 0.5 GB each, spread over two Availability Zones | No servers to manage. Failed tasks are replaced automatically, and losing one AZ doesn't stop orders. |
| **Elastic Load Balancing (Application Load Balancer)** | The single public entry point. Health-checks tasks and holds the blue and green target groups plus a private test listener | Blue/green needs a load balancer that can move traffic between two target groups. One entry point also keeps the attack surface small. |
| **Application Auto Scaling** | Adds tasks when average CPU goes above 50% and removes them afterwards | Handles peak days without anyone changing capacity, and the business pays less on quiet days. |
| **Amazon ECR** | Private image registry. Scans each image for known vulnerabilities and keeps the 30 newest releases | Each release is tagged with its git commit, so we always know what is running. The 30 kept images form the rollback window. |
| **Amazon S3** | Stores the versioned deployment bundle (`taskdef.json` and `appspec.yaml`). Old versions expire after 30 days | CodePipeline needs a versioned source for the "how to run it" files. Runtime settings change without rebuilding the image. |
| **GitHub Actions + IAM OIDC** | Tests, builds and publishes each change merged to `main` | No stored AWS keys. Credentials expire within an hour and only work for this repository's `main` branch. |
| **Amazon EventBridge** | Starts the pipeline the moment a new image is published | Releases start on their own, with nothing to schedule or poll. |
| **AWS CodePipeline** | Runs Source → **Manual approval** → Deploy, and keeps a history of every release | Gives an auditable record, a human approval gate before production, and a one-click "Start rollback" to any earlier release. |
| **Amazon SNS** | Emails the release owner when a release is waiting for approval. The email links to the exact image | Nobody needs to watch the console, and the approver can see exactly what they're approving. |
| **AWS CodeDeploy** | Runs the blue/green switch (all at once, 5-minute standby) and rolls back automatically on alarms | Zero-downtime releases, with rollback in seconds instead of a rebuild. |
| **Amazon CloudWatch** | Logs, Container Insights, and alarms on 5xx errors and unhealthy tasks | The alarms are what make rollback automatic, and the logs help find the cause. |
| **VPC endpoints** (ECR API, ECR Docker, CloudWatch Logs; S3 gateway) | Give the private tasks a path to AWS services | Containers have no internet route, which meets Security's rule. It's also cheaper than a NAT gateway in each AZ. |
| **AWS CloudFormation + Git sync** | Defines all five stacks in code and redeploys a stack when its file changes on `main` | Every infrastructure change is reviewed in a pull request. There is no console drift, and the platform is repeatable. |

### Cost estimate

Estimated with the AWS Pricing Calculator for the `prod` environment in Europe (Ireland), at on-demand prices with no free tier:

| Service | What is priced | Monthly (USD) |
|---|---|---|
| Amazon VPC | 3 interface endpoints × 2 AZs, 5 GB processed; 2 public IPv4 addresses for the ALB | 55.53 |
| Elastic Load Balancing | 1 Application Load Balancer, low traffic | 19.57 |
| AWS Fargate | 2 tasks running 24/7, 0.25 vCPU / 0.5 GB, Linux x86 | 18.01 |
| Amazon CloudWatch | 15 metrics (Container Insights), 1 GB of logs, 5 standard alarms | 5.57 |
| Amazon ECR | 5 GB of images (30 releases kept) | 0.50 |
| AWS CodePipeline (V2) | 200 action minutes (about 20 releases); manual approval minutes aren't charged | 0.20 |
| Amazon S3 | 1 GB of deploy bundles and pipeline artifacts | 0.03 |
| **Total** | | **99.41 / month, 1,192.92 / year** |

These services cost nothing at this usage: AWS CodeDeploy (ECS deployments), Amazon EventBridge (AWS service events), Amazon SNS (under 1,000 emails a month), Application Auto Scaling, AWS CloudFormation, IAM and the OIDC provider, AWS-managed KMS keys, and the S3 gateway endpoint. GitHub Actions is billed by GitHub, not AWS.

What this means for the Finance Lead:
- **The private network is the largest cost, and it is cheaper than the alternative.** The interface endpoints cost about USD 48 a month. Two NAT gateways (one per AZ) would cost about USD 70 a month plus data charges, and would give the containers an internet route that Security does not allow.
- **Peak days add little.** Scaling from 2 to 6 tasks for a few hours costs cents, and the extra tasks are removed afterwards.
- **Ways to save:**
  - In `dev`, turn on `OffHoursShutdown` so the tasks stop out of hours, which cuts the Fargate line by about 70%. Or delete the dev stacks and rebuild them from Git when they are needed.
  - A Compute Savings Plan lowers the Fargate cost for steady production use.
- **What isn't included:** taxes, AWS Support plans, a future database, and internet data transfer above the free 100 GB a month.

### Who benefits, and how I will know

| Stakeholder | Benefit | How I would know |
|---|---|---|
| **Chief Technology Officer** | Releases become routine. Merging and one approval click are the only manual steps | Release frequency goes up. Time from merge to live is about **10 minutes** (measured in CodePipeline history) |
| **Head of Security** | No long-lived keys anywhere. Containers have no internet route. One public entry point. Least-privilege roles | **Zero** IAM access keys in the account for CI. No NAT gateway or internet route in the private route tables |
| **Head of Operations** | Releases cause no downtime. Bad releases undo themselves. The service grows on busy days | `/health` responds throughout every release. Rollback completes in **under 5 minutes**. Task count rises during peaks |
| **Finance Lead** | A predictable bill of about **USD 99 a month** for production. Pay only for the capacity used. No NAT gateway charges. Dev can be turned off out of hours | The Cost Explorer total for the `Project=onward` tag stays close to the USD 99 estimate |
| **Merchants and customers** | Booking stays available during releases and peak days, and fixes arrive sooner | No 5xx spikes during releases. Fewer support tickets about the booking page |
| **Engineers** | No laptop builds. Small, low-stress releases with a clear way back | Change failure rate and time to restore service (DORA metrics) both go down |

### Limits and risks of my design

- **Database changes aren't covered.** Blue and green run side by side, so any future database schema change must work with both versions. The current app keeps orders in memory per task, so orders are lost when a task is replaced. A real launch needs a database such as Amazon RDS or DynamoDB.
- **The rollback window is limited.** You can roll back to any of the last 30 images. Older bundles expire after 30 days. Automatic rollback only works within the 5-minute standby; after that, rollback is a new release.
- **Alarms only catch what they measure.** A release that returns wrong prices without errors won't trigger a rollback. Better tests, and later a business metric alarm, would reduce this.
- **One person approves releases.** If the release owner is unavailable, releases wait. The approver group should include a second person.
- **A single Region.** The design survives the loss of one AZ, but not an outage of the whole `eu-west-1` Region.
- **It depends on GitHub.** If GitHub Actions is down, no new releases can be built. Production keeps running and console rollbacks still work.
- **Manual console changes cause drift.** Editing the pipeline or stacks by hand fights Git sync, as when duplicate triggers once caused double approvals. All changes must go through pull requests.
