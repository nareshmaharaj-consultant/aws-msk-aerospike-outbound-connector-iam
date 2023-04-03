
<p align="center">
<img width="1000" src="img.png">
</p>

In this article we create an AWS MKS Kafka resource using AWS IAM Roles 
and Policies to authenticate user access. We then create an Aeropsike 
cluster and send some sample messages to AWS Kafka via the Aerospike 
Kafka Source connector. The article is centred around using IAM to 
authenticase users and will guide the reader step by step on how to 
achieve this.
<div align="center">. . . .</div>

## AWS MKS Kafka

In this section we will setup a 3 node Kafka cluster. 
- From MSK go ahead and create a new cluster, selecting Quick Create
- Select Provisioned cluster type and select instance type kafka.t3.small
- Select the EBS storage as 10GB
- Make a note of the VPV, Subnets and the Security group ID

Next we create the IAM Policy and Roles

- Create a new IAM Policy, open the JSON tab and paste in the following replacing
  *region:Account-ID*. Save the policy with the name *msk-tutorial-policy*
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:Connect",
                "kafka-cluster:AlterCluster",
                "kafka-cluster:DescribeCluster"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:cluster/MSKTutorialCluster/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:*Topic*",
                "kafka-cluster:WriteData",
                "kafka-cluster:ReadData"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:topic/MSKTutorialCluster/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:AlterGroup",
                "kafka-cluster:DescribeGroup"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:group/MSKTutorialCluster/*"
            ]
        }
    ]
}
```
- Create the IAM Role. Under Common Use Cases, select EC2 and then next.
- Under permissions, select the policy named *msk-tutorial-policy* and then next.
- Give the role a name like *msk-tutorial-role* and click the Create Role button.
