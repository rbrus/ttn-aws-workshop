# The Things Network AWS Workshop
This workshop will help you get a Things Node connected to the AWS Cloud, where we'll use the power of AWS services to quickly
query the activity of your alarm and notify you when there's a burglar at your doorstep!

## AWS Architecture
During the workshop, we will use multiple AWS services and combine them in a data workflow. The workflow will branch out into:  
- a flow which will detect motion while it's dark and send out a SNS notification to your mailbox. (hands-on part of the workshop)
- a flow which will store the messages into S3's object storage (after a small transformation) and make the data queryable via
 AWS Athena. (will be demoed, the step by step log is also provided within this document).
![architecture](/assets/workshop.png)

## AWS Services

Below one can find all the AWS Services that will be used during this workshop. Each service is listed under an `AWS Service
Group` which is an abstract level categorising the AWS Services. Only a brief description is foreseen, during the workshop
we provide a bit more information about each service. Detailed information can be found via [Explore AWS Products](https://aws.amazon.com/)

**Group:** `Compute`
* **EC2:** Amazon Elastic Compute Cloud is a web service that provides secure, resizable compute capacity in the cloud.
* **Lambda:** small code blocks or functions which act on or transform data, without having to provision or manage servers.
* **Elastic Beanstalk:** is an easy-to-use service for deploying and scaling web applications and services.

**Group:** `Management Tools`
* **CloudWatch:** is a monitoring service for AWS cloud resources and the applications you run on AWS.
* **CloudFormation:** allows you to use a simple text file to model and provision, in an automated and secure manner,
all the resources needed for your applications across all regions and accounts.

**Group:** `Internet of Things`
* **AWS IoT:**  is a managed cloud platform that lets connected devices easily and securely interact with cloud applications
and other devices.
This is were we'll see our devices and can subscribe/publish messages.

**Group:** `Application Integration`
* **Simple Notification Service:** Allows you to send notifications to a mobile device, e-mail account, http endpoint etc.

**Group:** `Analytics`
* **Athena:** interactive query service that makes it easy to analyse data in AWS S3 using standard SQL.
* **Kinesis:** AWS message log, which makes it easy to collect, process, and analyze real-time data and deliver it to various
endpoints (such as S3).
* **QuickSight:**  is a fast, cloud-powered business analytics service that makes it easy to build visualizations, perform
ad-hoc analysis, and quickly get business insights from your data

**Group:** `Storage`
* **S3:** AWS object storage which allows you to store files of data, video, images,...

## Workshop resources
We will be using various code snippets during the workshop, which you can simply copy-paste from this repo.  

[TTN CloudFormation template](ttn-cloudformation-template): We made a small change to the default AWS CloudFormation template
provided by The Things Network. Therefore you are advised to use this template during the workshop. <br>
[Lambda to SNS notification](lambda-sns.js): We'll use this Javascript snippet for an AWS Lambda function. It will send out a
notification to your email inbox via SNS when it detects movement at night. <br>
[Lambda to flatten json](lambda-flatten-json-kinesis-records.js): We'll use this Javascript snippet for an AWS Lambda function.
It will take multiple records from a Kinesis stream and transform the json message of the TTN Node to a flat json with only the fields we require. <br>
[Lambda test record](lambda-test-record.json): You can use this sample Kinesis record to test your "flatten json" lambda. <br>
[Athena create table statement](athena-create-table.sql): We'll use this "create table" statement to create a table in Athena,
which maps on the data we stored in S3.

## Workshop log
This log will guide you through the steps of the workshop.

## Generic setup

### 1. Setting up the TTN Integration with AWS
During the workshop there will be one LoRa Gateway on which several Things Nodes will be connected via LoRa.

The TTN Integration with AWS brings the LoRaWAN to AWS IoT, which syncs thing registry, syncs thing shadows, acts on uplink messages and
sends downlink messages. This integration runs in your AWS account and security context and can connect to The Things Network public
community network and private networks.

To make this possible TTN has created a `CloudFormation` template which will setup the required:
* `IAM` (Identity and Access Management) roles
* `EC2` node on which the actual application responsible for the synchronization between the two IoT platforms will be deployed.
* `Elastic Beanstalk` configuration which is the TTN provided web application.
* `CloudWatch` configuration that captures and reports the TTN web application metrics.
* ...

To deploy the `TTN CloudFormation` template one can use the [Quick Start guide](https://www.thethingsnetwork.org/docs/applications/aws/quick-start.html) provided by The Things Network. However, you need to take one small change into account.
Instead of using their S3 template URL link, you should use [our template](ttn-cloudformation-template), and upload it during the
"select template" step of the guide.

`Note:` Before following this quick start guide one has to create a secure `KeyPair` which is
required during the `CloudFormation Stack` configuration. See section : `1.1 Create KeyPair` in this document.

**OR**

Use the provided abstract description below.

### 2. Create KeyPair
During the configuration of the CloudFormation stack one will be requested to provide a `KeyPair`. Amazon EC2 uses public–key cryptography
to encrypt and decrypt login information. The public and private keys are known as a key pair. So the only purpose for this `KeyPair` is
to have the private key which is required when one wants to login directly from their local development machine onto an `EC2 node`.

During this workshop there is no need to login to the `EC2 node` which is used to run the `TTN Elastic Beanstalk application`. But since an
`EC2 node` is automatically created during the deployment of the `CloudFormation template`, it is mandatory to provide one. The private key
belonging to this `KeyPair` can only be downloaded once, so make sure you store it in a safe place, since this will be the last time you'll be able to download it.

1. Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.
2. In the navigation pane, under **NETWORK & SECURITY**, choose **Key Pairs**. <br>
`Note`: The navigation pane is on the left side of the Amazon
EC2 console. If you do not see the pane, it might be minimized; choose the arrow to expand the pane.
3. Choose **Create Key Pair**.
4. Enter a name for the new key pair in the **Key pair name** field of the **Create Key Pair** dialog box, and then choose **Create**. <br>
`Note`: for this workshop it would be best to identify each resource with a **ttn** prefix. It would make life easier in cleaning your
environment after the workshop. So **set as Key pair name** => `ttn-workshop-key-pair`
5. The private key file is automatically downloaded by your browser. The base file name is the name you specified as the name of your
key pair, and the file name extension is .pem. Save the private key file in a safe place.

### 3. Deploy TTN CloudFormation Template
1. Open the Amazon CloudFormation console at https://console.aws.amazon.com/cloudformation/.
2. Choose **Create Stack**
3. In section **Choose a template** select **Upload a template to Amazon S3**
4. Download the content of the [TTN CloudFormation template](ttn-cloudformation-template) file and upload it to S3 via **Choose File**. <br>
`Note`: The best way to download the content is to open the file. Select **Raw** and copy the content in a new created file or with some
browsers one can select the shown raw page content, right mouse button and do a **save as**.
5. Choose **Next**
6. **Stack name** => `ttn-workshop-app`
7. **AccountServer** => keep the default (don't change)
8. **AppAccessKey** => `TBD`. This is the application access key which can be found in your TTN application.
9. **AppId** => `TBD` This is the application id which can be found in your TTN application.
10. **DiscoveryServer** => keep the default (don't change)
11. **EnvironmentName** => `ttn-beanstalk-env`. This will be the name given to the automatic created `EC2 node`
12. **InstanceType** keep the default (don't change)
13. **KeyName** => `ttn-workshop-key-pair`
14. **ThingShadowDeltaFPort** => keep the default (don't change)
15. **ThingSyncEnabled** => keep the default (don't change)
16. **ThingSyncInterval** => Choose `1m`
17. **ThingTypeName** => `ttnLoraWan`
18. Choose **Next**
19. In the next window nothing has to be changed. Choose **Next**
20. Check **I acknowledge that AWS CloudFormation might create IAM resources.**
21. Choose **Create**

Follow the progress of the Stack creation. This might take several minutes.

### 4. Check the EC2 Created Instance
1. Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.
2. In the navigation pane, under **INSTANCES**, choose **Instance**. <br>
`Note`: The navigation pane is on the left side of the Amazon
EC2 console. If you do not see the pane, it might be minimized; choose the arrow to expand the pane.

One should see an entry with the name `ttn-beanstalk-env` as which we provided during the creation of the Stack. Notice the
**Instance Type**, **Availability  Zone** and the **Instance State**.

### 5. Check the TTN Elastic Beanstalk Application
1. Open the Amazon Elastic Beanstalk console at https://console.aws.amazon.com/elasticbeanstalk/.
2. Choose **ttn-beanstalk-env** and notice the **Health**
3. Choose the **URL** in the top left corner containing `ttn-beanstalk-env....elasticbeanstalk.com`. Notice that a new page
is loaded with title **The Things Network Integration**.

### 6. Check the TTN Application Metrics via CloudWatch
1. Open the Amazon CloudWatch console at https://console.aws.amazon.com/cloudwatch/
2. In the navigation pane, choose **Metrics**  
    `Note`: The navigation pane is on the left side of the Amazon CloudWatch console.If you do not see the pane, it might be minimized; choose the arrow to expand the pane.
3. Under **Custom Namespaces** choose **TheThingsNetwork/Integration**
4. Choose **AppId, DiscoveryServer**  
    If your device is not switched on the only metric count that we should see is for the metric **ThingsCreatedInAWS**.  

5. Choose in the menu tab **All metrics** the metric name **ThingsCreatedInAWS**
6. In the menu tab **Graphed metrics**, set the **Statistic** column to **Sum**. 

Notice that the Graph shows a count with value 1. So we can conclude that once you've got the integration up-and-running,
it will automatically sync the devices of The Things Network to AWS IoT

With the above checks we have showed different ways for you to check if the **TTN Integration with AWS** was a success.
The only thing that needs to be checked is if data from the ThingNode is coming into AWS.

### 7. Verify the IoT Data of Your Things Node in AWS IoT.

1. open AWS IoT in the AWS Management Console: https://console.aws.amazon.com/iot/home
2. in the menu on the left, go to **Manage** -> **Things**. Here you should see your TTN things now.
3. in the menu on the left, go to **Test**. Here we'll try to subscribe on the MQTT messages published by your TTN Node.
4. in the **subscription topic** field, enter `<ttn-app-name>/devices/<ttn-node-name>/up`, were you'll have to replace the placeholders
with their actual values.
5. click on the **Subscribe to topic** button. When you interact with your Things Node (by moving with it, clicking the button, etc), you
should see your messages in AWS IoT.

## SNS notification flow

Within this flow we will illustrate how one can send a notification to an email account based on a certain condition.
The condition is that when the ThingsNode sends a message based on a `motion` event and the light sensor indicates it is dark
(light sensor value < 35) it will trigger a SNS notification.

### 1. Setup Simple Notification Service
Now we'll setup an SNS topic and subscription, to send out the notifications.

1. Go to SNS in the AWS Management console: https://console.aws.amazon.com/sns/v2/home
2. In the menu on the left, select **Topics**. A SNS topic is a communication channel to send messages and subscribe to notifications.
3. Click the **Create new topic** button, with name `ttn_alarm_notification`. Confirm with **Create topic**.
4. Your SNS topic is now created. Note the ARN for your SNS topic, as you'll need this in the next step.
5. In the menu on the left, select **Subscriptions**.
6. Click the **Create subscription** button. Enter the topic ARN, select **Email** as the protocol, fill in your email address as the endpoint and confirm with **create subscription**.
7. SNS will now send you an email to confirm your SNS subscription. Confirm it by clicking on the link in this email.

SNS is now setup to handle notifications.

### 2. Setup lambda
We'll now setup a lambda function, which will send out a notification to your SNS topic when it detects movement while dark.
As a trigger for this lambda, we'll use AWS IoT, so that it gets triggered on every AWS IoT message of your Things Node.

1. Go to the Lambda service in the AWS Management console: https://console.aws.amazon.com/lambda/home
2. In the menu on the left, go to **Functions**.
3. Click the **Create function** button.
4. Select **Author from scratch**.
5. Fill in the basic information by providing a name to your lambda function and role.
6. The role is used to provide your Lambda function with certain permissions. In the role dropdown, choose **create new role from template**.
7. In the **policy templates** dropdown, search for the **SNS publish policy**.
8. finalize your lambda function creation with clicking on the **Create function** button.
9. Now your enter the main screen of your Lambda function. At the top you have the **Designer panel**, at the bottom the **Function code** panel.
10. In the **Designer panel**, click on the **AWS IoT trigger** in the left menu. The trigger will be added and you'll have to setup the MQTT topic to listen to.
11. In the **Configure triggers** panel, select **Custom IoT rule** for the IoT type.
12. From the Rule dropdown, select **create a new rule**. Also enter a name for your rule.
13. As the Rule query statement, enter: `select * from <ttn_app_name>/devices/<ttn_node_name>/up`
14. Finally, in the code section, copy-paste the code from our [Lambda to SNS notification](lambda-sns.js) code snippet.
15. Finalize the lambda by clicking on the **Save** button at the top.

### 3. Testing the flow
Now, when you move the Things Node, while keeping it concealed from any light, an email should drop in your inbox.

## Store, Query and Dashboard flow

Within this flow we will illustrate how one can store all the received data coming from the ThingsNode to `S3` (both the raw data set and
a flattened json with a subset of the raw data set). We make use of `Kinesis` to compact the incoming data sets and to write
them to `S3`, to modify the incoming raw data sets to a flattened json we will make use of a `Lambda` function. With the AWS `Athena` service
we will create a table for the subset of the raw data on which queries can be performed on the flattened json data sets stored on `S3` and
which will be used via the BI tool `QuickSight`.

### 1. Create AWS Kinesis Firehose Delivery Stream
1. Open the Amazon Kinesis console at https://console.aws.amazon.com/kinesis/  
`Note`: If you don't have a Kinesis configuration yet, choose **Get Started** to continue. <br>
2. Choose **Create delivery stream**
3. **Delivery stream name** => `ttn-kinesis-delivery-stream`
4. **Source** => `Direct PUT or other sources`
5. Choose **Next**
6. **Record transformation** => `Enabled`
7. **Lambda function** choose **Create new**
8. Choose **General Firehose Processing**
9. **Name** => `ttn-lambda-flatten-json`
10. **Role** choose **create new role from template** from the dropdown
11. **Role name** => `ttn-lambda-flatten-json-role`
12. **Policy templates** choose **Basic Edge Lambda permissions** from the dropdown
13. The Lambda function code can't be edited at this moment so choose **Create function**
14. Now your enter the main screen of your Lambda function. At the top you have the **Designer panel**, at the bottom the **Function code** panel.
15. In the **Function code** panel in the code section, copy-paste the code from our [Lambda to flatten json](lambda-flatten-json-kinesis-records.js) code snippet.
16. Choose **Save**
17. Close the browser page of the lambda function
18. Close the **Choose Lambda blueprint** popup box that is still opened in your **Kinesis Firehose** Console
19. **Lambda function** => `ttn-lambda-flatten-json` from the dropdown.
20. Choose **Next**
21. **Select destination**
22. **Destination** => `Amazon S3`
21. **S3 bucket** choose **Create new**
22. **S3 bucket name** => `ttn-workshop-data-<unique-id>` <br>
`Note`: An S3 bucket must be globally unique
23. Choose **Create S3 bucket**
24. Notice that the create S3 bucket is automatically selected as the **S3 bucket**
25. **Prefix** => `iot-to-bi/` <br>
`Note`: It is very important to add an ending **/**
26. **Source record S3 backup** => `Enabled`. This is were you configure that the raw ingested data set should be stored on S3 as well.
27. **Backup S3 bucket** choose **Create new**
28. **S3 bucket name** => `ttn-workshop-raw-data-<unique-id>` <br>
`Note`: An S3 bucket must be globally unique
29. Choose **Create S3 bucket**
30. Notice that the create S3 bucket is automatically selected as the **Backup S3 bucket**
31. **Prefix** => keep the default (don't change)
32. Choose **Next**
33. S3 buffer conditions **Buffer size** => keep the default (don't change)
34. S3 buffer conditions **Buffer interval** => `60` seconds
35. S3 compression and encryption => keep the defaults (don't change)
36. Error logging => keep the default (don't change)
37. **IAM role** => `Create new, or Choose`
38. Choose **Allow** (so keep the suggested IAM Role and Create a new Role Policy)
39. Choose **Next**
40. Choose **Create delivery stream**

### 2. Add Action to IoT Thing to Send Data to Kinesis.

1. Open AWS IoT in the AWS Management Console: https://console.aws.amazon.com/iot/home
2. In the menu on the left, go to **Act**
3. Choose **Create a rule**
4. **Name** => `ttn_iot_to_kinesis`
5. **Attribute** => `*` (So the Rule query statement becomes: SELECT * FROM ")
6. **Topic filter** => `<ttn-app-name>/devices/<ttn-node-name>/up`
7. **Condition** => keep the default (blanc)
8. Choose **Add action**
9. Choose **Send messages to an Amazon Kinesis Firehose stream**
10. Choose **Configure action**
11. **Stream name** => `ttn-kinesis-delivery-stream` from the dropdown
12. **Separator** => `\n(newline)` from the dropdown
13. **IAM role name** => choose **Create a new role**
14. `ttn-iot-to-kinesis-role`
15. Choose **Create a new role**
16. **IAM role name** => choose `ttn-iot-to-kinesis-role` from the dropdown
17. Choose **Add action**
18. Choose **Create rule**

From the moment one should see data being stored on S3. (This is done in batches as configured with
the Kinesis Firehose S3 buffer conditions).

### 3. Athena
1. Open AWS Athena in the AWS Management Console: https://console.aws.amazon.com/athena/
2. Make sure the `default` database is selected in the left column under **DATABASE**
3. In the right panel in the code section, copy-paste the code from our [Athena create table statement](athena-create-table.sql)
4. Adapt the SQL syntax for the **LOCATION** part to select your proper **S3 bucket**
5. Choose **Run Query**
6. In the left panel under **TABLES** a new table should be seen with the name `ttn_iot_flat_data`
7. In the right panel in the code section type
```
select * from ttn_iot_flat_data
```
8. Choose **Run Query**
9. One should see in the **Results** tab a list of records corresponding to the data set on S3

### 4. QuickSight
1. Open AWS QuickSight in the QuickSight Console: https://quicksight.aws.amazon.com/
2. Choose **Sign up for QuickSight** <br>
`Note`: QuickSight is not a default supported service and one has to subscribe for it to activate the service on their
AWS Account.
3. Select the **Standard** Edition and choose **Continue**
4. **QuickSight account name** => `ttn-quick-sight-account`
5. **Notification email address** => `your account owners email address might be preferred`
6. **QuickSight capacity region** => `US East (N.Virgina)` <br>
`Note`: We have noticed that this service is not fully supported in other regions
7. Select **Amazon Athena**
8. Select **Amazon S3**
9. Select **S3 buckets linked to QuickSight account**
10. Select the S3 bucket **ttn-workshop-data-<unique-id>**
11. Choose **Select buckets**
12. Choose **Finish**

Now your QuickSight account is generated. After the creation of the account choose **Go to Amazon QuickSight**

13. Choose **Manage data** on the top right side
14. Choose **New data set** on the top left
15. Choose **Athena**
16. **Data source name** => `ttn-flat-iot-athena-data`
17. Choose **Create data source**
18. **Database: contain sets of tables** => `default` from dropdown
19. **Tables: contain the data you can visualize** => `ttn_iot_flat_data` select the radio button
20. Choose **Select**
21. **Finish data set creation** => select **Directly query your data**
22. Choose **Visualize**

Before one can use the time one has to modify the data type within the QuickSight data set.

23. Select **Visualize** from the most left column
24. **Fields list** select `Edit analysis data sets` from the drop down where **ttn_iot_flat_data** is shown
25. Choose **Edit** for `ttn_iot_flat_data`
26. In the column **time** select the **String** type and select **Date** type
27. Specify the following time format **YYYY-MM-DD HH:mm:ss**
28. Choose Update

Now the data set is ready to be used and one can start creating QuickSight visuals.

