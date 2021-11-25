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

