.. title: AWS - Secure CloudFormation variables using AWS SSM and KMS
.. slug: secure-cloudformation-variables-using-aws-ssm-and-kms
.. date: 2017-10-07 13:22:29 UTC
.. tags: AWS,SSM,KMS,CloudFormation
.. category: AWS
.. link:
.. description:
.. type: text


This article is a version 2.0 of my previous article `Protect your CloudFormation sensible values and secure them with KMS and DynamoDB <https://aws.amazon.com/sns/pricing/>`_

Use-Case
========

In my previous article, I was using DynamoDB as my Parameter Store and KMS to encrypt all the information I do not want to see accessible in plain text. The only issue with that is that a mistake in the Lambda code and you end up breaking your parameter store. In the code that is available on Github for my previous version of the parameter encryption, there wasn't much of return values capture to know what was wrong and protection of the existing values.

With AWS SSM, that problem is sorted out. AWS SSM manages all of those parameters nicely for me, probably in the same way that I did with DynamoDB and Lambda, but now it is their job to maintain it for me and they have provided a very nice API to me for that.

Given that SSM does that for me, let's integrate that to my CloudFormation templates !


The problem
===========

Some might call me paranoid, but I believe that a system that involves humans in getting access to secrets, isn't a good thing. Of course, somebody smart with elevated access can workaround security and get information, however, that's not a reason not to try.

So, as I read the documentation of CloudFormation for the SSM:Parameter, I found out (as I write this article) that SecureStrings aren't a supported parameter type. Which means, no KMS encryption of the value of my parameter.


Lambda my good friend
=====================

As usual, a limitation of AWS CloudFormation is a call for a little bit of DIY with AWS Lambda.

Now let's think again about the use-case. Most of the time, what I am going to have to generate are a username and password for a service, a database. One rarely goes without the other.
So the code I have written can be totally customized for a more specific use-case, but it is simple

The Lambda function code and it's "demo" CloudFormation template can both be found `On my github repository of my personal organization <https://github.com/EWS-Network/ews-lambda-functions/tree/ssm/cloudformation>`_ . I created this new branch not to break anything from the existing ones.

What you need to prepare to make it work
========================================

A lot of things can be automated and so, I have also created `here <https://github.com/EWS-Network/ews-lambda-functions/blob/ssm/cloudformation/cf_secure_ssm_iam.yml>`_ the CloudFormation template to create the Lambda function role that you will need to have for your function to work (thanks to all previous readers of the different articles for those suggesstions towards improving the blog content).

Also, you will need to allow that role to use the KMS Key (IAM -> Encryption Keys -> Select the right region -> Select the key -> Add the role to users).

The policy statement to add if you edit the policy should be like:

.. code-block:: json

   {
   "Sid": "Allow use of the key",
     "Effect": "Allow",
     "Principal": {
     "AWS": [
         "arn:aws:iam::234354856264:role/lambdaSSMregister",
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }

Where to use it?
================

This Lambda function returns two values, which have been stored in to SSM as SecureString and only those with access to those strings can Get the value.

Here, instead of doing two different Lambda functions like in the past, one to generate the user and password and another to get those, I thought that the simple successful return of SSM to add the value was enough for me to be sure that the parameter has been successfully stored and created.

So, the Lambda function returns those values in Clear to the CF Stack so they can be injected directly to where it was needed (most likely, your RDS DB).

Now, those values stored in your Parameter Store can also simply be retrieved by automation scripts. For example, within that CF Stack where you have generated the user / password, you also created the name of those parameters. Given that you know the parameters, if you create an instance or an AutoScaling group, you can assign an IAM Profile to either and add an inline policy that grants GetParameter to those very specific parameters in the store.

Small demo
----------

This isn't practically a demo, but just a small walkthrough of what you stack creation process and output would be like.

First, create the stack

.. code-block:: bash


   aws cloudformation create-stack --template-url https://s3.amazonaws.com/ews-cf-templates/cf_secure_ssm.yml --parameters ParameterKey=KmsKeyId,ParameterValue=<your KEY ID> --stack-name demo-article

Now, on the AWS Console we can take a look

.. thumbnail:: /images/ssm-kms/0001.PNG

As our stack is successfully created, we can also see our parameters in the parameter store

.. thumbnail:: /images/ssm-kms/0002.PNG

AS you can see, we have access to our Parameters in the store.

To get it from CLI, simply run:

.. code-block:: bash

    aws ssm get-parameter --name demo-article-dbusername --with-decryption
    PARAMETER       demo-article-dbusername SecureString    7sqFNA-t


.. note:: From an operations perspective, do not forget to restrict users / groups IAM access to not be able to see the Value if they are not supposed to.



Conclusion
==========

This is just a start of my adventure with SSM, but by the length of that article and of the code versus the old one, it is certain that using SSM will help with 100s of use-cases.
