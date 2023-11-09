## Pre-requisites
* linux machine
* Terraform
* jq
* kubectl
* eksctl
* aws-cli
* git
* ec2 ssh keys named eksa pre-generated
* Region used is us-east-2
* An AMI in us-east-2 with docker and docker-compose installed

## Steps for EC2 ASG
The below will generate a VPC with all networking required along with a Classic ELB and ASG with auto-scaling policies. In addition, it will create an EC2 instance for locust testing
```
git clone https://github.com/thecloudgarage/spit.git
cd spit
# EDIT variables.tf to insert the AWS keys
# EDIT main.tf to change the AMI ID of the locust instance
terraform init && terraform validate && terraform plan && terraform apply -auto-approve
```
Once done, open the locust EC2 IP suffixed by 8089 and insert the ELB url for testing. Validate auto-scaling.

### Steps for k8s auto-scaling demo
Navigate to Kubernetes directory and start executing the steps.
