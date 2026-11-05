# Architecture

GitHub renders the diagrams below. Export them to PNG/SVG from https://mermaid.live if you need an image file.

## Resources inside the VPC

```mermaid
flowchart TB
  user([Merchants and customers]) -->|HTTP :80| alb
  dev([Engineer, demo IP only]) -.->|HTTP :8080 test listener| alb

  subgraph region[AWS Region]
    subgraph vpc[VPC 10.20.0.0/16 - onward-prod-vpc]
      igw[Internet gateway]
      subgraph azA[Availability Zone A]
        pubA[Public subnet 10.20.0.0/24]
        subgraph privA[Private subnet 10.20.10.0/24 - no internet route]
          taskA1[Fargate task<br/>order-app]
          vpceA[Endpoint ENIs<br/>ecr.api, ecr.dkr, logs]
        end
      end
      subgraph azB[Availability Zone B]
        pubB[Public subnet 10.20.1.0/24]
        subgraph privB[Private subnet 10.20.11.0/24 - no internet route]
          taskB1[Fargate task<br/>order-app]
          vpceB[Endpoint ENIs<br/>ecr.api, ecr.dkr, logs]
        end
      end
      alb[[Application Load Balancer<br/>internet-facing, in both public subnets<br/>blue + green target groups]]
      s3gw[(S3 gateway endpoint<br/>ECR layer bucket only)]
    end
    ecr[(Amazon ECR)]
    cwl[(CloudWatch Logs)]
    s3[(S3 - ECR image layers)]
  end

  igw --- alb
  alb -->|:8000 app-sg only| taskA1
  alb -->|:8000 app-sg only| taskB1
  taskA1 -->|443| vpceA
  taskB1 -->|443| vpceB
  vpceA & vpceB -->|AWS network| ecr
  vpceA & vpceB -->|AWS network| cwl
  taskA1 & taskB1 -->|route table prefix list| s3gw --> s3
```

Security groups: `alb-sg` (80 from anywhere, 8080 from the demo IP) -> `app-sg` (8000 from `alb-sg` only; egress 443 to `vpce-sg` and S3 only) -> `vpce-sg` (443 from `app-sg` only). The private route table holds only the `local` route and the S3 gateway route; there is no NAT gateway and no `0.0.0.0/0` route.

## Delivery pipeline: how a change flows

```mermaid
flowchart LR
  pr[Pull request merged<br/>to main<br/><i>the only manual step</i>] --> gha

  subgraph gh[GitHub - onward-order-app repo]
    gha[GitHub Actions<br/>test, build, smoke test]
  end

  gha -->|OIDC token| sts[AWS STS<br/>short-lived credentials<br/>role: onward-prod-github-actions]
  gha -->|1. push image :git-sha| ecr[(ECR)]
  gha -->|2. upload taskdef.json + appspec.yaml| s3[(S3 deploy-bundles)]
  gha -->|3. push image :latest| ecr

  ecr -->|ECR Image Action PUSH latest| eb{{EventBridge rule}}
  eb -->|StartPipelineExecution| cp

  subgraph cp[CodePipeline onward-prod-order-release]
    src1[Source: ECR image] --> dep
    src2[Source: S3 bundle] --> dep
    dep[Deploy: CodeDeployToECS]
  end

  dep --> cd[CodeDeploy blue/green]
  cd -->|start green tasks| green[Green tasks<br/>new version]
  cd -->|10% for 5 min, then 100%| alb[[ALB prod listener]]
  alarms[[CloudWatch alarms<br/>5xx, unhealthy hosts]] -.->|alarm = automatic rollback| cd
  cd -->|after 15 min| blue[Blue tasks<br/>old version terminated]
```

## Infrastructure delivery (GitSync)

```mermaid
flowchart LR
  infra[onward-order-infra repo<br/>templates/ + deployments/prod/] -->|CodeConnections| gs[CloudFormation GitSync]
  gs --> n[onward-prod-network-Lab3-A]
  gs --> f[onward-prod-delivery-foundation-Lab3-A]
  gs --> s[onward-prod-service-Lab3-A]
  gs --> p[onward-prod-pipeline-Lab3-A]
  n -. exports .-> s
  f -. exports .-> s
  f -. exports .-> p
  s -. exports .-> p
```
