[Modernizing SFTP services]{.ul}

[Context]{.ul}

As an AWS Solution Architect in DXC Technology, when moving on premises
workload to AWS, we can see many clients having somewhere in their
datacenters usually a small box, but a very important one that is
dealing with exchanging files between applications and/or third parties.
This box is usually serving as SFTP file server or even sometimes still
as simple FTP Server.

It is not the core part of client applications but if it's not well
designed and managed, it can lead to serious issues, delaying the
launch/go live date, and affecting the overall migration.

In this blog post we'll discuss how we can address those topics.

The first reaction is usually "Well let's put a small T2 instance and
that would be right for now". Going further in the discussion with the
client, some additional challenges might appear.

[The business requirements]{.ul}

-   Support critical business operations: The solution needs to be
    highly available & highly scalable, as exchange with other
    applications or third parties can be critical in the overall
    availability of the services

-   Support of new services: Current exchange protocols are in SFTP/FTP
    but client would be able to offer new services with new types of
    protocols based on HTTP(S) protocol for example.

-   Automation of tasks: The client is aware that a lot of automation
    can be easily performed in the Cloud and particularly in AWS, and
    they would like to have some automation happening when there is a
    new file arriving, like putting in and Amazon EFS Share file,
    notifying mail lists and so on.

We will first cover the proposed solution, then deep dive in the key
steps of the AWS CloudFormation Code, and finally see some potential
evolution for further stages.

[Prerequisites]{.ul}

For this walkthrough, you should have the following prerequisites:

-   An AWS Account.

-   Administrative rights on that account.

To deploy the solution:

Download locally on your workstation the AWS CloudFormation Template
from here:
https://github.com/rsimonfr/awstransfer/blob/master/sftpserviceslambdaefs.yaml

Go to AWS Console and select AWS CloudFormation as a service.

Be sure to be in **eu-west-1** (AWS Ireland Region). It should work on
other regions but it was not tested.

Then click on Create a Stack with new resources

![](media\image1.png){width="3.0574890638670165in"
height="1.1033552055993001in"}

Then select the file you have downloaded as the template file

![](media\image2.png){width="6.5in" height="2.9590277777777776in"}

Click Next

Then give a Name to the stack and be sure you selected 2 different AZ in
the selected region.

You can leave the rest as is or change the values according to your
needs.

![](media\image3.png){width="6.5in" height="5.617361111111111in"}

Click Next

On Configure stack options tab click Next

On Review tab, check your settings, click on "I acknowledge that AWS
CloudFormation might create IAM Resources with custom names" and then
Click on Create Stack.

![](media\image4.png){width="6.5in" height="1.8798611111111112in"}

It will take some 5 minutes to complete.

[The solution]{.ul}

![](media\image5.png){width="6.083333333333333in"
height="4.751628390201224in"}

The proposed solution is based on the following AWS building blocks:

-   Amazon S3 to provide highly durable and highly scalable storage.

-   AWS Transfer Family is a Managed Service to provide SFTP Endpoints
    that are presenting S3 Buckets to end client and performs
    authentication. We will use Private Endpoints that allows only
    private connections to the SFTP Server. Note that FTP access is also
    possible in private mode, but I would definitively not recommend it.

-   Amazon Route 53 is an DNS Managed Services that provides Amazon
    Route 53 Private Hosted Zones & Amazon Route 53 resolvers to allow
    the usage of user-friendly names

-   Automation on reception of new file:

    -   Amazon S3 publish to a SNS Topic. Amazon SNS is a Managed
        Service used to publish/subscribe Messages between different
        components and allows decorrelation & scalability between
        components.

    -   An Amazon SNS Function is subscribing to this SNS Topic and copy
        newly created file to an Amazon EFS mount. AWS Lambda allow
        running code without infrastructure. Here we don't need
        dedicated always running EC2 Instances to perform the copy. We
        will only pay when a file will arrive. Amazon EFS is AWS Managed
        Service to provision highly scalable NFS Mounts.

![](media\image6.png){width="6.5in" height="2.2319444444444443in"}

All the resources will be provisioned with an AWS CloudFormation
template. It ensures that our deployment is repeatable the same way on
different accounts and regions, and that in the end we will be able to
clean up everything easily.

**[First Step : Let's create a VPC & Private Hosted Zone]{.ul}**

We will create a private only VPC in 2 Availability Zones with no Access
to the Internet. We also create a Route53 Private Hosted Zone with
myexample.com DNS Name.

**[Step 2: Creation of Amazon S3 bucket]{.ul}**

We create the Amazon S3 bucket which will be the backend of the SFTP
Server. It will host the files for the Sftp server.

We ensure this bucket is not publicly accessible and is encrypted a
Customer KMS Key.

Finally, we are declaring a Notification Configuration. Every time a new
file is put in root folder, we will publish an event in a SNS topic that
we will show later in the Automation part.

NotificationConfiguration:

TopicConfigurations:

\- Event: \'s3:ObjectCreated:\*\'

Filter:

S3Key:

Rules:

\- Name: \"prefix\"

Value: \"root/\"

Topic: !Ref SftpS3SnsTopic

**[Step 3 : Creation of AWS Transfer & all related required
resources]{.ul}**

First, let's create the SFTP Server.

We want to keep it fully private. We need to reference the VPC Endpoint
Id and specify the endpoint type to **VPC_ENDPOINT.**

We will use here the default Identity Provider type
(**SERVICE_MANAGED**). The users will be managed by the AWS Transfer
Service and support access through SSH keys. You can also use external
Identity provider like Active Directory (see the documentation there
[https://docs.aws.Amazon.com/transfer/latest/userguide/authenticating-users.html](https://docs.aws.amazon.com/transfer/latest/userguide/authenticating-users.html)
)

We want to log all the user activity (connection / downloads / upload).
We then specify a Logging Role so that AWS Transfer Service can write on
Amazon CloudWatch Log.

SftpServer:

Type: AWS::Transfer::Server

Properties:

EndpointDetails:

VpcEndpointId: !Ref SftpVPCEndpoint

EndpointType: VPC_ENDPOINT

IdentityProviderType: SERVICE_MANAGED

LoggingRole: !GetAtt SftpLogRole.Arn

Tags:

\- Key: Name

Value: SftpServer

Let's add the missing pieces to have the full configuration.

We create the Sftp VPC Endpoint that our users will connect to.

SftpVPCEndpoint:

Type: AWS::EC2::VPCEndpoint

Properties:

SecurityGroupIds:

\- !Ref SftpVPCEndpointSecurityGroup

ServiceName: !Sub com.Amazonaws.\${AWS::Region}.transfer.server

SubnetIds:

\- !Ref SubnetA

\- !Ref SubnetB

VpcEndpointType: Interface

VpcId: !Ref VPC

SftpVPCEndpointSecurityGroup:

Type: AWS::EC2::SecurityGroup

Properties:

GroupDescription: Sftp Endpoint Security Group

VpcId: !Ref VPC

SecurityGroupIngress:

\- IpProtocol: tcp

FromPort: 22

ToPort: 22

CidrIp: 10.0.0.0/8

Tags:

\- Key: Name

Value: !Sub SG-\${AWS::StackName}

As we want to have a user-friendly name, we declare a record in Route53.

SftpDNSRecord:

Type: \"AWS::Route53::RecordSetGroup\"

Properties:

HostedZoneId: !Ref DNSPHZ

RecordSets:

\- Name: !Sub mysftp.\${DNSPHZName}

Type: A

AliasTarget:

HostedZoneId: !Select \[0, !Split \[\":\", !Select \[0, !GetAtt
SftpVPCEndpoint.DnsEntries\]\]\]

DNSName: !Select \[1, !Split \[\":\", !Select \[0, !GetAtt
SftpVPCEndpoint.DnsEntries\]\]\]

Finally, we declare the Log Role so that AWS Transfer will be able to
push connection log to Amazon CloudWatch logs & we create a log group to
store the logs with the AWS IAM role and policy

SftpLogGroup:

Type: AWS::Logs::LogGroup

Properties:

LogGroupName: !Sub /aws/transfer/\${SftpServer.ServerId}

RetentionInDays: 90

SftpLogPolicy:

Type: AWS::IAM::ManagedPolicy

Properties:

ManagedPolicyName: !Sub SftpLogPolicy-\${AWS::Region}

Description: Sftp Log access policy

PolicyDocument:

Version: \'2012-10-17\'

Statement:

\-

Effect: Allow

Action:

\- \"logs:CreateLogGroup\"

\- \"logs:CreateLogStream\"

\- \"logs:PutLogEvents\"

\- \"logs:DescribeLogStreams\"

Resource: !Sub
arn:aws:logs:\${AWS::Region}:\${AWS::AccountId}:log-group:/aws/transfer/\*:\*

SftpLogRole:

Type: AWS::IAM::Role

Properties:

AssumeRolePolicyDocument:

Version: \'2012-10-17\'

Statement:

\- Effect: \'Allow\'

Principal:

Service:

\- \'transfer.Amazonaws.com\'

Action:

\- \'sts:AssumeRole\'

ManagedPolicyArns:

\- !Ref SftpLogPolicy

We need also a sftp user to connect.

We set its home directory: be careful with the SFTP style declaration.

We set the role, the AWS Transfer service will assume for the user, and
the SftpServer ID the user will connect to.

SftpUser:

Type: AWS::Transfer::User

Properties:

UserName: sftpuser

HomeDirectory: !Sub /mysftp-\${AWS::AccountId}\${AWS::Region}/root

Role: !GetAtt SftpAccessRole.Arn

ServerId: !GetAtt SftpServer.ServerId

And so, we need to define the SftpAccessRole that AWS Transfer will
assume for the user.

SftpAccessRole:

Type: AWS::IAM::Role

Properties:

AssumeRolePolicyDocument:

Version: \'2012-10-17\'

Statement:

\- Effect: \'Allow\'

Principal:

Service:

\- \'transfer.Amazonaws.com\'

Action:

\- \'sts:AssumeRole\'

ManagedPolicyArns:

\- !Ref SftpAccessPolicy

This role has the SftpAccessPolicy attached, which give the required
rights to put, get and delete files in the root folder of the bucket. As
our bucket is encrypted, we also need to allow Encryption / Decryption
with the KMS Key.

SftpAccessPolicy:

Type: AWS::IAM::ManagedPolicy

Properties:

ManagedPolicyName: !Sub SftpAccessPolicy-\${AWS::Region}

Description: Sftp access policy

PolicyDocument:

Version: \'2012-10-17\'

Statement:

\-

Effect: Allow

Action:

\- \'s3:PutObject\'

\- \'s3:GetObject\'

\- \'s3:DeleteObject\'

\- \'s3:GetObjectVersion\'

\- \'s3:DeleteObjectVersion\'

Resource: !Sub arn:aws:s3:::mysftp-\${AWS::AccountId}\${AWS::Region}/\*

\- Effect: Allow

Action:

\- kms:Encrypt

\- kms:Decrypt

\- kms:ReEncrypt\*

\- kms:GenerateDataKey

\- kms:DescribeKey

Resource: \"\*\"

\-

Effect: Allow

Action:

\- \'s3:ListBucket\'

\- \'s3:GetBucketLocation\'

Resource: !Sub arn:aws:s3:::mysftp-\${AWS::AccountId}\${AWS::Region}

Condition:

StringLike:

\'s3:prefix\': \'root/\*\'

**[Step 4: Automation when a file is put on Amazon S3 bucket]{.ul}**

For quite old applications, manipulating files in Amazon S3 bucket may
be not easy. A good way to share files between servers, is to use an
Amazon EFS drive. So, let's copy the file as soon as it arrives on
Amazon S3.

The best candidate to do that is an AWS Lambda function. AWS Lambda is
serverless and highly available by design, so we don't have to provision
any instance to perform this activity.

The AWS Lambda function needs to have access to Amazon EFS and to the
VPC in which it is hosted.

First let's create the AWS Lambda function. As we are going to connect
to an Amazon EFS drive we need to specify on which subnets we are
executing the function with which Security Group. We also need to
specify the mount point that will use the function. This has to be under
/mnt and so can be very different from the one you are using on your
application server. Here we are using /mnt/sftp and we need to specify
the specific mount point we will use to access the Amazon EFS mount.

The Python code below, retrieves from the SNS message the bucket and the
object name. Then it recreates the directory structure, and will copy
the file directly from Amazon S3 to Amazon EFS with Amazon S3 download
file API call.

S3toEFSCopyLambda:

Type: AWS::Lambda::Function

Properties:

FunctionName: S3toEFSCopy

Runtime: python3.7

Role: !GetAtt S3toEFSCopyLambdaRole.Arn

Timeout: 300

Handler: index.lambda_handler

VpcConfig:

SecurityGroupIds:

\- !Ref AppSG

SubnetIds:

\- !Ref SubnetA

\- !Ref SubnetB

FileSystemConfigs:

\- LocalMountPath: \"/mnt/sftp\"

Arn: !GetAtt EFSAccessPoint.Arn

Code:

ZipFile: \|

import json

import boto3

from pathlib import Path

def lambda_handler(event, context):

message = event\[\'Records\'\]\[0\]\[\'Sns\'\]\[\'Message\'\]

y = json.loads(message)

bucket=y\[\'Records\'\]\[0\]\[\'s3\'\]\[\'bucket\'\]\[\'name\'\]

print(bucket)

object=y\[\'Records\'\]\[0\]\[\'s3\'\]\[\'object\'\]\[\'key\'\]

objectname=object.split(\"/\")\[-1\]

print(object)

s3_client = boto3.client(\'s3\')

local_file = \'/mnt/sftp/\'+object

object_dir= \'/mnt/sftp/\'+object.rstrip(objectname)

print(object_dir)

Path(object_dir).mkdir(parents=True, exist_ok=True)

print(local_file)

response = s3_client.download_file(bucket, object, local_file)

print(response)

We need to create the topic to which Amazon S3 bucket is publishing.
This topic will send a message when a new file has arrived. We need to
define a policy that will allow Amazon S3 to push event, and the AWS
Lambda function to subscribe to the topic.

SftpS3SnsTopic:

Type: AWS::SNS::Topic

Properties:

TopicName: SftpS3SnsTopic

SftpS3SnsTopicPolicy:

Type: AWS::SNS::TopicPolicy

Properties:

PolicyDocument:

Id: SftpS3SnsTopicPolicy

Version: \'2012-10-17\'

Statement:

\- Sid: SftpS3SnsTopicPolicy

Effect: Allow

Principal:

AWS: !Sub arn:aws:iam::\${AWS::AccountId}:root

Action: sns:Subscribe

Resource: !Ref SftpS3SnsTopic

\- Sid: S3 Policy

Effect: Allow

Principal:

Service: s3.Amazonaws.com

Action: sns:Publish

Resource: !Ref SftpS3SnsTopic

Condition:

ArnLike:

aws:SourceArn: !Sub
arn:aws:s3:::mysftp-\${AWS::AccountId}\${AWS::Region}

Topics:

\- !Ref SftpS3SnsTopic

Now we create the Amazon EFS access point. In real life, be careful with
UID and GID and they need to fit with your Amazon EFS configuration

EFSAccessPoint:

Type: \'AWS::EFS::AccessPoint\'

Properties:

FileSystemId: !Ref EFSID

PosixUser:

Uid: \"10002\"

Gid: \"10002\"

RootDirectory:

CreationInfo:

OwnerGid: \"10002\"

OwnerUid: \"10002\"

Permissions: \"0770\"

Path: \"/\"

The AWS Lambda function requires the following rights:

-   Ability to mount and write files on the mount

-   Ability to create / delete Network interfaces in the VPC

-   Ability to read file on the S3 bucket (we don't need write rights
    here)

-   Ability to subscribe to the SNS topic

-   Ability to create logs in Amazon CloudWatch

-   Ability to decrypt file on the Amazon S3 buckets (we don't need
    encrypt as we don't need to write files)

S3toEFSCopyLambdaRole:

Type: AWS::IAM::Role

Properties:

RoleName: !Join \[\"-\", \[\"S3toEFSCopyLambda\", Ref:
\"AWS::Region\",\]\]

AssumeRolePolicyDocument:

Version: 2012-10-17

Statement:

\- Action:

\- \'sts:AssumeRole\'

Effect: Allow

Principal:

Service:

\- lambda.Amazonaws.com

Path: /

Policies:

\- PolicyName: S3toEFSPolicy

PolicyDocument:

Version: \'2012-10-17\'

Statement:

\- Action:

\- \"elasticfilesystem:ClientMount\"

\- \"elasticfilesystem:ClientWrite\"

\- \"elasticfilesystem:DescribeMountTargets\"

Effect: Allow

Resource: \"\*\"

\- Action:

\- \'s3:Get\*\'

\- \'s3:List\*\'

Effect: Allow

Resource: \"\*\"

\- Action:

\- sns:Subscribe

Resource: \"\*\"

Effect: \"Allow\"

\- Action:

\- ec2:CreateNetworkInterface

\- ec2:DeleteNetworkInterface

\- ec2:DescribeNetworkInterfaces

Resource: \"\*\"

Effect: \"Allow\"

\- Effect: Allow

Action:

\- logs:CreateLogGroup

Resource:

\- !Sub arn:aws:logs:\${AWS::Region}:\${AWS::AccountId}:\*

\- Effect: Allow

Action:

\- logs:CreateLogStream

\- logs:PutLogEvents

Resource:

\- !Sub
arn:aws:logs:\${AWS::Region}:\${AWS::AccountId}:log-group:/aws/lambda/S3toEFSCopy:\*

\- Effect: Allow

Action:

\- kms:Encrypt

\- kms:Decrypt

\- kms:ReEncrypt\*

\- kms:GenerateDataKey

\- kms:DescribeKey

Resource:

\- !GetAtt S3SFTPKey.Arn

Amazon SNS Service should be granted the right to invoke the AWS Lambda
function

S3toEFSCopypermission:

Type: AWS::Lambda::Permission

Properties:

FunctionName: !GetAtt S3toEFSCopyLambda.Arn

Principal: sns.Amazonaws.com

Action: lambda:InvokeFunction

SourceArn: !Sub
arn:aws:sns:\${AWS::Region}:\${AWS::AccountId}:\${SftpS3SnsTopic}

Finally, we create the Subscription to the SNS topic for the AWS Lambda
function

S3toEFSCopySNSSubscription:

Type: AWS::SNS::Subscription

Properties:

Endpoint: !GetAtt S3toEFSCopyLambda.Arn

Protocol: lambda

TopicArn: !Sub
arn:aws:sns:\${AWS::Region}:\${AWS::AccountId}:\${SftpS3SnsTopic}

[Testing]{.ul}

To verify that all works please refer to the Testing section in the
README.md file in the repo.

[Further developments]{.ul}

As files are on Amazon S3, more automation can be done like :

-   setting up a vulnerability scan with a ClamAV Lambda function (see
    this excellent post
    <https://engineering.upside.com/s3-antivirus-scanning-with-lambda-and-clamav-7d33f9c5092e?gi=b9c49071620e>
    on how to do that),

-   Amazon S3 replication to perform disaster in another region.

-   Now that AWS Event Bridge can be a source for Amazon S3 we could
    replace perhaps Amazon SNS.

[Cleanup]{.ul}

To delete all resources named in this post, just delete the AWS
CloudFormation Stack. Before doing that delete all the files you may
have in your Amazon S3 bucket, otherwise AWS CloudFormation will refuse
to delete it.
