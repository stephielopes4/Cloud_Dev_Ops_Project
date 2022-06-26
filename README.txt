# Cloud-Project-2-Anomaly Detection using CloudFormation and Codedeploy

#### Description:
	This system makes use of AWS services----CloudFormation and CodeDeploy to automate the anomaly detection of a patient's body temperature data.
The temperature data of a thing(device) 'DHT-001' is streamed through Kinesis data stream. This data is pushed on kinesis data stream by Ubuntu EC2 instance.
AWS Lambda function is triggered by the kinesis data. This function detects the temperature anomalies(any temperature below 96 is cold temperature and
any temperature above 101 is fever/hot temperature) and stores the anomaly data records in DynamoDB table and sends the notification through SNS service 
via email to the subscriber. All the resources are provisioned through a template(JSON template) with parameters and this template is uploaded on CloudFormation
to create a stack. CodeDeploy deploys the application revision file(archive .zip file) containing appspec.yml file, scripts folder and service folder 
containing script to push the data on kinesis stream. CodeDeploy deploys the application revision file on Ubuntu server. Once the data streaming begins, 
the Lambda is invoked for subsequent operations.


##Components:
	AWS Services  - CloudFormation, CodeDeploy
	JSON Template for (EC2, Kinesis, Dynamodb, SNStopic, TopicSubscription, S3Bucket, Lambda Function, EventSourceMapping for Lambda Trigger)
	Visual Studio Code - IDE

##Set Up & Modules, Packages :
	Ubuntu EC2 server logged in through putty in Windows and Code Deploy agent is installed on it.
	ruby-full and wget for code deploy agent
	python3, pip3.
	AWS SDK for Python ---  Boto3 (the package implementing the Python SDK itself)
	boto3-----to access the service client(kinesis, sns) and resource(dynamodb)
	AWS CLI to update the lambda function
	
##Languages used:
	Python--------for data push script(raw_data.py) and anomaly_detection.py(lambda handler)
	JSON--------to write template for cloudformation
	YML--------to write application specification file(appspec.yml) required for codedeploy.
	bash shell script------to write the application scripts for application 'hooks'

##AWS SERVICES USED:
1)CloudFormation----to upload the JSON template, fill in the parameters and create a stack.

2)CodeDeploy-------takes the role of ec2 server and fetches the appliction revision(.zip file---- m03p02_codedeploy.zip) from s3 bucket. 
	mo3p02_codedeploy.zip contains 3 files/folders:
	a) service folder-------this folder contains raw_data.py file which is a source code, used to push the data on kinesis stream.
	b) scripts folder-----this folder contains  bash shell scripts (specific to Linux - here, our Ubuntu AMI) that run various stages of the deployment process.
	c)appspec.yml-------The appspec.yml file (A YAML file) is what the CodeDeploy Agent executes on the EC2 instance when the deployment needs to be made.

3)lambda-deploymentpackage-bucket------this bucket is created in S3 where the lambda_deployment_package.zip is stored which is the code for the lambda function. 
	This .zip files contains anomaly_detection.py script which has lambda_handler() method to detect anomalies, tore them in table and send emails.

### FILES ###:
1) Cloud_Formation.json:
             This file first creates an Ubuntu server 18.04 LTS in region us-east-1 with resource as AWS::EC2::Instance and the name of the server is 			     AnomalyDetectionServer.
             The parameters are first created for KeyName, InstanceType, VPCId and SUBNETId and allowed values are given inside a list.
	     PARAMETERS AS FOLLOWS:
		KeyName-----gives all the available or created key pairs in the drop down list, any key can be chosen for ec2 server.
		InstanceType---------available instance types for my account are given in drop down list to choose from
		VPCId -------default vpc id for my account is given
		SUBNETId------for above VPC,  6 subnets are available to choose from
	MAPPINGS:
		Mappings is created to map the instance type with architecture for eg. t2.micro maps to HVM64 architecture. This architecture further maps with the                     region us-east-1 with correct ami id for the ubuntu server. We can thus create for any region the required instances. For my account US-East-1 region 		      is mandatory.
		 
	RESOURCES:
	1) ServerSecurityGroup--------of type "AWS::EC2::SecurityGroup is created with both inbound and outbound rules for VPCId mentioned above
	
	2)AnomalyDetectionServer-------This server takes the instance type parameter, keyname parameter, subnetid parameter. The reference for above security group is 			given for network interface. Also, for ImageId, the function Find In Map:: is used which referes to the Mapping given above and correctly takes the 
		 ami id for ubuntu server. Once the server is create through CloudFormation, then on this server code deploy agent is installed. This agent installs  
		 the source code(raw_data.py) , appspec.yml and scripts on the server. Using the scripts the raw_data.py is executed and data push to kinesis stream  
		 begins.

	
	3)KinesisDataStream--------of type AWS::Kinesis::Stream is created for streaming of raw temperature data.
	
	4)DynamoDBTable-------of type AWS::DynamoDB::Table  is created to store the anomaly data
	
	5)SNSTopic------of type AWS::SNS::Topic is created to create a topic to subscribe
	
	6)MySubscription-----of type AWS::SNS::Subscription is created with protocol as email to ask the subscriber to subscribe to the above topic to get the email-  			notification.
	
	7) S3Bucket------of type AWS::S3::Bucket is created for storing the deployment package(.zip file for source code and appspec.yml)
	
	8)KinesisEventSourceMap-------of type AWS::Lambda::EventSourceMapping is created for a trigger(kinesis data stream) for lambda function
	
	9)AnomalyLambdaFunction------of type AWS::Lambda::Function is created for lambda_handler. This function takes .zip package. The lambda execution role is 		created and arn is given to Role property. The code for the function will be uploaded from local machine as .zip file. This code is deployed from 		       command line by navigating to the project directory. Here, lambda_deployment_package.zip file is created and anomaly_detection.py file which contains 		    lambda_handler is added at the root of this zip file. The S3 Bucket by name lambda-deploymentpackage-bucket is seperately created in which this
	       lambda_deployment_package.zip is already uploaded. In the "Code" property of AWS::Lambda::Function, the "S3bucket" name--lambda-deploymentpackage-bucket
	       and the "S3key"---lambda_deployment_package.zip is given for the lambda function to deploy the anomaly_detection.py code and access 			       anomaly_detection.lambda_handler method for handler.	   
			
	10)OUTPUTS:
	          outputs for following resources are given-----
		1.InstanceId-------instance id for AnomalyDetectionServer
		2.Name-----data stream name for KinesisDataStream
		3.TableName------table name for DynamoDBTable
		4.ARN-------arn for SNSTopic
		5.ID-------id for MySubscription
		6.BucketName-------name for S3Bucket which is used in code deploy
		7.FunctionName---------the name of lambda function for AnomalyLambdaFunction

2)lambda_deployment_package.zip-------this is a zip file containing anomaly_detection.py(which has lambda_handler) which is uploaded in S3 bucket and the bucket name
				      and s3key which is .zip package is given to the lambda resource.

3)m03p02_codedeploy.zip-----------this file contains the code deployment package,  a .zip archive file. There are 3 folders in this zip file.
	a) service folder-------this folder contains raw_data.py file which is a source code used to push the data on kinesis stream.
	b) scripts folder-----this folder contains  bash shell scripts (specific to Linux - here, our Ubuntu AMI) that run various stages of the deployment process.
	c)appspec.yml-------The appspec.yml file (A YAML file) is what the CodeDeploy Agent executes on the EC2 instance when the deployment needs to be made.

###Credits & References:
Cloud DEV_OPS videos, lectures
AWS resource and property types reference
Week 1, 2, 3 notebooks
AWS CodeDeploy
Deploy Python Lambda functions with .zip file archives




	
	
	

