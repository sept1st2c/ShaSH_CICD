---

# Complete CI/CD with Terraform and AWS

This project is a small but realistic CI/CD setup that takes a simple Node.js service, builds it into a Docker image, provisions AWS infrastructure with Terraform, and deploys the container on an EC2 instance using GitHub Actions. It is meant to be easy to understand, reproducible from scratch, and reusable for other apps with minimal changes.[^1][^2]

---

## Technologies

- Terraform
- GitHub Actions
- Docker
- Node.js
- React (for optional frontend)
- Express
- AWS EC2
- AWS S3 (remote Terraform state)
- AWS ECR (container registry)[^3][^2]

---

## Architecture overview

At a high level, the pipeline looks like this:

1. Code lives in this GitHub repository (Node.js app + Terraform + workflow files).
2. On a push to the repository, GitHub Actions runs a workflow.
3. The workflow uses Terraform to provision or update AWS infrastructure (EC2 instance, SSH keys, security group, etc.), storing state remotely in S3.
4. The same workflow builds a Docker image for the Node.js app, tags it, and pushes it to an ECR repository.
5. Finally, the workflow connects to the EC2 instance over SSH and pulls and runs the latest image as a container on port 80.[^4][^2][^3]

So, GitHub Actions is the CI/CD engine, Terraform is the infrastructure layer, ECR is where container images live, S3 holds Terraform state, and EC2 is the runtime where the actual container runs.[^2][^1]

---

## Directory structure

The repository is intentionally kept simple:

- `nodeapp/`
  - Contains the Node.js/Express application and its `Dockerfile`.
  - You can replace this with any other Node.js or React+Node stack as long as you keep the Dockerfile consistent.[^2]
- `terraform/`
  - Contains Terraform configuration to provision the EC2 instance, security group, key pair, and any required IAM roles.
  - Uses an S3 backend to store the Terraform state file instead of keeping it locally.[^4][^2]
- `.github/workflows/`
  - Contains the GitHub Actions workflow YAML file that wires everything together: Terraform init/plan/apply, Docker build and push, and remote deployment via SSH.[^1][^2]

You can inspect each folder independently: `nodeapp` is about the app, `terraform` is infra as code, and the GitHub workflow is the glue that turns commits into running infrastructure and containers.

---

## Application (Node.js / Express)

The application itself is intentionally minimal:

```js
const express = require("express");
const app = express();

app.get("/", (req, res) => {
  res.send("Service is up and running");
});

app.listen(8080, () => {
  console.log("Server is up");
});
```

This keeps the focus on the pipeline and the AWS provisioning rather than on complex business logic. You can extend this later with more routes, a React frontend, or any additional services.[^2]

---

## Docker image

The app is containerized using a basic Dockerfile:

```Dockerfile
FROM node:14
WORKDIR /user/app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["npm","start"]
```

The CI workflow builds this Dockerfile in the `nodeapp` directory, tags it with the current commit SHA, and pushes it to an ECR repository. The EC2 instance then pulls and runs this same image so you get a consistent environment between builds and deployments.[^5][^2]

---

## Terraform and AWS setup

Terraform is used to manage the AWS resources required for deployment:

- EC2 instance (Ubuntu) that will run the Docker container.
- Security group to allow HTTP traffic (port 80) and SSH as needed.
- S3 bucket for storing Terraform state so that runs are consistent and shared.
- SSH key pair values passed in from GitHub secrets.[^4][^2]

The Terraform backend is configured via the GitHub Actions workflow using the `TF_STATE_BUCKET_NAME` environment variable so that the S3 bucket name does not need to be hardcoded.

---

## GitHub Actions CI/CD pipeline

The CI/CD pipeline is defined as a GitHub Actions workflow. It runs on pushes and is split into logical steps:

### 1. Environment variables

Secrets are injected from the repo’s settings:

```yml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  TF_STATE_BUCKET_NAME: ${{ secrets.AWS_TF_STATE_BUCKET_NAME }}
  PRIVATE_SSH_KEY: ${{ secrets.AWS_SSH_KEY_PRIVATE }}
  PUBLIC_SSH_KEY: ${{ secrets.AWS_SSH_KEY_PUBLIC }}
  AWS_REGION: us-east-1
```

This keeps credentials and sensitive data out of the codebase and in GitHub Secrets.[^6][^2]

### 2. Terraform init and backend

The workflow checks out the repository, sets up Terraform, and initializes the backend:

```yml
- name: checkout repo
  uses: actions/checkout@v2
- name: setup terraform
  uses: hashicorp/setup-terraform@v1
  with:
    terraform_wrapper: false
- name: Terraform Init
  id: init
  run: terraform init -backend-config="bucket=$TF_STATE_BUCKET_NAME" -backend-config="region=us-east-1"
  working-directory: ./terraform
```

This step links Terraform to the S3 bucket and prepares it to manage the AWS infrastructure from CI.[^7][^2]

### 3. Terraform plan and apply

The workflow then plans and applies the infrastructure changes:

```yml
- name: Terraform Plan
  id: plan
  run: |-
    terraform plan \
    -var="region=us-east-1" \
    -var="bucket=$TF_STATE_BUCKET_NAME" \
    -var="public_key=$PUBLIC_SSH_KEY" \
    -var="private_key=$PRIVATE_SSH_KEY" \
    -var="key_name=deployer-key" \
    -out=PLAN
  working-directory: ./terraform

- name: Terraform Apply
  id: apply
  run: |-
    terraform apply PLAN
  working-directory: ./terraform
```

Variables are passed in so the same configuration can be reused across environments by just changing values in the workflow or in TF vars.[^8][^2]

### 4. Exposing EC2 public IP

Once the infrastructure is applied, the workflow exposes the EC2 public IP for later steps:

```yml
- name: Set output
  id: set-dns
  run: |-
    echo "::set-output name=instance_public_dns::$(terraform output instance_public_ip)"
  working-directory: ./terraform
```

The IP is then put into an environment variable:

```yml
- run: echo SERVER_PUBLIC_IP=${{ needs.deploy-infra.outputs.SERVER_PUBLIC_DNS }} >> $GITHUB_ENV
```

This allows the deployment job to know where to SSH and where to run the container.[^1][^2]

### 5. Build and push Docker image

The image is built from the `nodeapp` directory and pushed to ECR:

```yml
- name: Login to AWS ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1

- name: Build, tag, and push docker image to Amazon ECR
  env:
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    REPOSITORY: example-node-app
    IMAGE_TAG: ${{ github.sha }}
  run: |
    docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
    docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
  working-directory: ./nodeapp
```

Each commit gets its own image tag (the SHA), which makes rollbacks and debugging easier.[^5][^2]

### 6. Remote deployment on EC2

Deployment is done over SSH using the `appleboy/ssh-action`:

```yml
- name: Deploy Docker Image to EC2
  env:
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    REPOSITORY: example-node-app
    IMAGE_TAG: ${{ github.sha }}
    AWS_DEFAULT_REGION: us-east-1
  uses: appleboy/ssh-action@master
  with:
    host: ${{ env.SERVER_PUBLIC_IP }}
    username: ubuntu
    key: ${{ env.PRIVATE_SSH_KEY }}
    envs: PRIVATE_SSH_KEY,REGISTRY,REPOSITORY,IMAGE_TAG,AWS_ACCESS_KEY_ID,AWS_SECRET_ACCESS_KEY,AWS_REGION
    script: |-
      sudo apt update
      sudo apt install docker.io -y
      sudo apt install awscli -y
      sudo $(aws ecr get-login --no-include-email --region us-east-1);
      sudo docker stop myappcontainer || true
      sudo docker rm myappcontainer || true
      sudo docker pull $REGISTRY/$REPOSITORY:$IMAGE_TAG
      sudo docker run -d --name myappcontainer -p 80:8080 $REGISTRY/$REPOSITORY:$IMAGE_TAG
```

This ensures the EC2 instance is up to date with the latest image and that the app is reachable over HTTP on port 80.[^9][^2]

---

## How to reuse this project

This setup is intentionally generic so you can reuse it in different scenarios:

- **Different app, same pipeline**
  Replace the `nodeapp` directory with your own Node.js or React+Node project, keep the Dockerfile pattern (same port, same entrypoint), and the rest of the pipeline will still work.
- **Different region or instance type**
  Change the Terraform variables (region, instance type, tags) without touching the GitHub Actions logic.[^8][^2]
- **Multiple environments (dev/stage/prod)**
  - Use different S3 buckets or key prefixes for Terraform state per environment.
  - Use different GitHub environments or branches that point to different TF vars and ECR repos.
- **Alternative deployment target**
  Instead of EC2, you could adapt the Terraform and deployment steps to target ECS or EKS later while keeping the GitHub Actions structure (build → push image → deploy) the same.[^10][^5]

Because everything is driven by code (Terraform, Dockerfile, and workflow YAML), this repository can act as a template for many AWS-based CI/CD setups with only a few variable changes.
<span style="display:none">[^11][^12][^13][^14][^15][^16][^17][^18][^19][^20][^21]</span>

<div align="center">⁂</div>

[^1]: https://spacelift.io/blog/github-actions-terraform
[^2]: https://github.com/sept1st2c/ShaSH_CICD
[^3]: https://developer.hashicorp.com/terraform/tutorials/automation/github-actions
[^4]: https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/create-a-ci-cd-pipeline-to-validate-terraform-configurations-by-using-aws-codepipeline.html
[^5]: https://registry.terraform.io/modules/cloudposse/cicd/aws/latest
[^6]: https://cicube.io/workflow-hub/hashicorp-setup-terraform/
[^7]: https://github.com/hashicorp/setup-terraform
[^8]: https://controlmonkey.io/blog/terraform-ci-cd-pipeline-aws/
[^9]: https://github.com/appleboy/ssh-action
[^10]: https://github.com/cloudposse/terraform-aws-cicd
[^11]: https://github.com/vikashishere
[^12]: https://github.com/darjidhruv26/AWS-CICD-Pipeline
[^13]: https://gist.github.com/0cb4841234d191431cd8
[^14]: https://github.com/appleboy/scp-action
[^15]: https://github.com/marketplace/actions/ssh-remote-commands
[^16]: https://dev.to/jzmt/how-i-built-a-terraform-cicd-pipeline-on-aws-with-github-actions-4o5f
[^17]: https://stackoverflow.com/questions/68132791/git-using-actions-checkoutv2-instead-of-appleboy-ssh-actionmaster-to-clone-r
[^18]: https://www.youtube.com/watch?v=5rvdMcf6ix0
[^19]: https://blog.benoitblanchon.fr/github-action-run-ssh-commands/
[^20]: https://www.env0.com/blog/terraform-github-actions
[^21]: https://faun.pub/setting-up-a-ci-cd-pipeline-with-terraform-cloud-github-and-aws-68a11427349d
