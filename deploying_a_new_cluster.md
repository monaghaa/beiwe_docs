There are fairly reasonable instructions for [deploying the Beiwe cluster][deploy_cluster].
However, there are some additions and clarifications that are needed to get the cluster
fully functional. I will reference the document sections below and put in the
clarifications needed.

### Prerequisites:

- The Existing AWS Access Key ID and Secret Access Key. If this is a completely fresh install (completeley new AWS account and VPC) follow the '[normal IAM instructions][normal_IAM_instructions]' to generate an access key associated with the proper policy.
- The existing [Sentry](https://sentry.io) key, again if this is a completely new install, follow the instructions to generate a Sentry key.

### Configure the AWS region to require EBS encryption by default
This is just good sense, and costs nothing extra as well as incurring zero
performance penalty, [more details][ebs_encryption_by_default].

1. Log in to to the AWS console
2. Select the region you wish to deploy the cluster into.
3. Under EC2 -> Settings (far right under account attributes) check the option 'Always encrypt new EBS volumes'

### In the '[Configure Your Application][configure_your_application]' section:

#### Step 4: 
Deplyment key is simply an SSH key. You can generate an SSH key in AWS, but for most of us we will 
want to keep it seperate. So an example global_configuration.json may make this clearer:

    {
      "DEPLOYMENT_KEY_NAME": "looneytr",
      "DEPLOYMENT_KEY_FILE_PATH": "/home/looneytr/.ssh/unixops_rsa",
      "VPC_ID": "vpc-e44de19c",
      "AWS_REGION": "us-west-2",
      "SYSTEM_ADMINISTRATOR_EMAIL": "looneytr@colorado.edu"
    }

Yes the SSH/Deployment key should be a shared key for groups that are managing the VPC, you can sort that out 
on your own. 

The VPC ID can be found in the AWS console using the 'VPC' applet. Region is of course users choice. 
OIT seems to mostly deploy to us-west-2. 

#### Step 5: 
I tend to use a virtualenv here, but it is the users choice what you wish to do, for completeness:

    looneytr:cluster_management/ (master) $ virtualenv venv
    looneytr:cluster_management/ (master) $ source venv/bin/activate
    looneytr:cluster_management/ (master) $ which python
    ~/repo/beiwe-backend/cluster_management/general_configuration/venv/bin/python
    looneytr:cluster_management/ (master) $ pip install -r launch_requirements.txt
 
#### Step 8:
Ensure that you have added the SSH key generated in the '[Generate a deployment key][generate_deplyment_key]' section to ssh agent before executing the create-manager step. 'create-manager' connects via ssh to the system and does not expect a password prompt.

### In the '[Set up the AWS Elastic Beanstalk Command-Line Interface (EB CLI)][setup_eb_cli]' section:
Again I use (the same) virtualenv for this, you can do whatever is comfortable. 

Though the beiwe-backend/.elasticbeanstalk/config.yml file already has the application_name set ensure that it is set to 'beiwe-application' per the wiki otherwise the eb deploy will fail as eb will not look in the right S3 location for the application.

Also be aware this config.yml file was accidently checked into the repo (per the header in the file), it should actually be excluded, this is a mistake on the part of the beiwe folks and we have to work around it.

### In the '[Configuring DNS][configuring_dns]' section:
This section can basically be skipped. You will need to request a CNAME from
NEO to be added pointing to the name of the ELB, which you can find in
EC2 -> Load Balancers section in the 'Description' tab. It will look something like:
awseb-e-j-AWSEBLoa-179RZLVUVIBYS-1818176523.us-west-2.elb.amazonaws.com

### In the '[Configuring SSL][configuring_ssl]' section:
At this point the cluster is up and the application (should) be working. However, apache on the host is configured to require the client to use TLS when talking to the load balancer, so in order to see the application via the load balancer for the first time, you will need to configure TLS on the load balancer to work. 

#### Obtain the TLS certificate:
If you are redeploying the cluster, the key pair has already been uploaded into IAM for you and this section can be skipped

Otherwise, you will need to manually generate a key and csr and submit those through the usual channels to obtain a signed certificate. Once the certificate is obtained it needs to be uploaded into IAM, despite what you may read online, it does not appear to be possible to upload these using the AWS console, this will need to be done via the CLI like so:

      aws --region=us-west-2 iam upload-server-certificate --server-certificate-name beiwe --certificate-body file://beiwe.colorado.edu.pem --private-key file://beiwe.colorado.edu.key --certificate-chain file://../Downloads/beiwe_colorado_edu_interm.cer

#### Configure the Load Balancer for TLS:
Be aware that step 2 listed in the wiki, is unnecessary, the application is already configured
for TLS, you just need to get the ELB to use the certificate.

Until [this commit](https://github.com/onnela-lab/beiwe-backend/commit/3b2ae8119c04b0f2bd8ff5783b0d85672f560a51) is merged into master the configuration stands this way TLS (443) -> ELB -> No TLS (80) -> Application. Follow the appropriate instructions depending on the state of the commit. 
 
##### TLS -> ELB -> No TLS -> Application:
This assumes the above referenced commit has NOT been merged

1. In the AWS console navigate to EC2 -> Load Balancers
2. Right click on the load balancer and choose 'Edit Listeners'
3. Set the 'Load Balancer Protocol' to 'HTTPS (Secure HTTP)'
4. Set the 'Load Balancer Port' to '443'
5. Choose Change for the 'Cipher' and set it to a reasonably secure default 'TLS-1-2-2017-01' at the time of this writing, then save
6. Choose 'Change' for the 'SSL Certificate'
7. Select 'Choose a certificate from IAM' radio button and select the 'beiwe' certificate then save.
8. Navigate to Ec2 -> Security Groups
9. Edit the inbound rule for the 'Elastic Beanstalk created security group used when no ELB security groups are specified during ELB creation' and modify the port from 80 to 443.


##### TLS -> ELB -> TLS -> Application:
If the above referenced commit HAS been pulled into master configure things as follows:

Once the certificate is uploaded into IAM you need to set the ELB to listen
on port 443 for HTTPS traffic and use the given certificate, like so:
1. In the AWS console navigate to EC2 -> Load Balancers
2. Right click on the load balancer and choose 'Edit Listeners'
3. Set the 'Load Balancer Protocol' to 'HTTPS (Secure HTTP)'
4. Set the 'Load Balancer Port' to '443'
5. Choose Change for the 'Cipher' and set it to a reasonably secure default 'TLS-1-2-2017-01' at the time of this writing, then save
6. Choose 'Change' for the 'SSL Certificate'
7. Select 'Choose a certificate from IAM' radio button and select the 'beiwe' certificate then save.
8. Navigate to Ec2 -> Security Groups
9. Edit the inbound rule for the 'SecurityGroup for ElasticBeanstalk environment.' and modify the port from 80 to 443
10. Edit the inbound rule for the 'Elastic Beanstalk created security group used when no ELB security groups are specified during ELB creation' and modify the port from 80 to 443.

## Secure the Application
By default a number of relatively poor choices are made in terms of security,
not terrible choices, but poor ones. As well when using the ebcli when you run 'eb ssh' to connect to the environment an allow all to port 22 is injected into the environment before the ssh connection starts, and is subsequently removed after the ssh connection is terminted, as such allowing ssh from everywhere is simply not necessary.

1. Navigate to EC2 -> Security Groups
2. Remove all inbound port 22 exceptions open to anywhere (0.0.0.0/0)

## Wrapping up the cluster deploy
The cluster should now be up and (relatively) secure. The default login credentials are:

    Default Admin Username: default_admin
    Default Admin Password: abc123

Obviously change the password on login.

[configure_your_application]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-Instructions---Scalable-Deployment#configure-your-application
[configuring_dns]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-instructions#configuring-dns
[configuring_ssl]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-instructions#configuring-ssl
[deploy_cluster]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-Instructions---Scalable-Deployment
[ebs_encryption_by_default]: https://aws.amazon.com/blogs/aws/new-opt-in-to-default-encryption-for-new-ebs-volumes/
[generate_deplyment_key]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-instructions#generate-a-deployment-key
[normal_IAM_instructions]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-Instructions---Scalable-Deployment#set-up-an-iam-user-with-sufficient-permissions
[setup_eb_cli]: https://github.com/onnela-lab/beiwe-backend/wiki/Deployment-Instructions---Scalable-Deployment#set-up-the-aws-elastic-beanstalk-command-line-interface-eb-cli
