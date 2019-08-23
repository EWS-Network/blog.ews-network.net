.. title: AWS Lambda and Secrets Manager to bootstrap RDS instances
.. slug: aws-lambda-and-secrets-manager-to-bootstrap-rds-instances
.. date: 2019-08-22 21:51:08 UTC
.. tags: AWS,CloudFormation,Lambda,RDS
.. category: AWS
.. link:
.. description: Document explaining how to use RDS-Auth-Helper
.. type: text

AWS Lambda and Secrets Manager to bootstrap RDS Instances
=========================================================

Yet another post about Secrets, Lambda and CloudFormation but AWS Keeps coming with Services that previously we had to make up ourselves.
Distinctively from the previous posts around this subject which was mostly about using SSM Parameter Store SecureStrings, here we are going to use Secrets Manager which saves us a lot of the work.

For all the following work, you can find all the source code and templates here: GithubRepo_


Workflow
--------

.. thumbnail:: /images/rds-auth-helper/Workflow.jpeg


Secrets Manager Secret
----------------------

The new Secrets Manager and its CFN object are replacing completely the need for having a Lambda Function to generate and store the user/password to SSM Secure String.
In addition to that, they come with a very handy resource policy which in our example template we are going to use in order to give access to the Lambda Function instead of
having to add that to the IAM Role used by the Lambda Function (that way, we can create as many secrets as we want and not have to feed back into the role the new ARN of the secrets).

Extract from example_template.yml:

.. code-block:: yaml

   Resources:
     MasterSecret:
       Type: AWS::SecretsManager::Secret
       Properties:
         Description: String
	 GenerateSecretString:
         ExcludeCharacters: <>%`|;,.
         ExcludePunctuation: true
         ExcludeLowercase: false
         ExcludeUppercase: false
         IncludeSpace: false
         RequireEachIncludedType: true
         PasswordLength: 32
         SecretStringTemplate: '{"username": "toor"}'
         GenerateStringKey: password
	 Name: !Sub: '${AWS::StackName}/MasterSecret'

     MasterSecretPolicy:
       Type: AWS::SecretsManager::ResourcePolicy
       Properties:
       SecretId:
         Ref: MasterSecret
       ResourcePolicy:
         Version: '2012-10-17'
         Statement:
         - Effect: Allow
           Action: secretsmanager:GetSecret*
           Resource:
           - !Ref MasterSecret
           Principal:
             AWS:
             - !GetAtt LambdaFunctionRole.RoleId

Lambda Function
---------------

Recently started to use cookiecutter which allows me to very easily create new pieces of projects, and one I have done is specifically for Lambda functions.
With the Makefile, I can create / update the lambda function code in the AWS Account I want simply by running **make create** or **make update**

Here our Lambda function will have to run within a VPC in order to connect to the database. The choice of Subnets to use is yours, remember that you will either need a route
to the internet via NAT Gateway or sort for the lambda to connect to SecretsManager service or have a VPC Endpoint for it that the subnets are routed to.

As the workflow above explains, the Lambda Function will be created as you create your RDS Instance. The idea is then to use it in your CFN that creates your application or your microservices, call it as a custom resource and if it was successful in adding the access for your application, will complete.

.. warning::

   Note that this does not do anything in case you rotate the credentials in the SecretsManager.

At the time of writting this, the code works for PostgreSQL databases but soon will do for MySQL as well. I chose p8000 lib for the Lambda Function because it is pure python library and does not require any postresql libs.


.. _GithubRepo: https://github.com/lambda-my-aws/rds-auth-helper
