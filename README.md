# AWS-Onward-Order-Infra-Lab3-A

Infrastructure for the Onward Logistics order system, written in AWS CloudFormation and deployed by **CloudFormation GitSync** from this repository. Nothing is created by hand in the console. The application code lives in a separate repository, [`onward-order-app`](../onward-order-app).

See [docs/architecture.md](docs/architecture.md) for the diagrams and [docs/discovery-report.md](docs/discovery-report.md) for the discovery work.

## How the pieces fit together

| Stack | Template | What it owns | Why it is separate |
|---|---|---|---|
| `onward-prod-gitsync-bootstrap` | `templates/bootstrap-gitsync.yaml` | GitHub connection, repository link, GitSync role, one sync configuration per stack | GitSync cannot create itself; deployed once from the CLI |
| `onward-prod-network` | `templates/network.yaml` | VPC, 2 public + 2 private subnets in 2 AZs, route tables, **all security groups**, VPC endpoints | Changes rarely; all traffic rules readable in one file |
| `onward-prod-delivery-foundation` | `templates/delivery-foundation.yaml` | ECR repository, deploy-bundle bucket, GitHub OIDC provider and role | Must exist before the service so an image can be pushed first |
| `onward-prod-service` | `templates/service.yaml` | ALB, blue/green target groups, ECS cluster, task definition, service, auto scaling, alarms | The running application |
| `onward-prod-pipeline` | `templates/pipeline.yaml` | CodeDeploy app + deployment group, CodePipeline, EventBridge rule | Release machinery; can be rebuilt without touching the app |

Stacks pass values with CloudFormation exports named `<project>-<env>-<Name>` (for example `onward-prod-VpcId`). Each file in `deployments/prod/` is a GitSync deployment file: it names the template, its parameters and the tags applied to every resource.

**Naming:** `<project>-<environment>-<component>`, e.g. `onward-prod-alb`. **Tags on every resource:** `Project`, `Environment`, `Owner`, `CostCenter`, `ManagedBy`. Align these with the Client Profile's standards if they differ.

**IAM:** no inline policies. Roles use AWS-managed policies or `AWS::IAM::ManagedPolicy` resources. The `validate` workflow fails any pull request that adds an inline policy.

## Build it from scratch

Prerequisites: AWS CLI v2 with an admin profile, two GitHub repositories (`onward-order-infra`, `onward-order-app`), and the `REPLACE-…` values in `deployments/prod/*.yaml` filled in.

1. **Find your Region's S3 prefix list** and put it in `deployments/prod/network.yaml`:
   ```bash
   aws ec2 describe-managed-prefix-lists \
     --filters Name=prefix-list-name,Values=com.amazonaws.$AWS_REGION.s3 \
     --query 'PrefixLists[0].PrefixListId' --output text
   ```
   Commit and push.

2. **Create the GitHub connection (stage 0):**
   ```bash
   aws cloudformation deploy --stack-name onward-prod-gitsync-bootstrap \
     --template-file templates/bootstrap-gitsync.yaml --capabilities CAPABILITY_NAMED_IAM \
     --parameter-overrides Owner=<you> CostCenter=<cc> GitHubOwner=<github-user> SyncStage=0
   ```
   The connection is created in `PENDING` state. AWS requires a person to authorise it once in GitHub: open *Developer Tools > Settings > Connections*, choose the connection and select *Update pending connection*. This is an OAuth handshake, not resource creation.

3. **Turn on GitSync one stack at a time.** Re-run the command above with `SyncStage=1`, wait for `onward-prod-network` to reach `CREATE_COMPLETE`, then `SyncStage=2` and wait for `onward-prod-delivery-foundation`.

4. **Publish the first image.** In the app repository set two GitHub *variables* (not secrets; neither is a credential): `AWS_REGION` and `AWS_ROLE_ARN` (output `GitHubActionsRoleArn` of the foundation stack). `ECR_REPOSITORY`, `TASK_FAMILY` and `DEPLOY_BUCKET` default to the names these stacks create and only need setting to override them. Run the *Build and release* workflow. It pushes the image and skips the bundle because no task definition exists yet.

   **Reusing an existing GitHub Actions role.** Set `ExistingGitHubActionsRoleName` in `deployments/prod/delivery-foundation.yaml` (this repository uses `github-actions-ecr-push`). The stack then attaches the `onward-prod-github-publish` policy to that role instead of creating `onward-prod-github-actions`. The role's trust policy is not managed by this stack: it must allow `repo:<owner>/onward-order-app:ref:refs/heads/main` through the GitHub OIDC provider.

5. **`SyncStage=3`** creates the service from that image. **`SyncStage=4`** creates the pipeline. Its first automatic run fails because there is no bundle yet; that is expected.

6. **Run *Build and release* again.** From now on every merge to `main` releases automatically.

From here, infrastructure changes are made by pull request to this repository; GitSync applies them when they merge.

### Things to know before changing the service stack

With `DeploymentController: CODE_DEPLOY`, CodeDeploy owns the running task definition, the listener rules and which target group is live. CloudFormation cannot update those on this service. To change CPU, memory or environment variables, change them in `service.yaml` (for new environments), and for the live service let the next release carry them: the app repository builds each release's task definition from the latest registered revision. Larger changes (subnets, load balancer wiring) mean replacing the service.

## Demonstrating the client's checks

Set `TestTrafficCidr` to `<your-ip>/32` before the review if you want to show the test listener.

| Client check | How to show it |
|---|---|
| One public address; containers in private subnets with no route to the internet | Open the `ApplicationUrl` output. In VPC > Route tables show `onward-prod-private-rt`: only `local` and the S3 prefix list, no `0.0.0.0/0`, no NAT gateway. Show tasks have no public IP (ECS > task > Networking). Show `app-sg` ingress is `alb-sg` only. |
| Push a change, new version runs, no manual steps | Change the page in the app repo, merge. Watch GitHub Actions > EventBridge > CodePipeline > CodeDeploy. The *Release* field on the page changes to the new commit SHA. |
| No downtime; bad release reversible | Run `while true; do curl -s -o /dev/null -w "%{http_code}\n" $URL; sleep 0.5; done` during a release: only `200`s. For a bad release, make `home()` raise an exception and merge. The canary sends 10% of traffic, the 5xx alarm fires, CodeDeploy rolls back automatically. Also show *Stop and roll back* in CodeDeploy and the *Roll back to a previous release* workflow. |
| No long-lived credentials | Search both repos for `AKIA` / `aws_secret` (nothing). Show the workflow uses `id-token: write` + `role-to-assume`. Show the role's trust policy is limited to `repo:<owner>/onward-order-app:ref:refs/heads/main`. IAM > Users: no access keys used by the pipeline. |
| Adds containers under load, removes them afterwards | `./scripts/load-test.sh $URL 8` in the app repo. Watch desired count rise from 2 toward 6 (scale-out cooldown 60 s), then fall back to 2 after load stops (scale-in cooldown 180 s). |
| Survives a container or AZ failure | Container: `aws ecs stop-task` on one task; the other keeps serving and ECS starts a replacement. AZ: the page header shows which task and AZ served you. Stop every task in one AZ (`aws ecs list-tasks`, then `describe-tasks` to filter by `availabilityZone`) and show the site keeps answering from the other AZ while ECS starts replacements. For a true AZ outage test in production, AWS Fault Injection Service has an AZ power-interruption scenario. |
| A new engineer can understand it | This README, the comments at the top of every template, `docs/architecture.md`, and the app repo README. |
| Explain cost and rejected alternatives | Sections below. |
| Rebuild everything from the repositories | Delete the stacks in reverse order (pipeline, service, foundation, network) after emptying the two S3 buckets and ECR repository, then follow *Build it from scratch*. |

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
