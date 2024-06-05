# Important


This setup is simplified and covers a limited scenario.
The topics discussed are too big to cover in one article and heavily depend on the use-case and business need.


**All code and configurations are not tested and displayed only as an example**



# Set Up AWS Environment


## Steps and General Recommendations


- Create an AWS Account.
- Set up MFA (Multi-Factor Authentication) for the root user.
- Set up IAM (Identity and Access Management) user.
- Apply a password policy and enforce password rotation for all IAM users.
- Enforce MFA for all IAM users with access to the Management Console. (Policy details can be found [https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_my-sec-creds-self-manage.html](LINK))


From now on, we will use only the IAM user we created, as usage of the root account may be dangerous and is not recommended.


## User Activity Logging


To log and monitor user activity, we can use CloudTrail. Documentation on how to can be found [https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-create-a-trail-using-the-console-first-time.html](LINK).


This will allow us to monitor all API requests to AWS. We can also pair it with GuardDuty to detect suspicious activity.


## Security Monitoring


* Enable GuardDuty in the AWS Management Console.
* Configure GuardDuty to monitor your AWS environment for unusual activity and potential security threats.
* Integrate GuardDuty with CloudWatch Events. Ceate CloudWatch Events rules to trigger notifications or remediation actions based on GuardDuty findings.


## Potential Improvements


If a multi-account architecture is required, it’s better to set up IAM Access Identity. It can be integrated with Azure AD, LDAP, or Google Workspace, to dynamically provision users (not supported with Workspace) based on their groups, create permission sets, and manage authentication and authorization to different accounts.


However, in a multi-account environment, this also can be achieved without a central user-management solution by having users authenticate on a central (management) account and then assuming roles in different accounts.


# Infrastructure Preparation


## ECR Repository


Create an image repository in ECR.


Repository name: `simple-flask-app`


## VPC and Networking


- Create 1 VPC per environment; as an example, let's use `10.0.0.0/16`.
- Create 3 private and 3 public subnets:
  - Private subnets: `10.0.1.0/20`, `10.0.2.0/20`, `10.0.3.0/20`
  - Public subnets: `10.0.11.0/20`, `10.0.12.0/20`, `10.0.13.0/20`


### Considerations


This CIDR range should ensure:
- VPC total addresses: `65536`
- Subnets total addresses: `4096` per subnet


AWS reserves 5 addresses in each subnet:
- `10.0.0.0` - network address
- `10.0.0.1` - VPC router
- `10.0.0.2` - DNS
- `10.0.0.3` - reserved for future use
- `10.0.0.255` - network broadcast


### Additional Resources


- 1 Internet Gateway for communication from VPC to the outside.
- NAT Gateways:
  - 1 NAT Gateway, from outside communication.
  - 1 NAT Instance provisioned in a single AZ can cause issues if AZ is down.
  - 3 NAT Instances, one per public AZ, more resilient, but more expensive.


Note: Also requires Elastic IPs.


### Routing


- 2 routing tables:
  - 1 for public subnets:
    - `0.0.0.0/0` to an Internet gateway.
  - 1 for private subnets:
    - `0.0.0.0/0` routes to NAT Gateway.


### Potential Improvements


- 1 route table per private subnet for better control over routing.
- Network ACLs to restrict traffic at the subnet level.


### Route 53


Create a hosted zone in AWS Route 53. Point NS records to this hosted zone, in case the domain is purchased outside AWS.


### AWS Certificate Manager


Request SSL certificate for `app.example.com`. This can be later used in ALB for secure connections. \
Validation via DNS as we already have hosted zone.


### S3


Create a simple private S3 bucket named `artifacts-bucket`.

Potential options may include: 
- Encryption with KMS.
- Versioning.
- Object lifecycle management, for example, moving old objects to different storage types to save costs.
- Replication to another region (if backups are needed).


### Relational Database Service


Setup a Postgres database:
- Network: Hosted in private subnets.
- Security groups: Traffic allowed only from required resources (like the web app) on port 5432.
- Encryption and encryption in transit are enabled, and TLS is forced by the parameters group.
- Multi-AZ enabled, for high availability.
- Daily backups, retention period at least 30 days.
- Authentication: single username/password.


### Potential Improvements


- Read replica, for heavy read workflows, or if access for Business Analysis is needed.
- IAM authentication.
- Password rotation via Secrets Manager.
- Backups can be copied to other regions for data integrity.
- Amazon Aurora (Postgres compatible) can be used if a multi-region environment is required.


## ECS


For simplicity of the task, I would go with the ECS cluster, as EKS requires deep knowledge of Kubernetes and more time for configuration.

Cluster name: `prod-cluster`


### ECS Configuration 


- AWS Fargate - for serverless computing (my pick for this task, because of configuration simplicity).
- Container Insights - if detailed metrics on container performance are required.

**NOTE: Task definitions and service is described bellow** 

# Create Dockerized Application


## Dockerize the Application


Create a Dockerfile.


```
FROM python:3.8-slim


WORKDIR /app


COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt


COPY . .


EXPOSE 5000
CMD ["python", "app.py"]


```


Build the container:
```
docker build -t simple-flask-app .
```


Push Docker Image to ECR
```
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.eu-west-1.amazonaws.com
docker tag simple-flask-app:latest <account-id>.dkr.ecr.eu-west-1.amazonaws.com/simple-flask-app:latest
docker push <account-id>.dkr.ecr.eu-west-1.amazonaws.com/simple-flask-app:latest
```


Now we have a web-app image in our ECR repository.


# CI/CD


For CI/CD I am using GitHub Actions.

We need at least 3 workflows to cover this example scenario:
1. Deploy branches to stable environments like `dev`,`stage`,`prod`
2. Dynamically provision QA temporary environments
3. Purge dynamically provisioned resources

In this part, we only focus on the manual deployment of any branch to two predefined environments.


## Deployment on stable environments


This workflow allows you to deploy any branch to a `stage` or `prod` environment.


Workflow File: `.github/workflows/deployment.yml`


```
name: Branch Deployment to Selected Env


on:
  workflow_dispatch:
    inputs:
      branch:
        description: 'Git branch to deploy'
        required: true
        default: 'main'
      environment:
        description: 'Deployment environment'
        required: true
        default: 'stage'
        type: choice
        options:
          - stage
          - prod


jobs:
  build-and-deploy:
    runs-on: ubuntu-latest


    steps:
    - name: Checkout code
      uses: actions/checkout@v2
      with:
        ref: ${{ github.event.inputs.branch }}


    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1


    - name: Log in to Amazon ECR
      id: ecr-login
      uses: aws-actions/amazon-ecr-login@v1
      with:
        region: eu-west-1


    - name: Build and tag Docker image
      run: |
        docker build -t ${{ github.repository }}:${{ github.event.inputs.branch }} .
        docker tag ${{ github.repository }}:${{ github.event.inputs.branch }} ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:${{ github.event.inputs.branch }}


    - name: Snyk Docker Image Scan
      uses: snyk/actions/docker@master
      with:
        args: --severity-threshold=high
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
 
    - name: Run Unit Tests
      run: |
        docker run --rm simple-flask-app pytest


    - name: Push Docker image to ECR
      run: |
        docker push ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:${{ github.event.inputs.branch }}


    - name: Deploy to ECS
      env:
        TASK_DEFINITION: ${{ secrets[github.event.inputs.environment == 'stage' && 'STAGE_TASK_DEFINITION' || 'PROD_TASK_DEFINITION'] }}
      run: |
          echo "${TASK_DEFINITION}" > ecs-task-definition.json
          aws ecs update-service \
            --cluster ${{ github.event.inputs.environment }}-cluster \
            --service ${{ github.event.inputs.environment }}-service \
            --task-definition ecs-task-definition.json \
            --force-new-deployment
 
    - name: Wait for service stability
      run: |
        aws ecs wait services-stable \
          --cluster ${{ github.event.inputs.environment }}-cluster \
          --services ${{ github.event.inputs.environment }}-service
```


### Steps:

- Uses the actions/checkout@v2 action to check out the specified branch.
- Uses the `docker/setup-buildx-action@v1` action to set up Docker Buildx, a Docker CLI plugin that extends the docker build command with the full support of the features provided by Moby BuildKit builder toolkit.
- Uses the `aws-actions/amazon-ecr-login@v1` action to authenticate Docker with Amazon ECR.
- Builds the Docker image with a tag corresponding to the Git branch name.
- Tags the image with the repository and branch name for ECR.
- Pushes the tagged Docker image to the specified ECR repository.
- Uses the `aws ecs update-service` action to deploy the Docker image to an ECS service.


**Required Secrets**
- `ECR_REGISTRY:` The URI of your ECR registry.
- `ECR_REPOSITORY:` The name of your ECR repository.
- `AWS_ACCESS_KEY_ID:` Your AWS access key ID.
- `AWS_SECRET_ACCESS_KEY:` Your AWS secret access key.
- `SNYK_TOKEN` - Your Snyk token for image security scanning (Included just as an example)
- `STAGE_TASK_DEFINITION`: The JSON content of the ECS task definition for the stage environment.
- `PROD_TASK_DEFINITION`: The JSON content of the ECS task definition for the prod environment.


Task definitions are included in the repository.




# QA environment




Infrastructure for QA is similar to that described above. The only thing we need to solve is dynamically provision and purge applications, so every feature can be tested in isolation.


The best-case scenario would be to use CI/CD and PR events to create and remove specific versions of application. We can leverage on "PR opened" and "PR closed/merged" events to do this automatically.


However, for simplicity, let's consider manual deployments.


The scenario is following:
1. Developed and finished working on a feature branch: `feature/new-awesome-feature`
2. Developer commits and pushes changes
3. he developer creates a pull request (PR) to merge the feature/new-awesome-feature branch into the QA branch for testing.




Goal is to provision new version of application from `feature/new-awesome-feature` branch.


```
name: Branch Deployment to Selected Env
'on':
  workflow_dispatch:
    inputs:
      branch:
        description: Git branch to deploy
        required: true
        default: main
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          ref: '${{ github.event.inputs.branch }}'
      - name: Generate unique tag
        id: tag
        run: 'echo "::set-output name=tag::qa-$(uuidgen | cut -d''-'' -f1)"'
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      - name: Log in to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v1
        with:
          region: eu-west-1
      - name: Build and tag Docker image
        run: >
          docker build -t ${{ github.repository }}:${{ steps.tag.outputs.tag }} .
          docker tag ${{ github.repository }}:${{ github.event.inputs.branch }}
          ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:${{
          steps.tag.outputs.tag }}
      - name: Snyk Docker Image Scan
        uses: snyk/actions/docker@master
        with:
          args: '--severity-threshold=high'
        env:
          SNYK_TOKEN: '${{ secrets.SNYK_TOKEN }}'
      - name: Run Unit Tests
        run: |
          docker run --rm simple-flask-app pytest
      - name: Push Docker image to ECR
        run: >
          docker push ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY
          }}:${{ steps.tag.outputs.tag }}


      - name: Deploy to ECS
        env:
            UNIQUE_TAG: ${{ steps.tag.outputs.tag }}
        run: |
            ecs_task_definition=$(cat <<EOF
            {
              "family": "flask-app-qa",
              "networkMode": "awsvpc",
              "containerDefinitions": [
                {
                  "name": "simple-flask-app-container-${UNIQUE_TAG}",
                  "image": "${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:${UNIQUE_TAG}",
                  "portMappings": [
                    {
                      "containerPort": 5000,
                      "hostPort": 8080
                    }
                  ],
                  "essential": true
                }
              ],
              "requiresCompatibilities": [
                "FARGATE"
              ],
              "cpu": "512",
              "memory": "4096"
            }
            EOF
            )

            echo "$ecs_task_definition" > ecs-task-definition.json
            aws ecs register-task-definition --cli-input-json file://ecs-task-definition.json

            aws ecs create-service --cluster qa-cluster --service-name ${{ steps.tag.outputs.tag }}-service --task-definition my-app-qa --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxxx],securityGroups=[sg-xxxxxxx],assignPublicIp=ENABLED}"
      - name: Get task ARN
        id: task-arn
        run: >
          SERVICE_ARN=$(aws ecs list-tasks --cluster qa-cluster --service-name
          ${{ steps.tag.outputs.tag }}-service --query 'taskArns[0]' --output
          text)

          echo "::set-output name=task_arn::$SERVICE_ARN"
      - name: Get ENI ID
        id: eni-id
        run: >
          ENI_ID=$(aws ecs describe-tasks --cluster qa-cluster --tasks ${{
          steps.task-arn.outputs.task_arn }} --query
          'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value'
          --output text)

          echo "::set-output name=eni_id::$ENI_ID"
      - name: Get public IP
        id: public-ip
        run: >
          PUBLIC_IP=$(aws ec2 describe-network-interfaces
          --network-interface-ids ${{ steps.eni-id.outputs.eni_id }} --query
          'NetworkInterfaces[0].Association.PublicIp' --output text)
          echo "::set-output name=public_ip::$PUBLIC_IP"
      - name: Set up DNS
        run: >
          aws route53 change-resource-record-sets --hosted-zone-id
          your-hosted-zone-id --change-batch '{
            "Changes": [
              {
                "Action": "CREATE",
                "ResourceRecordSet": {
                  "Name": "'${{ steps.tag.outputs.tag }}.qa.exmaple.com'",
                  "Type": "A",
                  "TTL": 300,
                  "ResourceRecords": [
                    {
                      "Value": "'${{ steps.public-ip.outputs.public_ip }}'"
                    }
                  ]
                }
              }
            ]
          }'
```


### Explanation
* Generates a unique identifier for the dynamic QA environment using uuidgen.
* Checks out the selected build branch.
* Sets up Docker Buildx for building Docker images.
* Authenticates Docker with Amazon ECR.
* Builds the Docker image and tags it with the unique identifier.
* Pushes the tagged Docker image to ECR.
* Registers the ECS task definition using the unique Docker image tag.
* Creates a new ECS service with a unique tag in the QA cluster.
* Retrieves the ARN of the ECS task created by the service.
* Retrieves the Elastic Network Interface (ENI) ID associated with the ECS task.
* Retrieves the public IP address associated with the ENI.
* Configures Route 53 to create a DNS record for the new QA environment, pointing to the ECS service's public IP.






# Production Environment


## Key deferences for production environment


For production infrastructure, we need to add the following:

1. Setup ALB in front of ECS service
2. Setup WAF in front of ALB
3. Utilize AWS Code Deploy to ensure Blue-Green Deployments for production(ECS supports only blue-green with Code Deploy)
4. Configure monitoring and alerting
5. Adjust CI/CD for production deployments


## Monitoring


* Create alarms for critical metrics such as CPU utilization, memory utilization, request latency, and error rates.
* Configure the alarms to send notifications to an SNS topic, which can alert operations.
* Enable AWS X-Ray for your application to trace requests and get insights into performance bottlenecks and errors.




# Puting all together
The last piece we are missing is actually running our application. For that, we need several components:
1. Execution Role - for ECS to run our task, access ECR etc.
2. Task Role - a role for web-app, used in the example to access the s3 bucket.
3. ECS Task Definition - runtime definition and settings for web-app container
4. ECS Service - covers load balancing, networking, auto-scaling, storage etc.
5. Monitoring and alarms
6. Security(WAF)
7. CI/CD adjustments


## Execution Role
Very simple role will basic permissions for ECS:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```
This will allow ECS to access the ECR image and create log groups.


## Task Role
Simple role with access to earlier created bucket:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::artifacts-bucket/*"
        }
    ]
}


```
## Task Definition
An example of it can be found in the repository under `task-definition-prod.json`
```
aws ecs register-task-definition \
    --cli-input-json file://task-definition-prod.json
```


Note, that this task definition is simplified.


## ECS Service
Service can be created from Management Console and include the following:
* Deployed in private subnets
* ALB load balancer in front with SSL certificate
* Scaling policies


## Monitoring and Alaram


* Create alarms for critical metrics such as CPU utilization, memory utilization, request latency, and error rates.
* Configure the alarms to send notifications to an SNS topic, which can alert your operations team.


## Security

Configure WAF in front of ALB to catch all suspicious traffic or user activities.


## CI/CD

Evaluate CI/CD to combine it with AWS Code Deploy. 

