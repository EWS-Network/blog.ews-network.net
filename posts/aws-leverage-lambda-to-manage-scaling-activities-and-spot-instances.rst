.. title: Leverage Lambda to manage Scaling Activities and Spot Instances
.. slug: aws-leverage-lambda-to-manage-scaling-activities-and-spot-instances
.. date: 2016-11-06 10:46:59 UTC
.. tags: AWS, Public Cloud, Spot Instance, Auto-Scaling, Lambda, SNS, CloudWatch
.. category: Cloud
.. link:
.. description:
.. type: text


The use-case
============

A group of friends asked me for help as they were in need of a SysAdmin for their startup. Their web application, mostly API based, integrates a Video Edition tool, which allows users to upload videos, sounds, images etc. which then they can cut, edit and render. The major requirement for an efficient production was to leverage a GPU. Historically, they were using OVH as their provider, but OVH offering for GPU servers starts at 2K Euros / months. Not really in their budget.

So, of course, I told them to go for AWS instead where they could have GPU instances and pay only for when they need it. So, after a few weeks of work, they had the necessary automation in place to have "Workers" running on GPU instances get created when the SQS queue was growing up. With CloudWatch and Scaling Policies, AWS was starting on-demand GPU instances. From 600USD per month for a G2.2xlarge to just a few USD a day, savings were already significant. But as I was working on that with them, I wanted to go even further and use Spot Instances. For the GPU instances, it is a potential 75% saving of the compute-hour. For production as for development, it is a significant saving.

The problem
===========

With the CloudFormation templates all ready, we simply duplicated the AutoScaling group and Launch Config for the GPU and instances. We now had 2 ASG with the same set of metrics and alarms. But, how could we distinguish which ASG should scale up first when the Queue grows in messages and we need a GPU to compute results faster ? I could not find an easy answer with integrated dispatch within AWS AutoScaling nor CloudWatch services..


Possible solutions
==================

Online we could find `SpotInst <https://spotinst.com/>`_, a company that manages the AutoScaling groups and whenever a scaling operations is necessary, is going to manage for you "Spot or On-demand ?" (at least, that's what I understand of the documentation). Of course, SpotInst proposes a lot more services integrated with that, but I personaly found a little bit of an overkill for our use-case.

That is where the integration with CloudWatch and SNS, paired with Lambda as a consumer or our SNS topic, comes in and does it all for us with what I have called the "SpotManager".


The idea
========

As you probably already guessed, the Spot Manager is my Lambda function which will distinguish which of the AutoScaling group should trigger a scale-up activity. Here is an overview of the workflow :

.. image:: /images/spotmanager/SpotManagerWorkflow.PNG

How to?
=======

For this solution, we will need:

1. Our 2 Autoscaling Groups

   A. Identical Launch Configuration apart from the SpotPrice
   B. Scale up policy configured with no alarm
   C. Scale down policy configured with CW Alarm

2. SNS Topic to get messages / notifications from CloudWatch

3. CloudWatch alarms on the Queue

   A. Alarm to raise "Jobs to do" signal
   B. Alarm to raise "No jobs anymore" signal

4. "SpotManager" Lambda Function



In terms of pricing, the EC2 side of things is purely hour-compute maths, so report to the `EC2 Spot pricing <https://aws.amazon.com/ec2/spot/pricing/>`_ and `EC2 On-Demand pricing <https://aws.amazon.com/ec2/pricing/on-demand/>`_.
Good news : SNS delivery to Lambda costs 0$USD as referenced `here <https://aws.amazon.com/sns/pricing/>`_, for CloudWatch we count ~0.4$USD per month. The Lambda pricing depends on how long the function runs. Here, it might take up to 1 second per invoke, so, per months you probably won't go over the free-tier.

Total cost : less than 1$USD per month per pair of ASG (Compute pricing excluded).

1 - The AutoScalingGroups
=========================

A. The Launch configurations
----------------------------


To simplify all the steps, I have published `here <https://github.com/EWS-Network/ews-cf-templates/blob/master/autoscaling/spotmanaged.yml>`_ a cloudformation template that will create 2 autoscaling groups, as explained earlier, with an identical Launch Configuration at the difference that one has the property * SpotPrice * set.

B. Scale up policy
------------------

Here, also in the Cloudformation template provided, we create a scale-up policy : when triggered, this will add 1 instance to the AutoScaling Group by raising the value of the "Desired Capacity". With the on-demand ASG, nothing fancy will happen if you trigger it: the EC2 service will kick off a new instance according to the ASG properties and the Launch Configuration. Now, if you do the same for testing with the ASG configured for Spot Instances, you will notice that first, in the "Spot Instances" section of the Dashboard, the spot request is being evaluated: a Spot request is sent with the Max bid you are willing to pay for that instance type. If the current spot market allows it, this spot request will be granted and an instance will be created in your account.

C. Scale down policy
--------------------

As we need machines for jobs coming in, we are also capable to tell when we don't need any compute resources anymore. Depending on how you do the queue messages length analysis, you should be able to determine pretty easily when there are no more messages your workers have to consume. Therefore, I have linked the Scale Down policy to the SQS Alarm. The good thing about an alarm is that you can have the same alarm go to multiple actions. So here, as we have 2 ASG we want to treat the same way regarding scale-down, we instruct the alarm to trigger both ASG' scale-down policies.

.. note:: The alarm can trigger mulitple actions, but remember that you need to configure a scaling policy on each individual ASG for it to work.


.. note:: For the rest of the blog, I am going to work with 2 ASG. If you haven't already, create those with a minimum at 0, maximum at 1 and desired capacity at 0. No need to pay before we get to the real thing.


2 - The SNS Topic
=================

SNS is probably one of the oldest service in AWS and doesn't stop growing in features. As a key component of the AWS eco-system, it is extremely easy to integrate other services as consumers of the different topics we can have. Here we go in our AWS Console :

.. thumbnail:: /images/spotmanager/2016-11-06_17-20-28.png


Via the cli ?

.. code-block:: bash

   aws sns create-topic --name mediaworkerqueuealarmsstatus


That was easy, right ? Let's continue.


3 - Cloudwatch and Alarms
=========================

For our demo, I have already created a queue called "demoqueue". From here, we have different metrics to work with. Here, I am going to use the * ApproximateNumberOfMessagesVisible * . This number will stack up as long as messages reside in the queue without being consumed.

.. warning:: Remember that the metrics for the SQS service are updated only every 5 minutes. If for any reason you have to get the jobs started faster than CW to notify you, you will have to find a different way to trigger that alarm


3A - Alarm "There are jobs to process!"
---------------------------------------

The new Cloudwatch dashboard released just recently makes it even easier to browse the Metrics and create a new alarm.

1. Identify the metrics

On the CloudWatch dashboard, click on Alarms. There, click on the top button "Create alarm". The different metrics available appear by category. Here, we want to configure the SQS metrics.

.. thumbnail:: /images/spotmanager/2016-11-06_18-43-48.png

2. Configure the threshold

.. thumbnail:: /images/spotmanager/2016-11-06_18-46-55.png

3. Check the alarm summary

.. thumbnail:: /images/spotmanager/2016-11-06_18-47-21.png


3B - Alarm "Chill, no more jobs"
--------------------------------

For that alarm, we are going to follow the same steps as for the previous alarm, but, we are going to use a different metric and configure a different action. Both our ASG have a scale-down policy. So, let's create that alarm.

1. - Identify the metric

2. - Configure the threshold

3. - Set the alarm actions


4 - The SpotManager function
============================

As explained earlier, I create about everything via CloudFromation, which allows me to leverage tags to identify my resources quickly and easily. That said, the function I share with you today is made to work in any region, the only thing you might have to implement to suit your use-case is *how to identify the asg ?*.


The code
--------

As usual, the code for the lambda function can be found `here, on my github account <https://github.com/johnpreston/spotmanager>`_. Be aware that this function is zipped with different files because I separated each different "core" function to be re-usable in different cases.

However, I have tried to get the best rating from pylint (^^) and document each functions params/return, each of those named with, I hope, self-explainatory names.


.. warning:: The code shared here is really specific to working with CloudFormation templates and my use-case. I use SQS where you might simply use EC2 metrics, or any kind of metrics. Adapt the code to figure out the action to trigger.


spotworth.py
~~~~~~~~~~~~

This is the python file that is going to analyze for each different AZ where you have a subnet. For each subnet, it is going to retrieve the average spot price for the past hour of the instance-type you want to have.

.. warning:: You could have 3 subnets in 2 AZs within a 5 AZs region, so you actually can run instances within those 2 AZ only, hence why the script takes a VPC ID as parameter.


getAutoScalinGroup.py
~~~~~~~~~~~~~~~~~~~~~

Here is the CloudFormation parser that will read all information from the stackname you created the two ASG with. Those functions are mostly wrappers around existing boto3 ones to make it easier to get right to the information we are looking for. In our case, we are going to:

1. Assume the stack we are looking our ASG in could be nested. Therefore, we look on the stack name given and we find our ASG with their *Logical Id* expressed in the template (ie: asgGPU)
2. Once the ASG *Physical ID* names are found (ie: mystack-asgGPU-546540984) we can retrieve any sort of informations

   A. Instances in the group ?
   B. Scaling Policies ?
   C. Any, but that's all we are looking for here ;)


Of course, we could have looked for the ScalingPolicy physical Ids right away from the CloudFormation template, but just in case you misconfigured / mislinked the ASG and the ScalingPolicy (the policy is not there ?), this helps us verify that that's not the case and our ScalingPolicy is linked to the right ASG.


spotmanager.py
~~~~~~~~~~~~~~

This is the central script from which all the others are going to be executed. Originally, this function was called right away by an Invoke API call providing most of the variables to the function. In the repository, you will find an file named *spotmanager_sns.py* which is the adaptation of the code to our use-case. The main difference is that, we assume the topic name is a combination of the stackname (AWS::StackName) and other variables. That way we can simply know which Stack runs it and we can find out the rest.

So here is the algorythm.

.. thumbnail:: /images/spotmanager/spotalgo.jpg


.. note:: Any Pull Request to make it a better function is welcome :)


The IAM Role
------------

As for every lambda function, I create an IAM role to control in detail every access of each individual function to the resources. Therefore, here are the different statements I have set in my policy

.. note:: Do not forget the AWS Managed policies *AWSLambdaBasicExecutionRole* and *AWSLambdaExecute* so you will have the logs in CloudWatch logs.


AutoScaling Statement
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: json

    {
    "Sid": "Stmt1476739808000",
    "Effect": "Allow",
    "Action": [
      "autoscaling:DescribeAutoScalingGroups",
      "autoscaling:DescribeAutoScalingInstances",
      "autoscaling:DescribeLaunchConfigurations",
      "autoscaling:DescribePolicies",
      "autoscaling:DescribeTags",
      "autoscaling:ExecutePolicy"
      ],
      "Resource": [
       "*"
       ]
    }


EC2 Statement
~~~~~~~~~~~~~~~~~~~~~~~~

There are a few EC2 calls we need to authorize. Here, all those calls will help me identify the subnets, the Spot pricing and other information necessary to decide what to do next.

.. code-block:: json

   {
    "Sid": "Stmt1476739866000",
    "Effect": "Allow",
    "Action": [
     "ec2:DescribeInstanceStatus",
     "ec2:DescribeInstances",
     "ec2:DescribeSpotInstanceRequests",
     "ec2:DescribeSpotPriceHistory",
     "ec2:DescribeSubnets",
     "ec2:DescribeTags"
    ],
    "Resource": [
      "*"
      ]
   }


CloudFormation statement
~~~~~~~~~~~~~~~~~~~~~~~~

As explained, the scripts I have written call the API of CloudFormation to get information about the stack resources etc. This allows me to identify the ASG I want to scale-up.

.. code-block:: json

   {
   "Sid": "Stmt1476740536000",
   "Effect": "Allow",
   "Action": [
    "cloudformation:DescribeStackResource",
    "cloudformation:DescribeStackResources",
    "cloudformation:DescribeStacks"
   ],
   "Resource": [
    "*"
    ]
   }


The full policy `can be found here <https://goo.gl/uKjZ7S>`_


How could we make it more secure ?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I built this function to be work for all my stacks, hence why the resources are "all" (\*). But if there is a risk of the possibility that the function could go rogue or exploited, we could do something very simple in our CloudFormation stack :

- Create the policies and the role as described earlier specifying the resources as we created them in the CF Template.
- Create the lambda function with the stack (requires to have bundled the function as a zip file)

A bit of extra work for extra security. Just keep in mind that, Lambda costs you a little bit for the code storage. But, probably negligeable compared to the financial risks of leaving the function go rogue ?


Conclusion
==========

So the code, I hope you will see different ways to approach this to find out which ASG to trigger. That is the tricky thing to do. However, this really helped my friends to drop their GPU footprint and for the future if the business is successful, extend the GPU footprint above the 1 GPU machine to multiple with the same cost management control.
