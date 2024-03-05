1. Log in to the AWS console with a user that is authorized to the instance with the cluster in it.
2. In the elastic beanstalk section, delete the beiwe-application
3. In the RDS section, delete the beiwe database
4. In the EC2 section -> Security Groups delete all of the security groups less the 'default VPC security group' this is often an iterative process, delete as many as you can, then try again.
5. In the S3 section delete the the elasticbeanstalk bucket and the data bucket associated with the application. In order to delete the elasticbeanstalk buckets you will need to edit the policy under Permissions -> Bucket Policy and remove the following:

        ,
                {
                    "Sid": "eb-58950a8c-feb6-11e2-89e0-0800277d041b",
                    "Effect": "Deny",
                    "Principal": {
                        "AWS": "*"
                    },
                    "Action": "s3:DeleteBucket",
                    "Resource": "arn:aws:s3:::elasticbeanstalk-us-west-1-322792565403"
                }
