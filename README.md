## Notes about course
### Links
* [Resources and types of resources](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

* [EC2 Instance Resource Type](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html)

* [CloudFormation Drift](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html)

* [Great nested stacks example](https://github.com/aws-samples/ecs-refarch-cloudformation/blob/master/master.yaml)

### Frequently Questions
#### I - About resources
**1 -** Can I create a dynamic number of resources ?
- Yes, you can by using Cloudformation Macrons and Transform

**2 -** Is every AWS Service supported?
- Almost. Only a select few niches are not there yet
- You can work around that using CloudFormation Custom Resources

## Notes

### 1 - Policys

#### Deletion Policy
-> Control what happens when the CloudFormation template is deleted or when a resource is removed directly from a CloudFormation template
-> Use with any resource
- **DeletionPolicy=Retain:**
  - Specify on recources to preserve / backup in case of CloudFormation deletes
  - To keep a resource, specify Retain (works for any resource / nested stack)
- **DeletionPolicy=Snapshot:**
-   EBS Volume, ElastiCache Cluster, ElastiCache ReplicationGroup
-   RDS DBInstance, RDS DBCluster, Redshift Cluster, Neptune DBCluster
- **DeletionPolicy=Delete (default behavior):**
  - Note: for AWS::RDS::DBCluster resources, the default policy is Snapshot
  - Note: for AWS::RDS::DBInsteance resources thas don't specify the DBClusterIdentifier property, the default policy is Snapshot
  - Note: to delete an S3 bucket, you need to first empty the bucket of its content

#### UpdateReplace Policy
-> Controls what happens to a resource if you update a property whose update behavior is **Replacement**
-> For example, updating RDS DBInstance's AvailabilityZone porperty
-> Use with any resource
- **UpdateReplacePolicy=Delete(default behavior):**
  - CloudFormation deletes the old resource and creates a new one with a new physical ID
- **UpdateReplacePolicy=Retain:**
  - Keeps the resource (it is removed from CloudFormation's scope)
- **UpdateReplacePolicy=Snapshot:**
  - EBS Volume, ElastiCache Cluster, ElastiCache ReplicationGroup
  - RDS DBInstance, RDS DBCluster, Redshift Cluster, Neptune DBCluster
  - The snapshot doesn't exist in CloudFormation's scope

### 2 - Mappings
-> Mappings are fixed variables within your CloudFormation template

-> They're very handy to differentiate between diferent environments (dev vs prod), regions (AWS regions), AMI types, etc.
- Example:

  ```yaml
    Mappings:
      Mapping01:
        key01:
          Name: Value01
        key02:
          Name: Value02
        key03:
          Name: Value03
  ```
 **When would you use Mappings vs Parameters?**
  - Mappings are great when tou know in advande all the values that can be taken and that they can be deduced from variables such as
    - Region
    - Availability Zone
    - AWS Account
    - Environment (dev vs prod)
    - etc...
  - They allow safer control over the tamplate
  - use parameters when the values are really user specific


#### Accessing Mapping Values (Fn::FindInMap)
-> We use Fn::FindInMap to return a named value from a specific key
-> !FindInMap [MapName, TopLevelKey, SecondLevelKey]

Example:
```yaml
Parameters:
  EnvironmentName:
    Description: Environment Name
    Type: String
    AllowedValues: [development, production]
    ConstraintDescription: must be development or production

  Mappings:
  # instance images available
  AWSRegionArch2AMI:
    af-south-1:
      HVM64: ami-06db08e8636583118
    ap-east-1:
      HVM64: ami-0921e2da2f22f9617
    ap-northeast-1:
      HVM64: ami-06098fd00463352b6
    ap-northeast-2:
      HVM64: ami-07464b2b9929898f8
  # instance types available
  EnvironmentToInstanceType:
    development:
      instanceType: t2.micro
    # we want a bigger instance type in production
    production:
      instanceType: t2.small

Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !FindInMap [EnvironmentToInstanceType, !Ref 'EnvironmentName', instanceType]
      # Note we use the pseudo parameter AWS::Region
      ImageId: !FindInMap [AWSRegionArch2AMI, !Ref 'AWS::Region', HVM64]
```

#### Concept: Pseudo Parameters

- AWS offers us pseudo parameters in any CloudFormation template
- These can be used at any time and are enabled by default

   | Reference Value | Example Returned Value |
   |---|---|
   | AWS::AccountId | 1234567812 |  
   | AWS::Region | us-east-1 |  
   | AWS::StackName | arn:aws:cloudformation:us-east-1:1234567812:stack/MyStack/1c2fa620-982a11e3-aff7-50e241623849e0 |  
   | AWS::NotificationARNs | [arm:aws:sns:us-east-1:1234567812:MyTopic] |  
   | AWS::NoValue | Doesn't return a value |  
   | AWS::Partition | Standard AWS region -> aws / Other regions -> aws-partitionname (ex.: China region -> aws-cn) |  
   | AWS::URLSuffix | amazonaws.com (ex. China region -> amazonaws.com.cn) |  

### 3 - Outputs
-> As a name indicates, outputs are optional outputs values that we can import into other stacks. So we first need to export the outputs from a CloudFormation stack and another one can reference them.

- The Outputs section declares optional outputs values that we can import into others stacks (if you export them first) !
- You can also view the outputs in the AWS Console or in using the AWS CLI
- They're very useful for example if you define a network CloudFormation, and output the variables such as VPC ID and your Subnet IDs.
- It's the best way to peform some collaboration cross stack, as you let expert handle their own part of the stack.

### 4 - Conditions
Consitions are used to allow, conditionally, the creation of resources or outputs when based on a condition.

- Conditions are used to control the creation of resources or outputs based on a condition.
- Conditions can be whatever you want them to be, but common ones are:
  - Environments (dev / test / prod)
  - AWS Region
  - Any parameter value
- Each condition can reference another condition, parameter value or mapping

#### How to define condition

```yaml
  Conditions:
    CreateProdResources: !Equals [ !Ref Envtype, prod ]
```
- The logical ID is for you to choose. It's how you name condition.
- The intrinsic function (logical) can be any of the following:
  - Fn::And
  - Fn::Equals
  - Fn::If
  - Fn::Not
  - Fn::Or

#### Using a Condition
- Conditions can be applied to resources / outputs / etc..

```yaml
  Resources:
    MoutPoint: 
      Type: AWS::EC2::VolumeAttachment
      Condition: CreateProdResouurces
```
#### Fn::GetAtt
- Attributes are attached to any resources you create
- To know the attributes of your resources, the best place to look at is the documentation
- For example: the AZ of an EC2 machine!

```yaml
  Resources:
    EC2Instance: 
      Type: AWS::EC2::Instance
      Properties:
        ImageId: ami-0742b4e673072066f
        InstanceType: t2.micro
```
```yaml
  NewVolume:
    Type: AWS::EC2::Volume
    Condition: CreateProdResources
    Properties:
      Size: 100
      AvailabilityZone: !GetAtt EC2Instance.AvailabilityZone
```
### Rules
#### **1 -** What are Rules used for ?
- Parameters section gives us ability to validate within a single parameter (Type, Min/MaxValue, AllowedValues, AllowedPattern)
- **Rules** used to perform parameter validations based on the values of other parameters (cross-parameter validation)
- For example: ensure that all subnets selected are within the same VPC

#### **2 -** How to define a Rule ?
- Each Rule consist of:
  - **Rule Condition (optional):** determines when a rule takes effect/assertions applied (only one per rule);
  - **Assertions:** describes whats values are allowed for a particular parameter. Can contain one or more asserts.
- If you don't define a Rule COndition, the rule's assertions will take effect with every create/update operation.

```yaml
  Rules:
    Rule01:
      Assertions:
        - Assert:
            rule-specific intrinsic function: Value01
          AssertDescription: Information about this assert
    Rule02:
      RuleCondition:
        rule-specific intrinsic function: Value02
      Assertions:
        - Assert:
            rule-specific intrinsic function: Value03
          AssertDescription: Information about this assert
        - Assert:
            rule-specific intrinsic function: Value03
          AssertDescription: Information about this assert
```
#### **3 -** Rules example
- Enforce users to provide an ACM certificate ARN if they configure an SSL listener on an Application Load Balancer

```yaml
  Rules:
    IsSSLCertificate:
      RuleCondition: !Equals
        - !Ref UseSSL
        - Yes
      Assertions:
        - Assert: Not
          - !Equals
            - !Ref ALBSSLCertificateARN
            - ''
          AssertDescription: 'ACM Certificate value can not be empty if SSL is required'
```
Rules-specific Intrinsic Functions
- Used to define a Rule condition and assertions
- Can only be used in the Rules section
- Functions can be nested, but **the result of a rule condition or assertion must be either true or false**
- Supported Functions:
  - Fn::And
  - Fn::Contains
  - Fn::EachMemberEquals
  - Fn::EachMemberln
  - Fn::Equals
  - Fn::If
  - Fn::Not
  - Fn::Or
  - Fn::RefAll
  - Fn::ValueOf
  - Fn::ValueOfAll

### Metadata
#### **1 -** What's Metadata?
- You can use the optional metadata section to include arbitrary YAML that provide details about the template or resource
- For example:
  ```yaml
    Metadata:
      Instaces:
        Description: "Information about the instances"
      Databases:
        Description: "Information about the database"
  ```
#### **2 -** Special Metadata Keys?
- We have already encoutered the Metadata section when dealing with the Designer
- There're 4 Metadata keys that have special meaning
  - _AWS::CloudFormation::Designer_ -> Describes how the resources are laid out in your template. This is automatically added by CloudFormation Designer;
  - _AWS::CloudFormation::Interface_ -> Define grouping and ordering of input parameters when they're displayed in the AWS Console;
  - _AWS::CloudFormation::Authentication_ -> User to specify authentication credentials for files or sources that you specify in AWS::CloudFormation::Init
  - _AWS::CloudFormation::Init_ -> Define configuration tasks for cfn-init. It's the most powerful usage of the Metadata.

### CFN-Init and EC2 User Data
- Many of the CloudFormation templates will be about provisioning computing resources in your AWS Cloud
- These resources can be either
  - EC2 Instances
  - Auto Scaling Groups
  - etc.
- Usually, you want the instances to be self configured so that they can perform the job they're supposed to peform
- You can fully automate your EC2 fleet state with CloudFormation Init

#### User Data in EC2 for CloudFormation
- See the 1-ec2-user-data.yaml archive at section 10.
> The importante thing to pass is the entire script through the function Fn::Base64

#### Enter CloudFormation Helper Scripts
- We have 4 Python scripts, that come directly on AmazonLinux 2 AMI, or can be installed using yum on non-Amazon Linux AMIs
  - cfn-init: Used to retrieve and interpret the resources metadata, installing packages, creating files and starting services
  - sfn-signal: A simple wrapper to signal with a CreationPolicy or WaitCondition, enabling you to synchronize other resources in the stack with the application being ready
  - cfn-get-metadata: A wrapper script making it easy to retrieve either all metadata defined for a resource or path to a specific key or sbtree of the resource metadata
  - cfn-hup: A daemon to check for updates to the metadata and execute custom hooks when the changes are detected
- Usual flow: cfn-init, then cfn-signal, then optionally cfn-hup.

#### AWS::CloudFormation::Init

```yaml
  Resources:
    MyInstances:
      Type: AWS::EC2::Instance
      Metadata: 
        AWS::CloudFormation::Init:
          config:
            packages:
              :
            groups:
              :
            users:
              :
            sources:
              :
            files:
              :
            commands:
              :
            services:
              :
      Properties:
        :
```

- A config contains the following and is executed in thar order
  - <font color= "blue" >Packages:</font> used to download and install pre-packeaged apps and components on Linux/Windows (ex.: MySQL, PHP, etc)
  - <font color= "blue" >Groups:</font> define user groups
  - <font color= "blue" >Users:</font> define users, and which group they belong to
  - <font color= "blue" >Sources:</font> download files and archives and place them on the EC2 instance
  - <font color= "blue" >Files:</font> create files on the EC2 instance, usind inline or can be pulled from a URL
  - <font color= "blue" >Commands:</font> run a series of commands
  - <font color= "blue" >Service:</font> launch a list of sysvinit

#### _-> Packages_
- You can install packages from the following repositories: apt, msi, python, rpm, rubygems and yum.
- Packages are processed in the following order: rpm, yum/apt, and then rubygems and python.
- You can specify a version, or no version (empty array - means latest)
```yaml
  rpm:
    epel: "http://download.fedoraproject.org/pub/epel1/5/i386/epel-release/5/4.noarch.rpm"
  yum:
    httpd: []
    php:
    wordpress: []
  rubygems:
    chef:
      - "0.10.2"
```
- Take a look 2-cfn-init.yaml in section 10.

#### _-> Groups and Users_
- If you want to have multiple users and groups (with an opitional gid) in your EC2 instance, you can do the following
```yaml
  groups:
    groupeOne: {}
    groupTwo:
      gid: "45"
  users:
    myUser:
      groups:
        - "groupOne"
        - "groupTwo"
      uid: "50"
      homeDir: "/tmp"
```
- Take a look to 2-cfn-init.yaml at section 10, line 46.

#### _-> Sources_
- We can download whole compressed archives from the web and unpack them on the EC2 instance directly
- It's very handy if you have a set of standardized scripts for your EC2 instances that you store in S3 for example
- Or if you want to download a whole GitHub project directly on your EC2 instace
- Supported formats: tar, tar+gzip, tar+bz2, zip.
```yaml
  sources:
    "/home/ec2-user/aws-cli": "https://github.com/aws/aws-cli/tarball/master"
```
#### _-> Files_
- Files are very powerful as you have full control over any content you wnat
- They can come from a specific URL or can be written inline
- Additional attributes:
  - source: URL to load file from
  - authentication: name of authentication method to use (used with AWS::CloudFormation::Authentication) - ex.: for files that are protected by user/pass
```yaml
  files:
    /tmp/setup.mysql:
      content: !Sub |
        CREATE DATABASE ${DBName};
        CREATE USER '${DBUsername'@'localhost' IDENTIFIED BY '${DBPassword}'
        GRANT ALL ON ${DBName}.*TO'$DBUsername}'@'localhost';
        FLUSH PRIVILEGES;
      mode: "000644"
      owner: "root"
      group: "root"
```
#### Function Fn::Sub
- Fn::Sub, or !Sub as a shorthand, is used to substitute variables from a text. It's a very handy function that will allow you to fully customize your templates
- For example, you can combine Fn:Sub with References or AWS Pseudo variables !
- String must contain ${VariableName} and will substitute them
```yaml
  !Sub
    - String
    - { Var1Name: Var1Value, Var2Name: Var2Value}
  !Sub String
```
#### Commands
- You can run commands one at a time in the alphabetical order
- You can set a directory from which that command is run, environment variables
- You can provide a test to control whether the command is executed or not (for example: if a file doesn't exist, run the download command)
- Example: call the echo command only if the doesn't exist
```yaml
  commands:
    test:
      command: "echo \"$MAGIC\" > test.txt"
    env:
      MAGIC: "I come from the environment!"
    cwd: "~"
    test: "test ! -e ~/test.txt"
    ignoreErrors: "false"
```

#### Services
- Launch a bunch of services at EC2 instance launch
- Ensure services are started when files changed, or packahes are update by cfn-init. How awesome is that?
```yaml
  services:
    sysvinit:
      nginx:
          enabled: 'true'
          ensureRunning: 'true'
          files:
            - "/etc/nginx/nginx.conf"
          sources:
            - "/var/www/html"
      php-fastcgi:
        enabled: 'true'
        ensureRunning: 'true'
        packages:
          yum:
            - "php"
            - "spawn-fcgi"      
      postfix:
        enabled: 'false'
        ensureRunning: 'false'
```
#### AWS::CloudFormation::Authentication
- Used yo specify authentication credentials for __files__ or __sources__ in AWS::CloudFormation::Init
- Two types:
  - basic: used when the source is a URL
  - S3: used when the source is an S3 bucket
> Prefer using Roles instead of access keys for EC2 instaces !

#### cfn-init
- With the cfn-init script, it helps make complex EC2 configuration readable;
- The EC2 instance will query the CloudFormation service to get init data
- Logs go to /var/lgo/cfn-init.log

#### cfn-signal & wait conditions
- We still don't know how tell CLoudFormation that the EC2 instance got properly configured after a cfn-init
- For this, we can use the cfn-signal script!
  - We run cfn-signal right after cfn-init
  - Tell CloudFormation service that the resource creation success/fail to keep on going or fail
- We need to define WaitCondition:
  - Clovk the template until it receives a signal from cfn-signal
  - We attach a CreationPolicy (also works on EC2, ASG)
  - We can define a Count > | (in case you need more than | signal)

```yaml
  CreationPolicy:
    ResourceSignal:
      Timeout: PT5M
      Count: 1
```
#### cfn-hup
- Can be used to tell your EC2 instance to look for Metadata changes every 15 minutes and aplly the Metadata configuration again
- It's very powerful but you really need to try it out to understand how it works
- It relies on a cfn-hup confiugration, see /etc/cfn/cfn-hup.conf and /etc/cfn/hooks.d/cfn-auto-reloader.conf

#### Creation Policy
- Prevent resource's status from reaching
  CREATE_COMPLETE until CloudFormation receives either
  - A specific number of success signal
  - Timeout period exceeded
- Use cfn-signal helper script to signal a resource
- Supported resources:
  - AWS::EC2::Instance, AWS::CloudFormation::WaitCondition
  - AWS::AutoScaling::AutoScalingGroup, AWS::AppStream::Fleet
- Example:waiting until your applications installed and configured on yout EC2 instance.

#### Wait Condition Didn't Receive the Required Number of Signals from an Amazon EC2 Instance
- Ensure that tha AMI you're using the AWS CloudFormation helper scripts installed. If the AMI doesn't include the helper scripts, you can also download them to your instance.
- Verify that the cfn-init & cfn-signal command was successfully run on the instance. You can view logs, such as /var/log/cloud-init.log or /var/log/cfn-init.log, to help you debug the instance launch
- You can retrieve the logs by logging in to your instance, but you must disable rollback on failure or else AWS CloudFormation deletes the instance after your stack fails to create
- Verify that the instance has a connection to the Internet. If the instance is in a VPC the instance should be able to connect to the Internet though a NAT device ig it's in a private subnet or though an Internet gateway if it's in a public subnet.
- For example, run: 
  ```shell
    curl -l https://aws.amazon.com 
  ```

#### User Data vs CloudFormation::Init vs Helper Scripts
**EC2 User data** is an imperative way to provision/bootstrap the EC2 instance using Shell syntax

**AWS::CloudFormation::Init** is a declarative way to provision/bootstrap the EC2 instance using YAML or JSON syntax

**AWS::CloudFormation::Init** is useless if it’s NOT triggered by a script within the EC2 User Data

Triggering AWS::CloudFormation::Init inside EC2 User Data is done by using **cfn-init** or **cfn-hup**

### CloudFormation Drift
- CloudFormation Allows you to create infrastructure
- But it doesn't protect you against manual configuration changes
- How do we know if our resources have drifted?
- We can use CloudFormation Drift!
- Detect drift on an entire stack or on individual resources within a stack
- We can resolve stack/resource drift by using Resource Import (discussed later)
- Not all resources are supported yet, so please, see the [documentation](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html).

### Nested Stacks
 - Nested stacks are stacks as part of other stacks
 - They allow you to isolate repeated patterns / common components in separete stacks and call them from other stacks
 - Example:
   - Load Balancer configuration that is re-used
   - Security Group that is re-used
 - Nested stacks are considered best practice
 - To update a nested stack, always update the parent (root stack)
 - Nested satcks can have nested stacks themselves

#### Nedted Stacks Update
- To update a nested stack...
- Ensure the opdated nested stacks are uploaded onto s4 first
- Then re-uploade your **root** stack

#### Nested Stacks Delete
- Never delete or apply changes to the nested stack
- Always do changes from the top-level stack

#### Cross Stacks vs Nested Stacks
- **Cross Stacks**
  - Helpful when stacks have different lifecycles
  - Use Outputs Export and Fn::ImportValue
  - When you need to pass export values to many stacks (VPC id, etc)
![Division schema for Cross Stacks](/images/nested.png "Cross Stacks")

- **Nested Stacks**
  - Helpful when components must be re-used 
  - Ex: re-use how to properly configure an Application Load Balancer
  - The nested stack only is important to the higher-level (it's not shared)
![Division schema for Nested Stacks](/images/nested-2.png "Nested Stacks")

#### Exported Stack Output Values vs. Using Nested Stacks
- If you have a central resource that is shared between many different other stacks, use Exported Stack Output Values
- If you need other stacks to be updated right away if a central resource is updated, use Exported Stack Output Values
- If the resources can be dedicated to one stack only and must be re-usable pieces of code, use Nested Stacks
- Note that you will need to update each Root stack manually in case of Nested stack updated

### StackSets
- Create, update, or delete stacks across multiple accounts and regions with a single operation/template.
- Administrator account to create StackSets
- Target accounts to create, update, delete stacks instances from StackSets
- When you update a stack set, _all_ associated stack instances are updated throughout all accounts and regions
- Regional service
- Can be applied into all accounts of an AWS organization
![Stacksets](/images/stacksets.png "Stacksets")

#### Stacksets Operations
- <font color="blue"> Create Stackset</font>
  - Provide template + target accounts/regions
- <font color="blue">Update StackSet</font>
  - Upsate always affect all stack (you can't selectively update some stacks in the StackSet but not others)
- <font color="blue">Delete Stacks</font>
  - Delete Stack and its resources from target accounts/regions
  - Delete stack from your StackSet (the stack will continue to run independently)
  - Delete all stacks from your StackSet (prepare for StackSet deletion)
- <font color="blue">Delete StackSet</font>
  - Must delete all stask instance within StackSet to delete it

#### StackSet Deployment Options
- <font color="blue">Deployment Order</font>
  - Order of regions where stacks are deployed
  - Operations performed one region at a time
- <font color="blue">Maximum Concurrent Accounts</font>
  - Max. number/percentage of target accounts per region to whicj you can deploy stacks at one time
- <font color="blue">Failure Tolerance</font>
  - Max. number/percentage (target accounts per region) of stack operation failures that can occur before CloudFormation stops operation in all regions
- <font color="blue">Region Concurrency</font>
  - Whether StackSet deployed into regions **Sequential (default**) or **Parallel**
- <font color="blue">Retain Stacks</font>
-   Used when deleting StackSet to keep stack and their resources running when removed from StackSet.

#### Permission Models for StackSet
- **Set-managed Permission**s
  - Create the IAM roles (with established trusted relationship) in both administrator and target accounts
  - Deploy to any target account in whicj you have permissions to create IAM role
- **Service-managed Permissions**
  - Deploy to accounts managed by AWS Organizations
  - Stackset create the IAM roles on your behalf (enable trusted access with AWS Organizations)
  - Must enable all features in AWS Organization
  - Ability to deploy to accounts added to your organization in the future (Automatic Deployments)

### StackSet Drift Detection
- Performs drift detection on the stack associated with each stack instance in the StackSet
- If the current state of a resource in a stack varies from the expected state:
  - The stack considered drifted
  - And the stack instance that the stack associated with considered drifted
  - And the StackSet is condidered drifted
- Drift detection indetifies unmanadeg changes (outside CloudFormation)
- Changes made through CloudFormation to a stack directly (not at the StackSet level), aren't considered drifted
- You can stop drift detection on a StackSet

### Intrinsic Functions
#### Fn::Ref 
- The ```Fn::Ref``` function can be leveraged to reference
  - Prameters => returbs the value of the parameter;
  - Resources => returns the physical ID of the underlying resource (ex.: EC2 ID)
- The shorthand for this in YAML is ```!Ref```

```yaml
  DbSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
```
#### Fn::GetAtt
- Attributes are attached to any resources you create
- To know the attributes of your resources, the best place to look at is the documentation
- For example: the AZ of an EC2 machine!

```yaml
  Resources:
    EC2Instance:
      Type: AWS::EC2::Instance
      Properties:
        ImageId: ami-0742b4e673072066f
        InstanceType: t2.micro
```

```yaml
  NewVolume:
    Type: AWS::EC2::Volume
    Condition: CreateProdResources
    Properties:
      Size: 100
      AvailabilityZone: !GetAtt Ec2Instance.AvailabilityZone
```
#### Fn::FindMap
- We use ```Fn:FindInMap``` to return a named value from a specific key
- !FindMap [MapName, TopLevelKey, SecondLevelKey]

#### Fn::ImportValue
- Import values that are exported in other stacks
- For this, we user the ```Fn::ImportValue``` function
```yaml
  Resources:
    MySecureInstance:
      Type: AWS::EC2::Instance
      Properties:
        AvailabilityZone: us-east-1a
        ImageId: ami-0742b4e673072066f
        InstanceType: ts.micro
        SecurityGroups:
          - !ImportValue SSHSecurityGroup
```
#### Fn::Join
- Join values with a delimiter
  ```yaml
    !Join [ delimiter, [comma-delimited list of value] ]
  ```
- This creates " a : b : c "
```yaml
  !Join [ ":", [ a, b, c] ]
```
#### Fn::Base64
- Convert String to it's Base64 representation
```yaml
  !Base64 valueToEncode
```
- Example: pass encoded data to EC2 Instance's UserData porperty

```yaml
  Resources:
    WebServer:
      Type: AWS::EC2::Instance
      Properties:
      ...
      UserData:
      Fn::Base64: |
        #!/bin/bash
        yum update -y
        yum install -h httpd
```
#### Fn::Cidr
- Returns an array of CIDR address blocks
```yaml
  !Cidr [ipBlock, count, cidrBits]
```
- Parameters:
  - ipBlock: CIDR address block to be split
  - count: number of CIDRs to generate (1-256)
  - cidrBits: number of subnet bits of the CIDR, inverse of subnet mask (32-subnet_mask)
```yaml
  !Cidr ['192.168.0.0/24', 6, 5]
```
- Example: generate 6 CIDRs with a subnet mask "/27" (32-5=27) from a CIDR with a subnet mask "/24"

CIDRs            |
-----------------|
192.168.0.0/27   |
192.168.0.32/27  |
192.168.0.64/27  |
192.168.0.128/27 |
192.168.0.160/27 |
192.168.0.192/27 |

#### Fn::GetAZs
- Returns an array of Availability Zones in a region in alphabetical order
```yaml
  !GetAZs region
```
- This example returns ["us-east-2","us-east-2b","us-east-2c"]
```yaml
  # Assume stack created in us-east-2
  !GetAZs: ""
  !GetAZs: !Ref "AWS::Region"

  # Directly specify region
  !GetAZs: us-east-2
```
#### Fn::Select
- Returns a single object from an array of objects by index
```yaml
  !Select [ index, listOfObjects ]
```
- Examples:
```yaml
  # This returns "grapes"
  !Select [ "1", [ "apples", "grapes", "oranges", "limons" ] ]
```

```yaml
  Resources: 
    MyEC2Instance:
      Type: AWS::EC2::Instance
      Properties:
      ...
      # Returns "us-east-1a" (assume stack in us-east-1)
      - 0
      - Fn::GetAZs: !Ref "AWS::Region"
```

#### Fn:Split
- Split a String into a set of String values
```yaml
  !Split [delimiter, sourceString]
```
- This returns the following: ["a", "b", "c"]

```yaml
  !Split [ "|", "a|b|c" ]
```

#### Fn::Transform
- Specifies a CloudFormation Macro(s) to perform custom preocessing on the template
```yaml
  Transform:
    Name: macroName
    Parameters:
      Key: value
```

## CloudFormatino Deployment Options
### ChangeSets
- When you update a stack, you need to know what changes will happen before it applying them for greater confidence
- ChangeSets won't say if the update will be successful
- For Nested Stacks, you see the changes across all stacks

### Stack Creation Failures
- If a CLoudFormation stack creation fails, you will get the status ROLLBACK_COMPLETE
- This means:
  - 1. CloudFormation tried to create some resources
  - 2. One resource creation failed
  - 3. CloudFormation rolled back the resources(ROLLBACK, NO_NOTHING)
  - 4. The stack is in failed created ROLLBACK_COMPLETE state
- To resolve the error, there's only one way:
  **Delete the failed stack and create a new stack**

- You can't update, validate or change-set on a create failed stack

### Rollback Triggers
- Enables CloudFormation to rollback stack create/update operation if that operation triggers CloudWatch Alarm
- CloudFormation monitors the specific CloudWatch alarms during:
  - Stack create/update
  - The monitoring period (after all resourceshave been deployed) 0 to 180 minutes (default: 0 minutes)
- If any of the alarms goes to the ALARM stat, CloudFormation rolls back the entire stack operation If you set a monitoring time but don't specify any rollback triggers, CloudFormation still waits the specified period before cleaning up old resources for update operations
- If you set a monitoring time of 0 minutes, CloudFormation still monitor the rollback triggers during stack create/update operation
- Up to 5 CloudWatch alarms

### Continue Rolling Back an Update
- A stack goes into the UPDATE_ROLLBACK_FAILED state when CloudFormation can't roll back all changes during an update
- A resource can't return to its original state, causing the rollback to fail
- Example: roll back to an old database instance that was deleted outside CloudFormation
- Solutions:
  - Fix the errors manually outside of CloudFormation and then continue update rollback the stack
  - Skip the resources that can't rollback successfully (CloudFormation will mark the failed resources as UPDATE_COMPLETE)
- You can't update a stack in this state
- For nested stacks, rolling back the parent stack will attempt to roll back all the child stacks as well

### Stack Policy
- A JSON document that defines the update actions allowed on stack resources
- Prevent stack resources from being unintentionally updated/deleted **during a stack update**
- By default, all update actions are allowed on all resources
- When enabled, all resources protected by default
- Actions allowed (Update: Modify, Update:Replace, Update:Delete, Update:*)
- Principal supportsonly the wildcard
- To update protect only the wildcard(*)
  - Create a temporary policy that overrides the stack policy
  - The override policy doesn't permanently change the stack policy
- Once created, can't be deleted (edit to allow all update actions on all resources)

### Stacks Termination Protection 
- To prevent accidental deletes of CloudFormation stacks, use TerminationProtection
- Applied to any nested stacks
- Tighten your IAM policies (Ex.: explicit deny on some user groups)
```json
{
  "Version":"2012-10-17",
  "Statement":[{
      "Effect":"Deny",
      "Action":[
          "cloudformation:UpdateTerminationProtection"
      ],
      "Resource": "*"
  }]
}
```
### CloudFormation Service Role / Template Role
#### Service Role
- IAM role that allows CloudFormation to create/update/delete stack resources on your behalf
- By default, CloudFormation uses a temporary session that it generates from your user credentials
- Use cases:
  - You want to achieve the least privilege principle
  - But you don't want to give the user all the required permissions to create the stack resources 
- Give ability to users to create/update/delete the stack resources even if they don't have permissions to work with the resources in the stack

### Quick-create Links For Stacks
- Custom URLs that used to launch CLoudFormation Stacks quickly from AWS Console
- Reduce the number of wizard pages and the amount of user input that's required
- For example: create multiple URLs that specify diferrent values for the same template
- CloudFormation ignores parametes:
  - That don't exist in the template
  - That defined with **NoEcho** property set to true

## Continuous Delivery
### Continuous Delivery with CodePipeline
- Use CodePipeline to build a continuous delivery workflow (building a pipeline for CloudFormation stacks)
- Rapidly and reably make changes to your AWS infrastructure
- Automatically build and test changes to your CloudFormation templates before promoting them to production stacks
- For example:
  - Create a workflow that automatically builds a test stack when you submit a CloudFormation template to a code repository
  - After CloudFormation builds the test stack, you can test it and then decide whether to push chandes to production stack
