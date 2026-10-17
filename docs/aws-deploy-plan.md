# ZunZun ToDo — AWS Deployment Plan (Days 13–14)

**Prerequisite:** Days 1–12 of the [implementation plan](implementation-plan.md) are complete — every feature works locally and all tests are green.
**Topology:** frontend (Next.js) on Vercel, function region `cdg1`; backend (Spring Boot) on AWS **eu-west-3 (Paris)**. All user data stays in the EU.

> **Security note:** this document is public. Use placeholders such as `<account-id>` and `<ecr-url>`.
> Never record real account IDs, endpoints, IP addresses, or credentials here.

---

## Principles

1. **Do it once by hand, verify each layer, then automate.** Image pushed → database reachable → container healthy → traffic through the load balancer.
2. **Watch costs.** Set a budget alert before creating anything; use the smallest sizes and a single Availability Zone for the database.
3. **No secrets in code or images.** Database credentials, the JWT signing key, and the Google client secret live in Secrets Manager and are injected into the task at start-up.
4. **Least privilege.** The ECS task role can read its secrets and read/write only the image bucket.

### Services

| Service | Role |
|---|---|
| ECR | Docker image registry |
| ECS Fargate | Runs the API container |
| RDS for PostgreSQL 17 | Production database (private subnet) |
| S3 | Profile images (private bucket, presigned URLs) |
| ALB + ACM | HTTPS entry point and certificate |
| Secrets Manager | Credentials and keys |
| CloudWatch Logs | Application logs |
| Route 53 (or your registrar's DNS) | `api.zunzuntodo.app` |

---

## Day 13 — Infrastructure

### 1. Account and image

- **Create a budget alert** (for example, notify above €40 per month)
- Use an IAM user or IAM Identity Center for daily work, never the root account
- Select region **eu-west-3** and use it for everything
- Production `Dockerfile` (multi-stage):
  - build stage: `eclipse-temurin:25-jdk` running `./mvnw -B package -DskipTests`
  - runtime stage: `eclipse-temurin:25-jre`, non-root user, `EXPOSE 8080`
- Verify locally with `docker build` and `docker run`
- Create the ECR repository `zunzuntodo-backend` and push:

```bash
aws ecr get-login-password --region eu-west-3 | docker login --username AWS --password-stdin <ecr-url>
docker build -t zunzuntodo-backend ./backend
docker tag zunzuntodo-backend:latest <ecr-url>/zunzuntodo-backend:latest
docker push <ecr-url>/zunzuntodo-backend:latest
```

### 2. Network and data

- VPC with public subnets (ALB) and private subnets (ECS tasks, RDS) in two Availability Zones
- Security groups: internet → ALB `443`; ALB → ECS `8080`; ECS → RDS `5432`. Nothing else
- RDS PostgreSQL 17: `db.t4g.micro`, single-AZ, 20 GB gp3, automated backups on, not publicly accessible
- S3 bucket for images: Block Public Access **on**, default encryption, no public policy
- Secrets Manager: database URL/user/password, JWT signing key, Google client ID and secret
- `application-prod.yml` reads every value from environment variables; `S3ImageStorage` enabled by the `prod` profile

### 3. Service

- ECS cluster (Fargate); task definition with 0.25 vCPU / 0.5–1 GB memory, secrets injected from Secrets Manager, logs to CloudWatch, `SPRING_PROFILES_ACTIVE=prod`
- Task role: `secretsmanager:GetSecretValue` on the app's secrets; `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the image bucket
- ALB with a target group on port 8080, health check `/actuator/health`, generous grace period (Spring Boot needs time to start)
- ECS service with one task in the private subnets (NAT gateway or VPC endpoints for ECR, S3, Secrets Manager, and CloudWatch Logs so the task can pull its image and read its secrets)
- Flyway runs on start-up and migrates the production database — check the logs

**Common failures:** the task stops immediately (secrets not readable → check the task execution role), the database is unreachable (security groups), or the health check fails before start-up finishes (raise the grace period). **Read CloudWatch Logs first.**

**Done when:** `https://<alb-dns-name>/actuator/health` returns UP.

---

## Day 14 — Release

### 4. Domain and HTTPS

- `zunzuntodo.app` → Vercel (production)
- `api.zunzuntodo.app` → ALB; ACM certificate in eu-west-3 attached to the HTTPS listener; HTTP redirects to HTTPS
- Sharing one registrable domain keeps the refresh-token cookie same-site (SDS §5.5, decision 5)

### 5. Application configuration

- CORS and the `Origin` allowlist: `https://zunzuntodo.app`
- Google Cloud Console: add `https://api.zunzuntodo.app/login/oauth2/code/google` as an authorized redirect URI
- Vercel: `NEXT_PUBLIC_API_URL=https://api.zunzuntodo.app/api/v1`, function region `cdg1`

### 6. Automated deployment

- GitHub Actions workflow `deploy-backend.yml`: on push to `main` touching `backend/**` → `./mvnw verify` → build image → push to ECR → update the ECS service
- Authenticate with **GitHub OIDC** and an IAM role scoped to this repository — no long-lived access keys

### 7. Release check

- Sign up (password and Google), create a project, record progress, view both charts, follow a second account, complete a public project, react, comment
- Load demo data; add the production URL to the README and the repository's About box
- Review Cost Explorer

**Done when:** the production site works end to end → tag `v1.0.0`.

---

## Cost Notes

| Item | Rough monthly cost |
|---|---|
| ALB | the largest fixed cost — runs whenever the environment is up |
| Fargate (0.25 vCPU, 0.5 GB, one task) | small |
| RDS `db.t4g.micro` | small; may be covered by the AWS Free Tier for eligible accounts |
| NAT gateway | significant — prefer VPC endpoints, or place the task in a public subnet with a public IP and a strict security group for a portfolio setup |
| S3, Secrets Manager, CloudWatch | minimal at this scale |

Check current prices in the AWS Pricing Calculator for eu-west-3. For a portfolio, stopping the environment when it is not being reviewed (scale the service to 0, stop RDS) keeps costs low.

- [ ] Budget alert created first
- [ ] Smallest sizes, single-AZ database
- [ ] No unused resources left after each work block
- [ ] NAT gateway avoided or justified

---

## After Release

Rename this document to `docs/deployment.md` and rewrite it as a runbook: how the system is deployed today, how to redeploy, roll back, rotate secrets, and troubleshoot.
