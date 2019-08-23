.. title: Protect your CloudFormation sensible values and secure them with KMS and DynamoDB
.. slug: aws-autogenerate-passwords-for-resources-and-store-them-in-dynamodb
.. date: 2016-11-03 00:42:00 UTC
.. tags: AWS, CloudFormation, KMS, RDS, Lambda, DynamoDB
.. category: AWS
.. link:
.. description:
.. type: text


The use-case
============

CloudFormation it probably one of my favorite AWS service. It allows hundreds of people today with the deployment of all the architecture resources required for their applications to run on AWS : Instances, Databases etc.

I use CloudFormation all the time, as soon as there is a piece of architecture that I can use in multiple places, in different environments, this becomes my default deployment method (even just to deploy a couple VMs..).

Some of those resources require a particular care : some of the parameters or some of the values have to be kept secret and possibly as less human readable as possible.

Today I want to share with you a thought and the process I have decided to go with for those delicate resources that we might want to secure as much as possible, removing human factor out of the process. As part of the use-case, I want a fully-automated solution which I can re-use anytime and will guarantee me that those values are never the same from one stack to another as well as secured, both in the recovery sense and security.


Different approaches
--------------------

Here, I am going to work with a very simple use-case : for my application, I neeed a RDS DB. That resource requires a password to get created and for consumers to get connected to it onwards.


1 - Generate from the CLI and hide the values
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In our CloudFormation templates, we have the ability to set a "noEcho" on some of the parameters, so after creation we can't read the value. I find this really useful in the case I have some settings I have access to as an elevated administator of the Cloud which I don't want others to be aware of (ie: The Zone ID of a Route53 managed domain). Generally speaking those are values I could have a default value for in the templates (assuming the templates access is as much restricted as the default value you want to keep from others) and once we describe the stack, won't be displayed in clear-text.

Pros :

- Very easy to implement (noEcho on the parameter)

- You can set default values and restrict template access (for non authorized users, deny  "cloudformation:GetTemplate")

Cons:
- Values are known by authorized users and show up in clear text without additional level of restriction

- If you have set a default value, you could forget to change it

- The values exist only in the templates and stack updates could not affect the resources we wanted to update.

2 - Leverage Lambda, KMS and DynamoDB
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

With CloudFormation, we can create CustomResource(s) and point it to a Lambda function. This Lambda function will execute some code and get treated as any sort of resource in the Stack. Now, in our use-case where we have to set a Master password for our RDS Instance, we want that password to be different for every stack we create, store it somewhere so we can retrieve it, but as we store it, it has to be protected so it won't be human readable.

That is true for a password, you could extend that function to encrypt and store any sort of information you would have compute generated and encrypted. You might just want the randomness, or the encryption, or both. It's up to you.

Pros:
- Every stack resource that needs some random value will get a new value everytime

- Each random value will be encrypted with an AWS managed encryption key (using AWS KMS), and DynamoDB will store it region-wise.

- We can backup our DynamoDB table to a S3 bucket (most likely, encrypted as well with a different key) for recovery (and leverage S3 replication to backup up globally).


Cons:

- Can look like an overkill for not so much (I honestly had that thought at first)

- Requires a good understanding of how CloudFormation and Lambda custom resources work together.

At the end of the day, the cost of that solution is probably around 1 USD per month, for the KMS key + the Lambda function + DynamoDB storage. So, it is a neutral argument, unless you end up with a bazillion of stacks and stored resources. If you think that would be your case, see the `At Scale`_ section.

3 - Use S3 bucket
~~~~~~~~~~~~~~~~~

Update on : 2016-11-07 in reponse to `Harold L. Spencer <https://twitter.com/hspencer77>`_. Thanks Harold for that proposal ;)

Here, instead of going with DynamoDB to backup the passwords etc, Harold asked if it would be better to use S3 to store the passwords : as the stack is created, we would create the same kind of record in a file, which we would encrypt and store into a S3 bucket (itself encrypted). So, here is what I see as pros and cons:

Pros:

- No need for DynamoDB, so potentially removes the Capacity Units for reads and writes more expensive than S3 Get/Put
- S3 has a replication mechanism multi-region, so we can save our data as we need
- S3 has a versioning system, so we could version each new configuration if need be.

Cons:

- No query capabilities in S3, so to find the file you are looking for, it needs to be unique and already need to know what is key is.
- The parsing necessary depending on the file architecture made in the S3 bucket or the payload file could make it more difficult to update / delete the file
- Even with versioning, you might not be able to determine what went wrong if you corrupted the file (or at least as complicated as with DynamoDB)

At this point, I would agree that using S3 for storage could be a viable and even cheaper solution. However, as said in the `At scale`_ section, here is why I think this might be a alike:

- For both DynamoDB and S3, you have to make a KMS call to encrypt and decrypt the payload that is going to be stored. Regardless of the scale, you call KMS the same way in both cases..
- In this very particular use-case, the chances that the DynamoDB table read requirements higher than the free-tier (25 Units) as extremly low.
- With the right combination of automation, you can as easily backup to one (or more) S3 bucket(s) a DynamoDB table as would a S3 bucket with replication.


How to ?
========

So, at this point, I have decided to use that second method for all my RDS resources I will create with CloudFormation. Here is what we need to do:

1. Create a DynamoDB table (per region) we are going to use to store our different stacks passwords into
2. Create a KMS key (0.53$ per month, so ..)
3. Create the Lambda functions

   A. Create a Lambda function to generate, encrypt and store the password in DynamoDB
   B. Create a Lambda function to decrypt the key for both CloudFormation and any Invoke capable resource

4. Create the cloudformation resources in our stack to generate all of the above


Here is a very simple diagram of the workflow our CloudFormation stack is going to go through to create our RDS resources.


.. thumbnail:: /images/lambdadyndbkms/Lambda_CF_KMS.png



----------

1 - The DynamoDB table
======================

Why DynamoDB ? Well, because it is very simple to use and very cheap for our use-case. Not to mention, you won't even go over the free-tier. But at first, DynamoDB is a NoSQL service that you can use directly via API calls as long as the consumer has permissions to write/read from it.
Very simple : we are going to create a table with a primary key and a sort key (ensure we aren't doing anything stupid). The DynamoDB table structure is discussable. Please comment if you have suggestions :)


Create the table - Dashboard
----------------------------

In your Dashboard, go to the DynamoDB service. There, start to create a new table.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-26-49.png
   :alt: Create the table


With CloudFormation, I use extensively the "Env" tag to be able to identify all other resources via mappings etc. To create my table, I decided to pair the stack name (which is unique inthe region, granted) and this env value. That way, it sorts of ensure me that I am not overwritting a key in the occasion of a mistake and instead of creating a new item, the function will update the field and you could possibly loose the information ..

There, we are going to use only a very little of the writes and reads. Therefore, there is no need to go with the default values of 5 RSU for reads and writes.
Wait for the table to be created (should only take a minute really ..). Make sure all settings look good.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-28-38.png
   :alt: Table is created. Ready to go.


------------

2 - The KMS Key
===============

DynamoDB doesn't come up with a native encryption solution, and furthermore, all data is potentially cleartext at a certain extent. So, prior to storing our password, we are going to leverage KMS to cypher our password.
The good thing is : KMS has probably less risks of loosing your key than you have to loose your USB key or tape, for old-school.

Create the KMS key - Dashboard
------------------------------

In IAM, select the right region and create a new key.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-29-28.png


I have decided to call my key so I have a very simple way to identify what each key does. Maybe something to exploit to mistake the enemy ? ^^ Make sure you get KMS to manage it for you ..


.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-31-49.png


The administator of the key are the users / roles who can revoke (delete) a key or change its configuration. Choose very carefully the users. Here, I select my user as the only administator of the key.


.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-32-35.png


Now, just as for the admins, I select which users can use the key to encrypt / decrypt data with it. For now, I only select my user. Later on, we will grant those user rights to our IAM role for the lambda functions.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-33-15.png


Final validation of the IAM policy that is for the key itself. This is a key policy, check it twice !

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-33-32.png


Click on Finish to complete the key creation.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_20-35-28.png


Here ! Your key has been created and we can start using it. Note the KeyID somewhere or remember how to come back here, we will need that key for later.

------------

3 - The Lambda functionS
========================

AWS Lambda .. How awesome service, right ? Write some code, store it, call it when you need it, with no additional pain. So, this is where you discover that I am a Python developer, and as such, all my Lambda functions are done in Python. A bit of history : I started with one of the first versions of boto 2. And, it was nice, but, once I tasted some of boto3 and its documentation .. this is where the sweetness comes ;) boto3 really makes it super easy for us to talk to AWS.


So, as for the code, you will be able to find it on gists / github.com in links as we go through that script.


3A - Generate, encrypt and store the password
---------------------------------------------

The lambda function's role
~~~~~~~~~~~~~~~~~~~~~~~~~~

The lambda function runs assuming an IAM role. Here we need a couple rights:

- Write-only to the dyamoDB table we created earlier
- Use the KMS key we used earlier

And that's it. Remember, in AWS as in general, the less privileges you give to a function, the lesser the risks of exposing problems where someone gains access to it.

Start with going in IAM again, in the roles this time. Now here, click on create a new role

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-10-35.png


I usually prefix the role with the roletype. Here, lambda as this role will be used by the Lambda function. Now, cfEncrypt tells me this is the role we will use for the encryption function.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-11-11.png


In the AWS Services roles, select AWS Lambda. This is what's called the trust policy. It simply exposes that for this role, IAM will allow API calls from the lambda functions.


.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-11-29.png


We are going to select 2 AWS managed policies as AWS preconfigured those for general purpose. Those policies are necessary for Lambda to create the logs files and other reports. If you find those too permissive, feel free to change them. Beware that you have to know all the details around CW and Lambda functions logging.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-12-01.png


Here, final step. We are good to create the role :)

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-12-38.png


Now we have the baseline for our Lambda function to have the appropriate powers, we still have to create a policy so it will be allowed to write (or read for the decrypt function) to our DynamoDB table. The policy should be as follows :

.. code-block:: javascript

   {
        "Version": "2012-10-17",
	"Statement": [
		{
		"Sid": "Stmt1478169389000",
		"Effect": "Allow",
		"Action": [
		   "dynamodb:PutItem"
		],
		"Resource": [
		   "arn:aws:dynamodb:eu-west-1:account_id:table/ewspasswordseuwest1"
		]
	   }
	]
   }


So, via the Dashboard again, here is simple run-through how-to create the policy properly.
Go to the IAM service, then in policies section, then click on "Create policy" button. On the next screen, select "Policy Generator"

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-32-37.png

The policy generator is a very simple and efficient tool to help you build the JSON policy if you aren't familiar / used to write and read JSON IAM policies.

In the service dropbox, select "AWS DyanmoDB", then select the "PutItem" Action. In the Resource ARN filed, use the DynamoDB ARN of your table. This is the ultimate way to be sure that the policy won't allow any other action against any other table.

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-35-04.png

Once you've clicked you will see a first statement has been created for the policy. For now, we don't need any other statement for the policy, so now click on "Next step"

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-35-14.png

The last step before creation is to review the JSON and the policy name / description. Once you have named your policy and description, click on "Create policy"

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-36-09.png

At this point, we simply have to attach the policy to our existing role.
So back to the roles in the IAM dashboard, select the "lambdaCfDecrypt" role. In the role description page, select "Attach policy". You are taken to a new page where you can select the role to attach:

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-38-43.png


Create the function
-------------------

So, first of all we want to generate a password that will comply to our security policy and works for our backend. In my use-case, it is a MySQL DB so, as it is, I go for letters (lower and major cases), numbers and a special caracter.
One the password is generated (0.05ms later ..) we are going to call KMS and use our key to cypher the password text (add another 20ms). Then, we write the base64 of the whole thing to our DynamoDB table with all the attributes necessary to make sure we are making it unique.

In this part, I am going to do it only via the Dashboard so it stays user friendly.

In the Lambda dashboard, go to create function. Skip the blueprint selection by clicking on the next step right away in the top left corner.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-09-14.png

As Lambda is an event triggered function, you can define trigger to execute the lambda function. Here, we don't need to configure a specific trigger as we are going to call our Lambda function only when CloudFormation will.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-09-48.png

"Oh oh .. lots of settings here" - Don't panic ! We have already prepared all the necessary for this step. Here, we give our function a lovely name, a meaningful description and the code. To make the tutorial user-friendly, I have selected "Upload a Zip file" just so everything fits within the page, but you will use the code `here <https://goo.gl/WLfppi>`_ and copy-paste it inline.

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-24-35.png

Here we are, ready to create the lambda function :)

.. thumbnail:: /images/lambdadyndbkms/2016-11-02_22-28-55.png

------------

3B - Decrypt the password
-------------------------

The other lambda function's role
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As you guessed, here we are going to create a role that is just like the previous one, but instead, we are granting read-only access to the DynamoDB table and decrypt rights on the KMS Key.

To create the lambdaCfDecrypt function, follow exactly the same steps as described in the "Encrypt" function.

.. code-block:: javascript

   {
        "Version": "2012-10-17",
	"Statement": [
		{
		"Sid": "Stmt1478169389000",
		"Effect": "Allow",
		"Action": [
		   "dynamodb:GetItem",
		   "dynamodb:Query"
		],
		"Resource": [
		   "arn:aws:dynamodb:eu-west-1:account_id:table/ewspasswordseuwest1"
		]
	   }
	]
   }

Once you've clicked you will see a first statement has been created for the policy. For now, we don't need any other statement for the policy, so now click on "Next step"

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-36-51.png

The last step before creation is to review the JSON and the policy name / description. Once you have named your policy and description, click on "Create policy"

.. thumbnail:: /images/lambdadyndbkms/2016-11-03_10-37-53.png


Create the function
~~~~~~~~~~~~~~~~~~~

To create the function, follow the exact same steps as for the cfRdsPasswordGenerate function.
The code is `in this gist <https://goo.gl/xegx4N>`_, so you can put that inline.

.. note:: Do not forget to change the role of the function to the lambdaCfDecrypt role.

4 - Put it all together with CloudFormation
===========================================

Now we have created our Lambda functions and tested those, it is time to get our Cloudformation running. As for the lambda functions, you can find the full CloudFormation template on my Github account or here.

So, you might know all AWS:EC2:Instance resource attributes, but do you know the custom resource ? Here is the special one that we are going to call our Lambda function with parameters and that is going to generate and capture the values we want. Here is a very simple snippet of those two resources that we want to get through our Lambda functions (those go in the parameters object of your template).


.. code-block:: javascript

   "lambdaDBPassword": {
      "Type": "AWS::CloudFormation::CustomResource",
      "Version": "1.0",
      "Properties": {
        "ServiceToken": "arn:aws:lambda:eu-west-1:account_id:function:cfGeneratePassword",
        "KeyId": "arn:aws:kms:eu-west-1:account_id:key/key_id",
        "PasswordLength": "20",
	"TableName": "mypasswordtablename"
        "Env": {
          "Ref": "Environment"
        },
        "StackName": {
          "Ref": "AWS::Stackname"
        }
      }
    },
    "lambdaGetDBPassword": {
      "Type": "AWS::CloudFormation::CustomResource",
      "DependsOn": "lambdaDBPassword",
      "Version": "1.0",
      "Properties": {
        "ServiceToken": "arn:aws:lambda:eu-west-1:account_id:function:cfGetPassword",
	"TableName": "mypasswordtablename",
        "Env": {
          "Ref": "Environment"
        },
        "StackName": {
          "Ref": "AWS::StackName"
        }
      }
    }


If the resources creation succeded, how can we get the password out of the lambdaGetDBPassword resource ?

Below is a very very small snippet of a RDSInstance resource for which I volountarily kept only the DB password attribute. Here, we first ensure that the lambda custom resource worked and could be created successfully, using the "DependsOn" attribute. Then, for the password, we simply have to get the password out of it, using the function "Fn::GetAtt".

The attribute name is the one that in the code we previously set in the "Data" object of the response.

.. code-block:: javascript

    "rdsDB": {
      "Type": "AWS::RDS::DBInstance",
      "DependsOn": "lambdaGetDBPassword",
      "Properties": {
        "MasterUserPassword": {
          "Fn::GetAtt": [
            "lambdaGetDBPassword",
            "password"
          ]
        }
      }
    }


I had the question : why have 2 separate functions used by CloudFormation instead of have the generator function return the cleartext directly ?

Well, it might make you save around 10 seconds in the stack creation to use only one function, but I find use a function to decrypt a very nice way to be sure that at the creation of the Lambda function, the "reverse" process of get and decrypt the password works as expected. That way, you know that you can reuse that function in different places again and again and keep the logic very simple.

Conclusion
==========

This is a very simple example of all the possibilities Lambda and CloudFormation offer us. I hope this will help you in your journey to AWS and automation.

At scale
--------

When we created the DynamoDB table, as you can see, we have set the read and write capacity units to 1, because this table will be potentially used only when we will create a new stack for dev/test and our resources need a password. But if tomorrow you find yourself in the position where you have 100 RDS dbs, and for each individual DB you have 10s of consumers which when they initialize themselves, will call our Lambda function. Lambda won't be our limitation here, but DynamoDB might be. In this case, you might want to look at the table metrics, and maybe raise the read capacity so you can have more consumers potentially reading all at the same time without a throttle.

Also, it is worth mentionning that KMS has a cost per call. So again depending on the kind of resources that need to decrypt the information with the key, you might have to make sure that your resources are asking for decrypt only when it is necessary (Free-tier ends at 20k requests globally, then goes at 0.03$ per 10k requests).

*Edited on 2016-11-07*

The 3rd option, `3 - Use S3 bucket`_, could give us an alternative, but, at risks : if the number of calls you have to make to KMS to get and decrypt the payload becomes a struggle for your bill, if you are super confident in your ability to write S3 bucket policies and your VPC network configuration, you could have the payload non-encrypted in the bucket, leverage VPC Endpoint to S3, and have the instances / resources that need the information and get it in clear-text.

At your own risks if you happen to store your non-protected root password for your black-box.
