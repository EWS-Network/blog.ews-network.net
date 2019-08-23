.. title: CFN Macro Example - S3 Bucket replicated with KMS Key
.. slug: cfn-macro-example-s3-bucket-replicated-with-kms-key
.. date: 2019-08-23 23:42:21 UTC
.. tags: AWS, CloudFormation, Macro, KMS, IAM, S3, Encryption
.. category: AWS
.. link:
.. description: Simple Example on how-to use CFN Macro to help with multi-region bucket encryption
.. type: text

AWS CloudFormation Macros
=========================

It has been a few months since AWS CloudFormation macros are out and I believe this to be one of the biggest game changer since Custom resources. Yes, I do.
Why is that? What are CFN Macros ?

CloudFormation macros are small snippets you can insert and use via **Transform** instructions in a template. They are quite smart in the sense that it will go about those in order of depth in the template: if your macro wants a parameter that a **Fn::Sub** can resolve, it will first resolve the value from the Sub and send it to your macro.

So when to use a Macro: macros are insanely good and provide what most people today will do in AWS CDK or used to do in CI/CD pipelines with Troposphere (and some people in the dark I have seen render templates with Jinja :facepalm:): generate with Python CFN templates so that the template fits to the region it is into (I am thinking VPC in general being one of the most common challenges).

As I am here, I want to give a huge shoutout to the people helping to work on Troposphere, it has been a few months since I started using it and it is simply awesome, Python native, and it will be your swiss knife for CFN Macros.


When macros go crazy
--------------------

Here is an example of me using CFN macro to generate an entire VPC template only within the macro (VpcMacro_ can be found here). That's how deep you can go into using those for.

As the README of the macro says, the template is as simple as:

.. code-block:: yaml

   AWSTemplateFormatVersion: 2010-09-09

   Parameters:
     VpcCidr:
       Type: String
       Default: 10.242.0.0/22

   Transform:
     - cfnmacro-vpc


Crazy, right ?

Use a macro or a custom resource ?
----------------------------------

In case of the VPC above, a macro is mandatory. However, and this is where our S3 bucket example is going to come in: when should I use a CFN Macro vs a Custom Resource?
Well, the answer as usual is going to be, **it depends**.

However, I believe that Macros give something that Custom Resources do not in the same way: Logging.
But before we come to that detail, there is something else that is worth mentioning:

Ever found yourself in the case where your custom resource uses the wrong version of the lambda code ? Or there is a python bug which you didn't catch and now your function will simply never respond to CFN and CFN will forever wait on your function to respond until CFN finally times out ? Sounds familiar ?

Well, here with Macros, it is not the case: If you have use the SAM transform which allows to pack a Lambda Function / Application together as a single object and SAM will take care of creating role, version, alias etc., it does that before trying to create any resources, by taking the template and rendering it.

Macros are the same: a macro will be called in order to render a final template before CFN goes about creating the resources within.

Like I said earlier, the logging of CFN macros is also quite nice: when you create it, you can indicate a CloudWatch log group where to log the call of the render and it will tell you whether or not the render of the macro was successful.

.. note::

   A perfect example of when not to do that would be if you are fetching a secret value which you do not want to have in clear text in the template, but rather use !GetAtt on your custom resource.

Bucket with KMS encryption and replication role. Where is the problem in CFN ?
------------------------------------------------------------------------------

Well, I am glad you asked ! Actually, there is no problem at all with CFN. You can absolutely have 1 single template, use parameters and have CFN create the destination bucket then the source bucket and it will just work. However, you need an IAM role to perform the replication. And in order for the replication to work, you will need to put in the IAM policy of that role the KMS Key ID ARN ! Yes, helas, the KMS Key Alias will not work in the IAM policy. The documentation is kinda contradicatory, says you can put the Alias ARN for some actions to perform, but then the support tells you otherwise, and it gets very confusing.

So to start with, before I knew about that issue, I was very happy in CFN to render the KMS Key Alias based on a regexp and use that on both sides (source and destination buckets templates).
Some people told me then, that using CDK or tropopshere then I could generate the "source" template and input the KMS Key ID ARN then, but I did not quite like it. I want to have a template that is going to work everywhere, on any account, and CFN is going to deal with getting the right value.

However, as I said earlier, I would rather CFN fail and tells me so before it is too late. So, I am using a Macro at the very start and that creates the Stack by injecting the right value in the template.


The template
-------------

Well, I wanted to have just the one template for both the source and replication region, and simply use CFN Conditions to create some resources.
For all the following examples, you can find the source code `here<SourceRepo>`_

The template is generated by Troposphere, just because I like to use it. But what's interests us is the template.yml

The Replication Role is as follows:

.. code-block:: yaml

  ReplicationRole:
    Condition: IsSourceRegion
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          - Action:
              - sts:AssumeRole
            Effect: Allow
            Principal:
              Service:
                - !Sub 's3.${AWS::URLSuffix}'
        Version: '2012-10-17'
      Policies:
        - PolicyDocument:
            Statement:
              - Action:
                  - kms:Decrypt
                Condition:
                  StringLike:
                    kms:ViaService: !Sub 's3.${SourceRegion}.${AWS::URLSuffix}'
                Effect: Allow
                Resource:
                  - !GetAtt 'BucketEncryptionKey.Arn'
              - Action:
                  - kms:Encrypt
                Condition:
                  StringLike:
                    kms:ViaService: !Sub 's3.${ReplicaRegion}.${AWS::URLSuffix}'
                Effect: Allow
                Resource:
                  - Fn::Transform:
                      - Name: cfnmacro-kmskey
                        Parameters:
                          KeyRegion: !Ref ReplicaRegion
                          KeyAlias: !Sub 'alias/${ReplicaRegion}/${BucketName}'
            Version: '2012-10-17'
          PolicyName: !Sub 'KMSAccess'

As you can see, we call the macro all the way down at the Resource of the policy. So, what does that do ?

* Sub generates the KMS Key Alias which we create via a defined Regexp
* Macro catches the Alias, scans the keys in the replica region, matches the alias and the key
* Returns the string of the KMS KEY ID ARN

And that is all there is to the macro really. Again, one could use a custom resource in this particular example, but I also find the way to return the value to CFN much easier and straight forward with the macro.

So why bother with macros ?
---------------------------

Ah, so we have seen 1 use-case where we have a VPC created based on the CIDR alone. Okay, why not!? Then we have another example for the KMS key ID, which again, could be done with existing method of using a CFN Custom resource.
So why go down the rabbit hole ?

The Bucket example really was the first thing I did to try out macros, and it turns out to work very simply and superbly. And again, the tiny extra logging CFN puts to tell you what went wrong when rendering the complete template is quite helpful.
Now, obviously, the KMS Key would have a limitation: it looks up keys within the same account. Extra effort would be needed if we wanted to do replication from one bucket in an account to another, including at the KMS key level to allow the role to use
both those keys.

However, the VPC, is something you might want to use everywhere and have this simplicity of saying that the only parameter you need, is the CIDR.
Well that is where macro sharing across accounts is bliss. You can share a macro across multiple accounts.

Or better yet, if you have a product in Service Catalog, and use CFN template, so long as the macro exists and is available to CFN to use, you can then just create one product in Service Catalog, share it to all your accounts in your Organiation Unit, and
not have to worry about dispatching it in all accounts.

What is next ?
--------------

Well the primary objective of this Bucket replication with KMS keys was to use for CloudTrail logs and rotation to Glacier etc.
So the next step for me is to use CFN Macros to go and generate the entire KMS Key policy and inject it inline with all my AWS Accounts in the OU that are going to federate their CloudTrail logs to it.


I have a bunch of ideas for a library of cfn macros which would help a wider range of microservices. So stay tuned for future CFN Macros work using Lambdas Functions and Layers.

.. _SourceRepo: https://github.com/lambda-my-aws/replicated-bucket-kms-encrypted
.. _VpcMacro: https://github.com/lambda-my-aws/cfnmacro-vpc
