# Lab 1 - Cost management and EC2 scaling - Detailed Steps

## 1.0  Ensure you have run the pre-requisites

1.  Make sure you have an EC2 keypair created.

    1.  Sign-in to the AWS EC2 console at https://console.aws.amazon.com/ec2/ 
    2.	Click Key Pairs and then click Create Key Pair. 
    3.	Give the key pair a name and click Create. The console will generate a new key pair and download the private key.
    4.  Keep the key pair somewhere safe

2. Deploy the initial CloudFormation template.  This creates IAM roles, an S3 bucket, and other resources that you will use in later labs. The template is called cfn-templates/Lab0-baseline-setup.yml If you are sharing an AWS account with someone else doing the workshop, only one of you needs to create this stack. In Stack name, enter catsndogssetup. Later labs will reference this stack by name, so if you choose a different stack name you will need to change the LabSetupStackName parameter in later labs.

    1. In the **Management Tools** section click **CloudFormation**.

    2. Click **Create Stack**.

    3. Select **Upload a template to Amazon S3**, then click **Choose** File and choose the file named **Lab0-baseline-setup.yml**

    4. In Stack name, enter **catsndogssetup**

    5. Accept the defaults and ensure that you tick **I acknowledge that AWS CloudFormation might create IAM resources with custom names**.


## 1.1	Create a new ECS cluster using Spot Fleet

1.	Sign-in to the AWS management console and open the Amazon ECS console at https://console.aws.amazon.com/ecs/.

2.	In the AWS Console, ensure you have the correct region selected.

3.	In the ECS console click **Clusters**, then click **Create Cluster**. Select **EC2 Linux + Networking** then hit **Next Step**

4.	In Cluster name, type **catsndogsECScluster** as the cluster name. This name is used in later labs. If you name the cluster something else you will have to remember this when running later commands.

5.	In **Provisioning Model** select **Spot**. 

6.	Leave **Spot Instance allocation strategy** as **Diversified**.

7.	In **EC2 instance types** add several different instance types and sizes. We recommend you pick smaller instances sizes, such as:

    1. m5.xlarge
    
    2. c5.large
    
    3. r5.large
    
    4. i3.large
    
**Note:** You can also pick older generation families such as m3.large..

8.	In **Maximum big price (per instance/hour)** you can click the **Spot prices** link to view the current spot prices for the instance types and sizes you have selected. More information on how EC2 Spot instance pricing works is available on the Amazon EC2 Spot Instances Pricing page: https://aws.amazon.com/ec2/spot/pricing/

9.	Enter a maximum bid price. For the purposes of the workshop, 0.25 should offer an excellent chance of your Spot bid being fulfilled. It does not matter if your spot bid is not fulfilled. In a later step you will add an on-demand instance to the cluster.

10.	In **Number of instances** enter **3**.

10. use the default EC2 AMI.

10. EBS Storage (GiB), leave at default (**22**)

11.	In **Key pair** select an existing EC2 Key pair for which you have the private key.

12.	In **VPC** select **ECSVPC**

13.	In **Subnets** select all subnets containing the word **Private**.

14.	In **Security group**, select the Security Group containing the term **InstanceSecurityGroup**.

14. In **Security Group inbound rules**, leave it as default

15.	In **Container Instance IAM role** select the IAM role containing the term **catsndogssetup-EC2Role**.

16.	In **IAM role for a Spot Fleet request** select the role with a name containing **catsndogssetup-SpotFleetTaggingRole**.

17.	Click **Create**.

18.	You will see the cluster creation steps appear. The final step is the creation of a CloudFormation stack. Note the name of this stack.  This step should take a few minutes.

19.	Open the AWS console in a new browser tab and under **Management Tools**, click **CloudFormation**.

20.	Select the checkbox for the CloudFormation stack, and click the **Template** tab.

21.	The ECSSpotFleet resource has a Property named **LaunchSpecifications**, which contains **UserData**. This is about half way down the template. This UserData creates a termination watcher script described below. You will not be able to see the contents directly from the CloudFormation console.

**Note:** This script creates a Spot instance termination notice watcher script on each EC2 instance. That watcher script runs on each instance every two minutes. It polls the EC2 instance metadata service for a Spot termination notice. If the instance is scheduled for termination (because you have been outbid) the script sends a command to the ECS service to put itself into a DRAINING state. This prevents new tasks being scheduled on the instance, and if capacity is available in the cluster, ECS will start replacement tasks on other instances within the cluster.

More information about this script can be found on the AWS Compute blog: https://aws.amazon.com/blogs/compute/powering-your-amazon-ecs-cluster-with-amazon-ec2-spot-instances/

More information about the ECS DRAINING state can be found in the ECS documentation: http://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-draining.html

## 1.2	Set up Auto Scaling for the Spot fleet

In this task we will set up Auto Scaling for the Spot fleet, to provide cost-effective elasticity for the ECS Container Instances. Auto Scaling will use the ECS cluster MemoryReservation CloudWatch metric to scale the number of EC2 instances in the Spot fleet.

1.	In the AWS Console **Management Tools** section click **CloudWatch**.

2.	Click **Alarms**, then click **Create Alarm** to create an alarm for scaling out.

3.	Click **ClusterName** under ECS Metrics.

4.	Select the **MemoryReservation** metric for the cluster you created earlier, then click **Next**. It might take a minute or two for this new metric to appear in the CloudWatch console. If the metric is not yet listed, refresh the page and try again.

5.	Leave the **Metric name**, as **Memory Reservation**

6.	For the **statistic** select **Maximum**.

7.	For the **Period** select **1 minute**.

8.	Fill in the following under **Conditions**:

    1. Thresholde type: **static**
    
    2. Whenever Memory Reservation Is: **greater**

    3. than... **20**

    4. Additional Configuration Datapoints to Alarm : **2** out of 2 datapoints

9.  Click **next** 

9.	In **Configure Actions**, remove the pre-created Notification action.

10.	Click **Next** and enter an **alarm name**, for example **ScaleUpSpotFleet**.

10. Review the details and then **Create Alarm**

We will now create a second alarm for scaling down

11.	Click **Create Alarm** to create the alarm for scaling down.

12.	Click **ClusterName** under ECS Metrics.

13.	Select the **MemoryReservation** metric for the cluster you created earlier, then click **Next**.

14.	For the **statistic** select **Maximum**.

15.	For the **Period** select **1 minute**.

16.	Fill in the following under **Conditions**:

    1. Thresholde type: **static**
    
    2. Whenever Memory Reservation Is: **Lower**

    3. than... **20**

    4. Additional Configuration Datapoints to Alarm : **2** out of 2 datapoints

17.  Click **next**

18.	In **Configure Actions**, remove the pre-created Notification action.

19.	Click **Next** and enter an **alarm name**, such as **ScaleDownSpotFleet**

20. Review the details and then **Create Alarm**

When we created the cluster, it automagically created a Spot Request.  We will now update that Spot Request to use our newly created alarms.  Start by opening the **EC2 Console**.

21.	Click **Spot Requests**.

22.	Select the checkbox by the Spot request.

23.	Click the **Auto Scaling** tab in the lower pane, then click **Configure**.

24.	In **Scale capacity** between, set **3 and 10** instances.

25.	Under **Scaling policies**, click the **Scale Spot Fleet using step or simple scaling policies** option.  This will be under the default scaling policy.

26.	In Scaling policies first update the ScaleUp policy:

    1. In **Policy Trigger** select the **ScaleUpSpotFleet** alarm you created earlier.
    
    2. Click **Define steps**.
    
    3. Click **Add step**.
    
    4. In **Modify Capacity**:
        
        1. Add 2 instances when 20 <= MemoryReservation <= 50
        
        2. Add 3 instances when 50 <= MemoryReservation <= infinity

        3. Change cooldown Period to **30** seconds

27.	Then update the ScaleDown policy:
    
    1. In **Policy Trigger** select the **ScaleDownSpotFleet** alarm you created earlier.
    
    2. Click **Define steps**.
    
    3. Click **Add step**.
    
    4. In **Modify Capacity**:
    
        1. Remove 1 instances when 20 >= MemoryReservation > 10
        
        2. Remove 2 instances when 10 >= MemoryReservation > -infinity

        3. Change cooldown Period to **30** seconds

28.	Click **Save**

More details on Auto Scaling for Spot fleet is available in the Spot Instances documentation: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet-automatic-scaling.html

## 1.3	Add an On-Demand Auto Scaling group to the cluster

In this task you will create an Auto Scaling group composed of two EC2 instances. If the Spot price goes above the maximum bid price, some or all of the Spot instances could be terminated. By using on-demand instances as well as Spot instances, you ensure the cluster will have capacity even if the Spot instances are terminated.

**Note:** For ECS clusters that will operate for a year or more, EC2 Reserved Instances provide both a capacity reservation and lower price per hour. We will not use Reserved Instances in this workshop but you should consider them for long-lived clusters.

If you have used Auto Scaling groups with ECS before, you can launch a CloudFormation stack that creates the resources below automatically. The CloudFormation template is called **Lab1-add-ondemand-asg-to-cluster.yml**. If you have not used Auto Scaling groups with ECS, you can follow the steps below to learn how to do this.

1.	In the AWS Console **Compute** section click **EC2**, then click **Instances**.

2.	Right click on an instance and click **Launch more like this**.

3.	You will see the review page, find the **Instance Type** section and click **Edit**

4.	Select the **m5.large** instance type.

5.	Click **Configure Instance**. If you receive a pop-up dialog, select “Yes, I want to continue with this instance type (m5.large)” and click **Next**.

6.	Beside **Number of instances** click **Launch into Auto Scaling Group**.

7.	In the pop-up dialog click **Create Launch Configuration**. This launches the Auto Scaling Launch Configuration wizard and preserves the AMI and instance type and size.

8.	In the **Configure Details** step enter **On-Demand-ECS** as the name.

9.	In IAM role select the IAM role containing the term **EC2InstanceProfile**.

9.  Leave the other options as default values.

10.	Expand Advanced Details. Copy the following text and paste it into the User data dialog box. This controls which ECS cluster the instance will join:

```
#!/bin/bash
echo ECS_CLUSTER=catsndogsECScluster >> /etc/ecs/ecs.config
```

11.	Click **Next: Add storage** and accept the default values.

12.	Click **Next: Configure Security Group**.

13.	Click **Select an existing security group** and choose the Security Group containing the term **InstanceSecurityGroup**.

14.	Click **Review** then click **Create launch configuration**.

15.	In the pop-up dialog, select **Choose an existing key pair**, then select an EC2 key pair that you have the private key for. Click the checkbox and then click **Create launch configuration**.

16.	The completes the Launch Configuration wizard and starts the Auto Scaling Group wizard. In Group name, enter **ECS-On-Demand-Group**.

17.	In **Network**, select the **ECSVPC**.

18.	In **Subnet** select all subnets containing the word **Private**. Click **Next: Configure scaling policies**.

19. Click **Next:Configure Scaling policies**

19. Select **Keep this group at its initial size**

19. Click **Next: configure notifications**

19. Click **Next: configure tags**

19.	Click **Review**.

20.	Click **Create Auto Scaling group**, then click **Close**.

21.	Return to the AWS Console and in the **Compute** section, click **EC2 Container Service**.

22.	Click the ECS cluster **catsndogsECScluster**.

23.	Click the **ECS Instances** tab and wait until the On-Demand instance appears in the list. You can continue the next task once the instance appears. If the instance does not appear within a few minutes, check the configuration of the Launch Configuration, specifically the **User data** script, the **IAM role**, and the **VPC and subnet** selections.

You should now have an ECS cluster composed of three instances from the Spot fleet request, and one instance from the on-demand Auto Scaling group.

# What's Next
[ECS Service deployment and task Auto Scaling](../Lab-2-Artifacts/)
