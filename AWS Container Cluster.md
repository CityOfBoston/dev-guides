# AWS Container Cluster Setup

### Naming

Names should be as consistent as possible throughout the system. All apps should
be referred to by their GitHub repo name. When resources are created globally in
the AWS account, they should be prefixed with “digital”. Things releated to the
apps cluster itself should be prefixed with “digitalApps”. Truly global names
(like S3 buckets) should be prefixed by “cob”.

## One-time Setup

While Amazon has a cluster-creation wizard, here are the manual steps necessary
for setting a cluster up from scratch. These are preferred in order to make all
of the components of the system clear.

### S3 > Bucket

Create a bucket to store our protected production environment variables. Turn on
versioning. Do not grant any permissions.

Add the following policy to require encrypted data transfer, replacing `BUCKET_NAME`
with the name of the new bucket:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnEncryptedObjectUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::BUCKET_NAME/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        },
        {
            "Sid": " DenyUnEncryptedInflightOperations",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::BUCKET_NAME/*",
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

Documentation: [How to Manage Secrets for Amazon EC2 Container Service–Based
Applications by Using Amazon S3 and
Docker](https://aws.amazon.com/blogs/security/how-to-manage-secrets-for-amazon-ec2-container-service-based-applications-by-using-amazon-s3-and-docker/)

### CloudWatch > Log Group

Create a new log group. Set its retention policy to 3 months.

### VPC > Virtual Private Cloud (VPC)

Documentation: [VPCs and
Subnets](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html)

Create a new VPC with at least 2 subnets across different AZs, and an Internet
gateway. Or, use an existing VPC.

Ensure that the VPC has an Internet Gateway and its routes table sends 0.0.0.0/0
to it.

### VPC > VPN

Add a connection from the VPC to the City’s hardware VPNs. Ensure that the
routes tables are updated to use these devices.

Details TBD, we haven’t set this up at the time of writing. 

### VPC > Security Groups

In the VPC, create 2 security groups:

 * For the load balancer, accept HTTP and HTTPS traffic from Anywhere.
 * For the instances, accept all TCP traffic from the load balancer’s security
   group as the source. For debugging, it is good to allow SSH access from
   either anywhere or (preferrable) City Hall’s proxy IP addresses.

### IAM > Roles

Make roles for the following AWS services:

 * **Amazon EC2 Role for EC2 Container Service** This is your instance role.
 * **Amazon EC2 Container Service Role** This is your service role.
 * **Amazon EC2 Container Service Task Role** This your task role.

Be sure to select the suggested policies when you create the roles!

### IAM > Users

Create an IAM user for Travis CI. Give it programmatic access only.

### IAM > Policies

Create a policy for read-only access to our secret credentials S3 bucket,
replacing `BUCKET_NAME` with the bucket name:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::BUCKET_NAME"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::BUCKET_NAME/*"
            ]
        }
    ]
}
```

**Attach this policy to the task role created above.**

Create a policy for deploying from a CI server, replacing `SERVICE_ROLE_ARN` and
`TASK_ROLE_ARN` with the identifiers for those IAM roles you created above.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "application-autoscaling:Describe*",
                "application-autoscaling:RegisterScalableTarget",
                "ecs:List*",
                "ecs:Describe*",
                "ecs:RegisterTaskDefinition",
                "ecs:CreateService",
                "ecs:UpdateService",
                "elasticloadbalancing:Describe*",
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetRepositoryPolicy",
                "ecr:DescribeRepositories",
                "ecr:ListImages",
                "ecr:DescribeImages",
                "ecr:BatchGetImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": [
                "<<<SERVICE_ROLE_ARN>>>",
                "<<<TASK_ROLE_ARN>>>"
            ]
        }
    ]
}
```

**Attach this policy to the Travis CI IAM User.**

### ECS > Cluster

Create an empty cluster. We need to do this before creating instances so that we
put its name in their `ecs.config` files.

### EC2 > Key Pair

Create a key pair to install on the EC2 instances so that you can SSH into them
for debugging. Download the PEM file and put it in your `.ssh` directory, then
use `ssh-add` to put it in your `ssh-agent` session.

(On a Mac, you can use `ssh-add -K` to put the (blank) passphrase in the
Keychain, so that the key can be automatically re-added to `ssh-agent` after
reboots. On Mac OS X Sierra and later, you will have to do [additional
setup](https://superuser.com/a/1163862) to auto-add.)

### EC2 > Launch Configuration (technically optional)

Documentation: [Launching an Amazon ECS Container
Instance](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/launch_container_instance.html)

Create a launch configuration. Use **amazon-ecs-optimized** as the AMI. Use
whatever instance type makes sense. Associate it with VPC, instance IAM role,
and instance security group from above. Enable advanced CloudWatch metrics.

Under Configure Details > Advanced Details, set the User data to register
instances automatically with our cluster using the following script:

```
#!/bin/bash
echo ECS_CLUSTER=YOUR_CLUSTERS_SHORT_NAME_HERE >> /etc/ecs/ecs.config
```

Also set the instances to always receive a public IP. Public IPs are necessary
for our app container services to make outgoing network requests, and for
incoming SSH connections.

On creation, a dialog will appear about adding a key pair. Select the key pair
created above.

Launch configurations cannot be modified once created. If you do need to make
changes, duplicate an existing one and then update the Autoscaling Group to use
your newly-created launch configuration.

### EC2 > Autoscaling Group (technically optional)

Create an autoscaling group from your launch configuration. Use your VPC and
associate it with the VPC’s subnets so that it can scale across AZs.

Set the health check grace period to 0. Don’t worry about any autoscaling rules
or notifications.

_The Launch Configuration and Autoscaling Group are technically optional. You
can launch instances individually and, by setting their `ecs.config` files, have
them associate with the cluster. Nevertheless, we use these so that the options
used to create an instance are easily visible and repeatable._

### EC2 > Load Balancer

Create an Application Load Balancer. (We can use ALB rules to route different
hosts to different ECS services.) Listen on HTTP and HTTPS. Use the VPC from
above and put the ALB in all of the availability zones we’ll be using.

Choose the load balancer security group (that allows public access) from above.

The wizard will require you to create a target group. Make a temporary one that
will be deleted later. We’ll start hooking things up to the ALB when we do
per-service configuration.

### Verification and Troubleshooting

Make sure that the autoscaling group has at least one “Desired” instance and
check to make sure that it appears in the cluster’s “ECS Instances” tab.

If it does not, check the following:

 * Ensure that the Autoscaling Group’s “Instances” tab shows Healthy instances.
   Check the ”Activity History“ tab for hints about what’s wrong if this is not
   the case.
 * Check that the instance has an IPv4 Public IP. If it does not, update your
   launch configuration to always assign instances public IPs.
 * Check the ecs-agent logs on the instance. Use `ssh-add` to add the PEM file
   from the keypair if you haven’t already, then `ssh ec2-user@IP_ADDRESS` to
   connect. Check the ECS agent logs in the `/var/log/ecs/` directory for hints.
   If you see permission errors, verify that your launch configuration has
   assigned your container instance IAM role to its instances, and that that
   role has the AmazonEC2ContainerServiceforEC2Role policy attached.

The ALB should be accessible at its auto-created domain name, but is expected to
return Service Unavailable responses because we haven’t associated any services
with its Target Groups.

## Per-Service Setup

For each app that we launch, there is some amount of manual setup that needs to
happen in AWS.

Our deploy scripts handle both first-run of a container service (creating it and
registering it with the load balancer) as well as subsequent deploys.

### Route 53

TBD. Associate a new domain name with our ALB.

### CloudFront

Create a new CloudFront Web Distribution. For the origin, choose our ALB.

Add a “Host” header for the app’s domain name, since CloudFront doesn’t natively
work with Target Groups.

Set the Price Class to only US, Canada, and Europe. Put the app’s name in the comment.

_Next.js currently only works with reverse-proxy CDNs. The ability to pre-upload
JS to a CDN (such as an S3 bucket) is planned._

### EC2 > Target Group

Create a new Target Group for the app.

The port doesn’t matter, since the container’s port will override it. Associate
it with our VPC. Set the health check URL to “/admin/ok”.

After creation, edit the “Deregistration delay” attribute to be 30s.

### EC2 > Load Balancers

Edit the listeners for HTTP and HTTPS to add a rule that forwards the host name
from above to the new Target Group.

### ECS > Repository

Create a new repository for the app’s container images. It should match the
pattern `NAMESPACE/SERVICE_NAME`.

### IAM > Users

For the Travis user, create a new Access Key under Security credentials and save
it for later.

### S3 > Config Bucket

Upload an `.env` file to the S3 config bucket, under a path matching the GitHub
repo / service name that you’re using (_e.g._ `s3://config-secrets/registry-certs/.env`).

Include any credentials needed, and set `NEXT_ASSET_PREFIX` to the https URL for
the CloudFront distribution created above.

### App Repository

Make sure that the app has a Dockerfile, docker-compose.yml, and the
ecs-deploy.js, install-ecs-cli.sh, and entrypoint.sh scripts. These can be
copied from an existing repository, as they are generic across our apps.

Add the following to the .travis.yml file:

```
before_deploy:
- ./deploy/install-ecs-cli.sh
deploy:
  provider: script
  script: node ./deploy/ecs-deploy.js
  on:
    branch: develop
```

### Travis

We use several environment variables to configure the creation and deployment of
the container. This is done to keep Boston-specific identifires and especially
secret tokens out of the GitHub repository.

Private variables should have the “Display value in build log” set to **OFF**.

Public environment variables are identifiers that are good to see for our own
debugging, and for comparing values between repositories. They should have
“Display value in build log” switched to **ON**.

#### Global Variables (Public)

These variables are the same for all of our apps.

 * **AWS_CLOUDWATCH_LOGS_GROUP**: The name of the log group created above.
 * **AWS_DEFAULT_REGION**: Likely `us-east-2`. 
 * **AWS_ECS_CLUSTER**: The short name of the cluster (_e.g._ “digital-apps”)
 * **AWS_ECS_REGISTRY**: Name of the ECS registry, such as
   `XXXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com`.
 * **AWS_ECS_REPOSITORY_NAMESPACE**: Namespace for Digital Apps containers,
   which will preceed the container name by a '/'.
 * **AWS_ECS_SERVICE_ROLE_ARN**: The ARN of our service role.
 * **AWS_ECS_TASK_ROLE_ARN**: The ARN of our task role (the one that has the
   permission to access the config S3 bucket)
 * **AWS_S3_CONFIG_BUCKET**: Name of the bucket that will hold our config .

#### App-specific Variables (Public)

These will be different app-to-app.

 * **AWS_ACCESS_KEY_ID**: The access key created for the Travis user for this
   app. Will probably vary app-to-app unless someone is remembering the secret
   tokens.
 * **AWS_EC2_TARGET_GROUP_ARN**: The ARN for the app’s ALB Target Group.
 * **AWS_ECS_SERVICE_NAME**: Name of the service to create / update in the
   cluster. Should match the GitHub repository, optionally with a
   “-staging” suffix. Will also be used as a prefix for logging, s3, container
   repository, _&c._

#### App-specific Variables (Private)

 * **AWS_SECRET_ACCESS_KEY**: The secret access key that goes with the Access
   Key ID from above. Remember that this key has the power to deploy code into
   our VPC in our VPN, read database credentials, _&c._ Be careful with it, and
   if it accidentally leaks, disable it immediately in the IAM console.

### Testing Locally

#### Docker

Install [Docker](https://www.docker.com/) for your desktop.

Run `docker build -t app:dev -f deploy/Dockerfile .` in the repository
directory. Docker should build the container, installing node_modules and
building the JavaScript.

Start the container with `docker run -p 3000:3000 -e NODE_ENV=production --entrypoint= --rm app:dev npm start`

_This command binds the container’s server port to your local machine, sets the
`NODE_ENV` so that nothing will be recompiled, overrides the entrypoint that
downloads .env from the private S3 bucket, and ensures that the container will
be deleted when the task stops._

If you visit http://localhost:3000/ you should see your app running.

Press Control-C a few times to kill the server.

#### AWS

 * Install the [AWS command-line tool](https://aws.amazon.com/cli/)
 * Install the [ecs-cli tool](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI.html)

Run `aws configure` to set up your access credentials and set the default
region, so you don’t have to fill those in for other commands. Your user account
will need to have at least the permissions from the Travis deploy policy.

In order to run the deploy script, you will need to set all of the environment
variables above, excluding `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and
`AWS_DEFAULT_REGION`.

Run `node deploy/ecs-deploy.js` to build the container, push it to the ECR, and
deploy a new version of the service.

## Monitoring

### CloudWatch > Alarm

This has to be set up after a container has registered with the Target Group
(_i.e._ after the first deploy) for the metric to show up.

Add an alarm for the “Per AppELB, Per TG” metric for “Healthy Host Count” for
the new Target Group.

Set the alarm to “< 1” for 2 consecutive periods and change “Treat missing data
as” to “bad.” Set the Period to 1 minute.

### Things to Check In On

 * See how containers are using their memory.
 * Be aware of the CPU and memory usage on the instances.
 * ELB responses, possibly adding a CloudWatch alarm for 500s.

## Housekeeping

### ECS > Task Definitions

Old versions of task definitions can be Deregistered to keep them from
cluttering up the UI. They cannot, however, be deleted.

### ECS > Repositories

Old container images can be deleted to save space and keep us under the 1000
image limit per repository.
