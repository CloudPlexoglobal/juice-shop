# Cyber Sheriff's Juice Shop

This is a fork of [OWASP Juice Shop](https://github.com/juice-shop/juice-shop), kept on the
`cybersheriff` branch at **v17.1.1**. It's deliberately vulnerable, and serves three uses:

- **Demos.** It runs at `https://juice.cybersheriff.ai`, a target we own that anyone can watch
  Cyber Sheriff scan.
- **The agent benchmark.** The answer key in Cyber Sheriff's
  `backend/src/benchmark/suites/juice_shop.toml` is written for exactly this version.
- **Code scanning.** Connect this repository to Cyber Sheriff and run a code security scan on the
  `cybersheriff` branch.

Stay on v17.1.1 unless you also update the benchmark's answer key. A newer Juice Shop changes what
there is to find.

## How it is fenced off

`cybersheriff/stack.yaml` keeps the demo away from everything else:
- **Network:** its own subnets (`10.0.200.0/24`, `10.0.201.0/24`) and route table in
  `aws-default-vpc`. The account is at its VPC quota, so the demo shares that VPC and the security
  groups do the fencing.
- **Load balancer:** its own, HTTPS only.
- **Container:** only that load balancer can reach it, and it can send nothing out but HTTPS, which
  it needs to pull its image.
- **No IAM task role.** Juice Shop has SSRF-style challenges, and without a role there are no AWS
  credentials for a visitor to reach.

## Deploying

The workflow `.github/workflows/cybersheriff-deploy.yml` runs on every push to `cybersheriff`. It
builds this branch into an image, pushes it to ECR, and updates the `juice-shop-demo` stack.

It assumes a narrow role, which can only:
- push that one image;
- ask CloudFormation to update that one stack, through the CloudFormation role.

The CloudFormation role holds the infrastructure permissions.

### Once, before the first deploy

Deploy the bootstrap stack (the ECR repository and the two roles) with an administrator's session:

```bash
aws cloudformation deploy --region eu-west-1 --stack-name juice-shop-demo-bootstrap \
  --template-file cybersheriff/bootstrap.yaml --capabilities CAPABILITY_NAMED_IAM
```

Then enable GitHub Actions on this fork, and run **Deploy the Cyber Sheriff demo**.

The DNS zone for `cybersheriff.ai` lives in another AWS account. Add a CNAME there, from
`juice.cybersheriff.ai` to the stack's `LoadBalancerDnsName` output.

## Resetting it

Visitors can change the shop: they register users, solve challenges, and post reviews. Juice Shop
starts fresh whenever its container restarts. To get a clean shop before a demo:

```bash
aws ecs update-service --region eu-west-1 --cluster juice-shop-demo \
  --service juice-shop-demo --force-new-deployment
```

## Upstream workflows

This branch drops upstream's workflows (CI, CodeQL, ZAP, releases and others), which would
otherwise run on every push here. Only `cybersheriff-deploy.yml` remains. The app's code is
unchanged from v17.1.1.

The Dockerfile's build stages use `node:20-bullseye` instead of upstream's `node:20-buster`:
Buster's package servers are archived, so `apt-get update` fails there. Bullseye is the Debian
release the final distroless image (`nodejs20-debian11`) runs on, so the native `libxmljs`
module built in it still matches at runtime.

The Dockerfile also pins `@cyclonedx/cyclonedx-npm` to 1.19.3 instead of `latest`. The 4.x line
dropped the bundled XML writer, so `npm run sbom` fails on it.
