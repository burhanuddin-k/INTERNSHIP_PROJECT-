# Project 3: Docker Images in Amazon ECR

## Overview
This project demonstrates building Docker images, pushing and pulling images from Amazon Elastic Container Registry (Amazon ECR), and configuring repository permissions and a lifecycle policy using the AWS CLI.

## Project Details
- **AWS Service:** Amazon Elastic Container Registry (Amazon ECR)
- **Region:** `eu-north-1`
- **Repository:** `burhanuddin-private-registry`
- **Registry URI:** `225874124228.dkr.ecr.eu-north-1.amazonaws.com`
- **Image tags:** `v1`, `v2`, `v3` (the AWS Console screenshot also shows `v4` and `latest`)

## Project Structure
```text
Project-3/
├── docker-app/
│   ├── Dockerfile.txt
│   └── index.html
├── policies/
│   ├── lifecycle-policy.json
│   └── repository-policy.json
├── screenshots/
│   ├── 2.png
│   ├── 3.png
│   ├── 4.png
│   ├── 5.png
│   ├── 6.png
│   ├── 7.png
│   ├── 8.png
│   ├── 9.png
│   ├── 10.png
│   ├── 11.png
│   ├── 12.png
│   ├── 13.png
│   ├── 14.png
│   └── 15.png
└── README.md
```

## Step 1: Prepare the Docker Application
The `docker-app` directory contains the Docker build instructions and the HTML file used by the sample application.

![Docker application files](screenshots/2.png)

## Step 2: Authenticate Docker with Amazon ECR
The AWS CLI was used to retrieve an authentication token and log in to the private ECR registry.

```powershell
aws ecr get-login-password --region eu-north-1 --profile project3 | docker login --username AWS --password-stdin 225874124228.dkr.ecr.eu-north-1.amazonaws.com
```

![Docker image and ECR login](screenshots/3.png)

## Step 3: Check Local Docker Images
The local Docker image list was checked before pushing images to ECR.

```powershell
docker images
```

![Local Docker images](screenshots/4.png)

## Step 4: Push the Docker Image to ECR
The Docker image was pushed to the private repository.

```powershell
docker push 225874124228.dkr.ecr.eu-north-1.amazonaws.com/burhanuddin-private-registry:v1
```

![Docker push v1](screenshots/5.png)

## Step 5: Verify Images in ECR
The AWS CLI was used to inspect the image details stored in the repository.

```powershell
aws ecr describe-images --repository-name burhanuddin-private-registry --region eu-north-1 --profile project3
```

![ECR image details](screenshots/6.png)

## Step 6: Tag and Push Additional Versions
Additional Docker image versions were tagged and pushed to the repository.

```powershell
docker tag burhanuddin-app:v3 225874124228.dkr.ecr.eu-north-1.amazonaws.com/burhanuddin-private-registry:v3
docker push 225874124228.dkr.ecr.eu-north-1.amazonaws.com/burhanuddin-private-registry:v3
```

![Push v3](screenshots/7.png)

The CLI output confirms the image tags and statuses.

```powershell
aws ecr describe-images --repository-name burhanuddin-private-registry --region eu-north-1 --profile project3 --query "imageDetails[?imageTags!=null].[imageTags[0],imagePushedAt,imageStatus]" --output table
```

![Image tags and status](screenshots/8.png)

## Step 7: Configure the Lifecycle Policy
A lifecycle policy was created to retain the latest three tagged images matching the `v*` pattern.

Policy file: `policies/lifecycle-policy.json`

![Lifecycle policy file](screenshots/9.png)

The policy was previewed and then applied to the repository.

![Lifecycle policy preview and apply](screenshots/10.png)

The lifecycle policy was checked after it was applied.

```powershell
aws ecr get-lifecycle-policy --repository-name burhanuddin-private-registry --region eu-north-1 --profile project3
```

![Lifecycle policy verification](screenshots/11.png)

## Step 8: Configure the Repository Policy
A repository policy was applied for the `Project3-ECR-User` IAM user, using the permissions defined in `policies/repository-policy.json`.

```powershell
aws ecr set-repository-policy --repository-name burhanuddin-private-registry --policy-text file://policies/repository-policy.json --region eu-north-1 --profile project3
```

![Set repository policy](screenshots/12.png)

The policy was retrieved to verify its configuration.

```powershell
aws ecr get-repository-policy --repository-name burhanuddin-private-registry --region eu-north-1 --profile project3
```

![Verify repository policy](screenshots/13.png)

## Step 9: Test Pulling an Image from ECR
After logging out, the image pull failed because Docker had no registry credentials. After logging in again, the image pull succeeded.

```powershell
docker logout 225874124228.dkr.ecr.eu-north-1.amazonaws.com

aws ecr get-login-password --region eu-north-1 --profile project3 | docker login --username AWS --password-stdin 225874124228.dkr.ecr.eu-north-1.amazonaws.com

docker pull 225874124228.dkr.ecr.eu-north-1.amazonaws.com/burhanuddin-private-registry:v1
```

![Logout, login, and pull test](screenshots/14.png)

## Step 10: Verify the Repository in AWS Console
The AWS Console was used to inspect the repository and its image list.

![Amazon ECR console images](screenshots/15.png)

## Outcome
- Prepared a Docker application and local Docker images.
- Authenticated Docker to Amazon ECR with the AWS CLI.
- Pushed and pulled images from a private ECR repository.
- Verified image tags and statuses.
- Created and applied lifecycle and repository policies.
- Verified the repository in the AWS Console.

## Security Notes
- Never commit AWS access keys, secret keys, session tokens, or local credential files.
- Keep the `project3` AWS CLI profile configured locally.
- Review IAM and repository permissions before using this setup outside a lab environment.
