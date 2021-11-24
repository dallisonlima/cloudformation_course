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

### Notes

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