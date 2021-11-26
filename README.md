## Notes about course
### Links
* [Resources and types of resources](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

* [EC2 Instance Resource Type](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html)

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
    ap-northeast-3:
      HVM64: ami-0b96303a469fa0678
    ap-south-1:
      HVM64: ami-0bcf5425cdc1d8a85
    ap-southeast-1:
      HVM64: ami-03ca998611da0fe12
    ap-southeast-2:
      HVM64: ami-06202e06492f46177
    ca-central-1:
      HVM64: ami-09934b230a2c41883
    eu-central-1:
      HVM64: ami-0db9040eb3ab74509
    eu-north-1:
      HVM64: ami-02baf2b4223a343e8
    eu-south-1:
      HVM64: ami-081e7f992eee19465
    eu-west-1:
      HVM64: ami-0ffea00000f287d30
    eu-west-2:
      HVM64: ami-0fbec3e0504ee1970
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
### AWS::CloudFormation::Authentication
- Used yo specify authentication credentials for __files__ or __sources__ in AWS::CloudFormation::Init
- Two types:
  - basic: used when the source is a URL
  - S3: used when the source is an S3 bucket
> Prefer using Roles instead of access keys for EC2 instaces !

