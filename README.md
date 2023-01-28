# Handson Microservice-XRay-AppMesh

## 1 - Setup Microservice

* start with _productapp_ and _paymentapp_ taskdefinition

```bash
cd 1-Setup
aws ecs register-task-definition  --cli-input-json file://td-productapp-setup.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-paymentapp-setup.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-frontendapp.json --region us-west-2
```

* create ECS service for both, _productapp_ and _paymentapp_ both without public URL assignment and without LB
* grab URLs (host:port) of both services and set it in the td-frontendapp.json taskdefinition, environment variables
* create taskdefinition for frontendappapp (the frontend)

```bash
aws ecs register-task-definition  --cli-input-json file://td-frontendapp-setup.json --region us-west-2
```

* create frontendapp service
* grab public URL of frontendapp ALB Url and make some requests

## 2 - Adding tracing with AWS XRay and logging with AWS CloudWatch

Extend role _ecsTaskRole_ by attaching policy **AWSXRayDaemonWriteAccess**

Updating the task definitions to add xray-daemon sidecar container and additional env properties.  

```bash
cd 2-Tracing-Logging
aws ecs register-task-definition  --cli-input-json file://td-productapp-tracing.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-paymentapp-tracing.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-frontendapp-tracing.json --region us-west-2
```

Ensure to update the final IP addresses of the product- / payment-app within the frontendapp taskdefinition, section _environment variables_

To apply all the changes, redeploy the corresponding ECS service and select the latest revision of the task definition.

## 3 - Adding service discovery

## changes to our setup

- productapp and paymentapp PORT 80, instead of 9001/9002
- adjust security groups "productsvc" and "paymentsvc" to allow port 80
- adjust frontendapp task definition to replace the product_HOST and payment_HOST env variables by the DNS names of the corresponding services, productsvc.ecs-course.local and paymentsvc.ecs-course.local
- apply the changed task definitions
- delete existing services for product- , and paymentsvc
- recreate service for product- , and paymentsvc including _service discovery_
- redeploy the frontendapp service with latest revision

##  Service discovery details

- namespace: _ecs-course.local_
- service discovery services: _productsvc_ and _paymentsvc_

## apply task definitions

```bash
cd 3-ServiceDiscovery
aws ecs register-task-definition  --cli-input-json file://td-productapp-servicediscovery.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-paymentapp-servicediscovery.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-frontendapp-servicediscovery.json --region us-west-2
```

## 4 - Adding AppMesh

## extend TaskExecutionRole
attach policy _AWSAppMeshEnvoyAccess_

## create AppMesh resources

* open AWS mgm console, _AppMesh_ service
* click _Create Mesh_
* provide name _frontendapp-mesh_ and click button _Create mesh_
* create AppMesh components

```bash
aws cloudformation create-stack --stack-name appmesh-resources --template-body file://./mesh-resources.yaml
```

## adjust ECS services

### frontendapp service

#### Taskdefinition

* change to networking mode _awsvpc_ and launchtype _FARGATE_
* ensure env variables for product and payment hosts match their service discovery names **product-service.ecs-course.local** and **payment-service.ecs-course.local**
* delete environment variable **AWS_XRAY_DAEMON_ADDRESS**
* delete _Links_ entry _xray-daemon_ (this is no longer required in network mode awsvpc)
* enable AppMesh integration by clicking checkbox _Enable App Mesh integration_
  * select _frontendapp_ as application container name
  * select _frontendapp-mesh_ as Mesh name
  * select _frontendapp-service-vn_ as Virtual node name
  * **click Apply** !
  * click _Confirm_
* in Container definitions, open the _envoy_ container
  * add environment  variable _ENABLE_ENVOY_XRAY_TRACING_ with value _1_
  * enable Cloudwatch logging
* click on _Create_ to create the new task definition revision

#### ECS service

* delete existing ECS service _frontendappsvc_
* delete listener _9000_ in ALB
* create new service
  * add to loadbalancer, new listener port _9000_, new target group _frontendappsvc_
  * click _Enable service discovery integration_
  * select the existing namespace _ecs-course.local_
  * create new service discovery service _frontendapp-service_


### product service

#### Taskdefinition

* move to _Fargate_ launchtype (to avoid the limitation of ENIs on our t2.small EC2 instance) by switching from _EC2_ to _Fargate_ in Requires compatibilities
* set _Task memory_ to _1GB_
* set _Task CPU_ to _0.5vCPU_
* enable AppMesh integration by clicking checkbox _Enable App Mesh integration_
  * select _productapp_ as application container name
  * select _frontendapp-mesh_ as Mesh name
  * select _product-service-vn_ as Virtual node name
  * **click Apply** !
  * click _Confirm_
* in Container definitions, open the _envoy_ container
  * add environment  variable _ENABLE_ENVOY_XRAY_TRACING_ with value _1_
  * enable Cloudwatch logging
* click on _Create_ to create the new task definition revision

#### ECS service
* recreate ECS service
* create new security group, open port 80 from everywhere
* click _Enable service discovery integration_
  * select existing namespace
  * select existing service discovery service
  * select _product-service_
* click _Next step_
* click _Next step_
* click _Create service_

### payment service

#### Taskdefinition

* enable AppMesh integration by clicking checkbox _Enable App Mesh integration_
  * select _paymentapp_ as application container name
  * select _frontendapp-mesh_ as Mesh name
  * select _payment-service-vn_ as Virtual node name
  * **click Apply** !
  * click _Confirm_
* in Container definitions, open the _envoy_ container
  * add environment  variable _ENABLE_ENVOY_XRAY_TRACING_ with value _1_
  * enable Cloudwatch logging
* click on _Create_ to create the new task definition revision


## apply task definitions

```bash
cd 4-AppMesh
aws ecs register-task-definition  --cli-input-json file://td-productapp-appmesh.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-paymentapp-appmesh.json --region us-west-2
aws ecs register-task-definition  --cli-input-json file://td-frontendapp-appmesh.json --region us-west-2
```


export CORE_STACK_NAME="ecs-course-core-infrastructure" 


ecs-cli up \
--subnets subnet-06620e92772d7d0bc, subnet-0558b5109de70d1b3 \
--vpc vpc-0aad0e33a58c19a14 \
--launch-type EC2 \
--size 1
--instance-type t2.small \
--cluster ecs-ec2







create a linux ec2 instance
sudo yum install docker
sudo yum install git
sudo systemctl start docker
sudo systemctl status docker
sudo systemctl enable docker
Clone your repo (E-commerce)
ls
do ls -al
cd into ecommerce-app-source-code
ls
cd product
do ls
	*** Docker Images Creation ***
# Replace hodalo by your docker registry username
sudo docker build -t hodalo/productapp:latest .
switch to root (sudo su)
- then docker images (to see the newly build image)
- cd ../paymentapp/
- ls
- docker build -t hodalo/paymentapp:latest .
- docker images
- cd ../frontendapp/
- ls
- docker build -t hodalo/frontendapp:latest .
- docker images
(ignore the error message)
# check your CLI has been configured
aws s3 ls
# If not
aws configure
	*** IAM Role Creation ***
				
- Navigate to IAM
- Create roles
- Select AWS service
- Use case: ec2 role for elastic container service (last option)
- Next, Next
- Role name: ecsInstanceRole
- Create role
	*** CloudFormation Stacks Creation ***
	
1- Navigate to your Github and Copy the content of create_IAM_roles.yml from ecs-project-cluster-setup/1.create_IAM_roles
Navigate back to CloudFormation and create a stack named ecs-project-iam-roles
2- Navigate to your Github and Copy the content of core-infrastructure-setup.yml from ecs-project-cluster-setup/2.Core-infrastructure-setup
Navigate back to CloudFormation and create a stack named ecs-project-infrastructure-setup
3- Navigate to your Github and Copy the content of alb-external.yml from ecs-project-cluster-setup/3.loadbalancer/ALB
Navigate back to CloudFormation and create a stack named ecs-project-loadbalancer
4- Navigate to your Github and Copy the content of ecs-ec2-via-cloudformation.yml from Prof's ecs-project-cluster-setup/CloudFormation-based
Navigate back to CloudFormation and create a stack named ecs-project-cluster-env
- Navigate to IAM
- IAM roles
- Search and open ecsTaskExecution  role
- Copy role's ARN
- Navigate and open the Ecommerce-app-deploy-project-ecs repository on your local (vscode)
- Navigate and open the microservices-td
- Update the 3 files names to have <td-frontendapp-setup.json>, <td-paymentapp-setup.json>, and <td-productapp-setup.json>
- In the td-productapp-setup.json:
	. replace the role's ARN copied by the one on line 2 and 3
	. line 13: "image": "pauloclouddev/flamencoapp:latest",
	. line 22: "name": "productapp"
	. line 25: "family": "td-productapp",
- In the td-paymentapp-setup.json:
	. replace the role's ARN copied by the one on line 2 and 3
	. line 13: "image": "pauloclouddev/operaapp:latest",
	. line 22: "name": "paymentapp"
	. line 25: "family": "td-paymentapp",
- In the td-frontendapp-setup.json:
	. replace the role's ARN copied by the one on line 2 and 3
	. line 21: "image": "pauloclouddev/musicboxapp:latest",
	. line 31: "name": "frontendapp"
	. line 34: "family": "td-frontendapp",
- cd Ecommerce-app-deploy-project-ecs/
# run these commands from the README.md file by replacing this region with your default region
- aws ecs register-task-definition  --cli-input-json file://td-productapp-setup.json --region us-east-1
- aws ecs register-task-definition  --cli-input-json file://td-paymentapp-setup.json --region us-east-1
- aws ecs register-task-definition  --cli-input-json file://td-frontendapp-setup.json --region us-east-1
	*** Create an ECS Service ***
	
- Navigate to ECS
- Open the  ecs-project-ec2
- Create service
- Launch type: EC2
- Task Definition: td-productapp
- Service Name: productapp-service
- Number of tasks: 1
- Leave everything else default
- Next
- VPC: choose the one with CICR 172.16.0.0/16
- sUBNETS: BOTH
- Security group: Edit
	. name: productapp-service-sg
	. type: Custom TCP, pORT 9001 to Anywhere
- Next, Next, Create service, View service
- Once running, copy Private IP Id
	*** Create a task definition ***
- Task Definition
- Open frontendapp
- select latest revision
- create new revision
- scroll all the way down and click the container name (productapp-service)
- Replace the <product_host> place holder with the Private IP copied to have something like 172.0.0.0:9001
REPEAT STEPS TO CREATE  paymentapp-service (port 9002) with its task definition revisionon, frontendapp-service (port 9000) with its task definition revision.
