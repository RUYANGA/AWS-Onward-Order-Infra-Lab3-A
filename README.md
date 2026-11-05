# AWS-Onward-Order-Infra-Lab3-A

Infrastructure for the Onward Logistics order system, written in AWS CloudFormation and deployed by **CloudFormation GitSync** from this repository. Every resource is defined here; the stacks are deployed from Git, never edited by hand. The application code lives in a separate repository, [`AWS-Onward-Order-App-Lab3-A`](https://github.com/RUYANGA/AWS-Onward-Order-App-Lab3-A).

See [docs/architecture.md](docs/architecture.md) for the diagrams and [docs/discovery-report.md](docs/discovery-report.md) for the discovery work.

## How the pieces fit together

| Stack (live name) | Template | What it owns | Why it is separate |
|---|---|---|---|
| *GitSync setup* | `templates/bootstrap-gitsync.yaml` | Repository link, the deploy role `onward-prod-App-cloudformation-deploy`, one sync configuration per stack | GitSync cannot create itself; set up once |
| `Onward-prod-network-App-Lab3-A` | `templates/network.yaml` | VPC, 2 public + 2 private subnets in 2 AZs, route tables, **all security groups**, VPC endpoints | Changes rarely; all traffic rules readable in one file |
| `onward-prod-delivery-foundation-App-Lab3-A` | `templates/delivery-foundation.yaml` | ECR repository, deploy-bundle bucket, GitHub Actions role (OIDC) and publish policy | Must exist before the service so an image can be pushed first |
| `Onward-prod-service-Lab3-A` | `templates/service.yaml` | ALB, blue/green target groups, ECS cluster, task definition, service, auto scaling, alarms | The running application |
| `Onward-prod-pipeline-Lab3-A` | `templates/pipeline.yaml` | CodeDeploy app + deployment group, CodePipeline, EventBridge rule | Release machinery; can be rebuilt without touching the app |

Region: **eu-west-1**. Application URL: output `ApplicationUrl` of the service stack.

Stacks pass values with CloudFormation exports named `<project>-<env>-<Name>` (for example `onward-prod-VpcId`). Each file in `deployments/prod/` is a GitSync deployment file: it names the template, its parameters and the tags applied to every resource.

**Naming:** `<project>-<environment>-<component>`, e.g. `onward-prod-alb`. **Tags on every resource:** `Project`, `Environment`, `Owner`, `ManagedBy`. Align these with the Client Profile's standards if they differ.

**IAM:** no inline policies. Roles use AWS-managed policies or `AWS::IAM::ManagedPolicy` resources. The `validate` workflow fails any pull request that adds an inline policy.

## How it is deployed

GitSync watches `main` of this repository. Each stack has a sync configuration that points at its deployment file in `deployments/prod/`; a merge to `main` that changes a template or deployment file updates the matching stack.

| Sync configuration (stack) | Deployment file | Role | Trigger |
|---|---|---|---|
| `Onward-prod-network-App-Lab3-A` | `deployments/prod/network.yaml` | `onward-prod-App-cloudformation-deploy` | any change on `main` |
| `onward-prod-delivery-foundation-App-Lab3-A` | `deployments/prod/delivery-foundation.yaml` | same | any change on `main` |
| `Onward-prod-service-Lab3-A` | `deployments/prod/service.yaml` | same | any change on `main` |
| `Onward-prod-pipeline-Lab3-A` | `deployments/prod/pipeline.yaml` | same | any change on `main` |

**Deploy role** `onward-prod-App-cloudformation-deploy`: trusted by `cloudformation.amazonaws.com` (builds the resources) and `cloudformation.sync.codeconnections.amazonaws.com` (starts each sync), with `PowerUserAccess` and `IAMFullAccess` (the stacks create IAM roles). Tighten to service-specific policies for real production.

**GitHub Actions role** `onward-prod-github-actions` (created by the foundation stack): only `RUYANGA/AWS-Onward-Order-App-Lab3-A` on `main` can assume it, through the account's existing GitHub OIDC provider. Both subject formats are trusted: classic `repo:<owner>/<repo>:ref:refs/heads/main` and GitHub's immutable `repo:<owner>@<id>/<repo>@<id>:ref:refs/heads/main`.

## Build it from scratch

Prerequisites: two GitHub repositories (this one and the app), an **Available** GitHub connection in *Developer Tools > Settings > Connections* with access to this repository, and the console in **eu-west-1**.

1. **S3 prefix list.** `deployments/prod/network.yaml` uses `pl-6da54004` (the S3 prefix list for eu-west-1). For another Region use the ID shown under *VPC > Managed prefix lists* for `com.amazonaws.<region>.s3`.

2. **GitSync setup.** *CloudFormation > Create stack > With new resources > Upload a template file* with `templates/bootstrap-gitsync.yaml`. Its defaults reproduce the live setup (stack names, deploy role, connection and repository link). Enter `GitHubOwner = RUYANGA`, keep `SyncStage = 1`, acknowledge IAM resources with custom names and submit. In a fresh account set `ExistingConnectionArn` and `ExistingRepositoryLinkId` to empty values to create them (a new connection must be authorised once in the console).

   *Equivalent console route, used for the live environment:* *Create stack > Sync from Git*, once per stack in the order below, choosing the linked repository `AWS-Onward-Order-Infra-Lab3-A`, branch `main`, "I am providing my own file" with the deployment file path from the table above, and `onward-prod-App-cloudformation-deploy` as the IAM role.

3. **Network, then foundation.** Wait for the network stack to complete, then raise `SyncStage` to `2` (*Update stack > Use existing template*) and wait for the foundation stack. Stacks import each other's outputs, so a stack created before the one it depends on fails with `No export named onward-prod-… found`; retry it from its *Git sync* tab once the dependency is complete.

4. **Publish the first image.** In the app repository set two GitHub *variables* (not secrets): `AWS_REGION = eu-west-1` and `AWS_ROLE_ARN` = output `GitHubActionsRoleArn` of the foundation stack. Run *Build and release*: it pushes the image and skips the deployment bundle because no task definition exists yet.

5. **Service, then pipeline.** `SyncStage = 3` creates the service from that image (about 5 minutes, until both tasks are healthy); `SyncStage = 4` creates the pipeline.

6. **Run *Build and release* again.** It uploads the deployment bundle and moves `latest`, and CodePipeline runs the first blue/green release. From then on every merge to the app's `main` releases automatically.

Infrastructure changes are made by pull request to this repository; CI validates them and GitSync applies them when they merge.

### Things to know before changing the service stack

With `DeploymentController: CODE_DEPLOY`, CodeDeploy owns the running task definition, the listener rules and which target group is live. CloudFormation cannot update those on this service, so **drift detection on the service stack always reports `Service`, `ProductionListener` and `TestListener` as modified** (live task definition revision and active blue or green target group). That drift is expected and alternates with every release; the network, foundation and pipeline stacks should always be `IN_SYNC`. To change CPU, memory or environment variables, change them in `service.yaml` (for new environments), and for the live service let the next release carry them: the app repository builds each release's task definition from the latest registered revision. Larger changes (subnets, load balancer wiring) mean replacing the service.

## Demonstrating the client's checks

Set `TestTrafficCidr` to `<your-ip>/32` before the review if you want to show the test listener.

| Client check | How to show it |
|---|---|
| One public address; containers in private subnets with no route to the internet | Open the `ApplicationUrl` output. In VPC > Route tables show `onward-prod-private-rt`: only `local` and the S3 prefix list, no `0.0.0.0/0`, no NAT gateway. Show tasks have no public IP (ECS > task > Networking). Show `app-sg` ingress is `alb-sg` only. |
| Push a change, new version runs, no manual steps | Change the page in the app repo, merge. Watch GitHub Actions > EventBridge > CodePipeline > CodeDeploy. The *Release* field on the page changes to the new commit SHA. |
| No downtime; bad release reversible | Run `while true; do curl -s -o /dev/null -w "%{http_code}\n" $URL; sleep 0.5; done` during a release: only `200`s. For a bad release, make `home()` raise an exception and merge. The canary sends 10% of traffic, the 5xx alarm fires, CodeDeploy rolls back automatically. Also show *Stop and roll back* in CodeDeploy and the *Roll back to a previous release* workflow. |
| No long-lived credentials | Search both repos for `AKIA` / `aws_secret` (nothing). Show the workflow uses `id-token: write` + `role-to-assume`. Show the role's trust policy is limited to `repo:RUYANGA/AWS-Onward-Order-App-Lab3-A:ref:refs/heads/main (or the immutable ID form)`. IAM > Users: no access keys used by the pipeline. |
| Adds containers under load, removes them afterwards | `./scripts/load-test.sh $URL 8` in the app repo. Watch desired count rise from 2 toward 6 (scale-out cooldown 60 s), then fall back to 2 after load stops (scale-in cooldown 180 s). |
| Survives a container or AZ failure | Container: `aws ecs stop-task` on one task; the other keeps serving and ECS starts a replacement. AZ: the page header shows which task and AZ served you. Stop every task in one AZ (`aws ecs list-tasks`, then `describe-tasks` to filter by `availabilityZone`) and show the site keeps answering from the other AZ while ECS starts replacements. For a true AZ outage test in production, AWS Fault Injection Service has an AZ power-interruption scenario. |
| A new engineer can understand it | This README, the comments at the top of every template, `docs/architecture.md`, and the app repo README. |
| Explain cost and rejected alternatives | Sections below. |
| Rebuild everything from the repositories | Delete the stacks in reverse order (pipeline, service, foundation, network) after emptying the two S3 buckets and the ECR repository, then the GitSync setup, then follow *Build it from scratch*. |

## What it costs

Approximate on-demand prices in us-east-1, 730 hours per month. Prices vary by Region, so confirm with the [AWS Pricing Calculator](https://calculator.aws/) for the Region you deploy to.

| Item | Calculation | Per month |
|---|---|---|
| Interface endpoints (ecr.api, ecr.dkr, logs) | 3 endpoints x 2 AZs x $0.01/h | ~$43.80 + $0.01/GB processed |
| S3 gateway endpoint | free | $0 |
| Application Load Balancer | $0.0225/h + LCUs | ~$16.40 + usage |
| ALB public IPv4 addresses | 2 x $0.005/h | ~$7.30 |
| Fargate, 2 tasks at 0.25 vCPU / 0.5 GB | 2 x ($0.04048 x 0.25 + $0.004445 x 0.5) per hour | ~$18.00 (scales with load) |
| CloudWatch Logs, Container Insights, alarms | low volume | a few dollars |
| ECR storage (30 images max) | $0.10/GB-month | under $1 |
| CodePipeline V2 | $0.002 per action-minute | cents |
| CodeDeploy to ECS | free | $0 |
| **Baseline** | | **roughly $85-95** |

**Private connectivity versus a NAT gateway.** A NAT gateway per AZ (needed to keep AZ resilience) costs 2 x $0.045/h = ~$65.70/month plus $0.045/GB processed. The three interface endpoints cost ~$43.80 plus $0.01/GB. Endpoints are cheaper at our scale, charge less per GB on image pulls, keep traffic on the AWS network, and give the tasks no path to the internet at all, which is what the Head of Security asked for. Every extra AWS service the app needs later (for example Secrets Manager) adds ~$14.60/month for another endpoint, which is the trade-off to watch.

**Idle environments.** A `dev` environment uses the same templates with `Environment: dev` and `OffHoursShutdown: "true"`, which scales the service to zero at 18:00 and back at 06:00 on weekdays (UTC). Endpoints and the ALB still charge while idle, so an unused environment should be deleted; because everything is in these repositories it can be rebuilt in about 20 minutes.

## Design decisions and rejected alternatives

| Decision | Chosen | Rejected and why |
|---|---|---|
| Outbound connectivity | VPC endpoints | NAT gateway: costs more, opens a path to the internet |
| Compute | ECS on Fargate | EC2 launch type: servers to patch and scale. EKS: cluster fee and much more to learn for one service. App Runner: no CodeDeploy blue/green |
| Release strategy | CodeDeploy blue/green, canary 10% for 5 min, auto rollback on alarms, old version kept 15 min | Rolling update: old and new run mixed and rollback is a new deployment. All-at-once blue/green: faster but every customer meets a bad release |
| Hand-off to CodeDeploy | GitHub Actions uploads `taskdef.json` (rendered from the live revision) and `appspec.yaml` to S3; CodePipeline takes the image digest from the ECR event | Task definition JSON in the app repo: would duplicate role ARNs and sizes owned by the infra repo |
| CI credentials | GitHub OIDC, role restricted to one repo and branch | IAM user access keys in GitHub secrets: long-lived, the exact thing Security forbade |
| Scaling signal | Target tracking on 50% average CPU, 2 to 6 tasks | Request-count-per-target: tied to one target group, which changes every blue/green release |
| Stack layout | Four layered stacks | One large stack: slower, riskier changes, harder for a new engineer to read |
| HTTPS | Not enabled (no domain supplied for this lab) | Production should add an ACM certificate, an HTTPS listener and redirect port 80 |
