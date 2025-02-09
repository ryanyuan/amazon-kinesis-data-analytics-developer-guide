# Example: Writing to Kinesis Data Firehose<a name="get-started-exercise-fh"></a>

In this exercise, you create a Kinesis Data Analytics application that has a Kinesis data stream as a source and a Kinesis Data Firehose delivery stream as a sink\. Using the sink, you can verify the output of the application in an Amazon S3 bucket\.

**Note**  
To set up required prerequisites for this exercise, first complete the [Getting Started \(DataStream API\)](getting-started.md) exercise\.

**Topics**
+ [Create Dependent Resources](#get-started-exercise-fh-1)
+ [Write Sample Records to the Input Stream](#get-started-exercise-fh-2)
+ [Download and Examine the Apache Flink Streaming Java Code](#get-started-exercise-fh-5)
+ [Compile the Application Code](#get-started-exercise-fh-5.5)
+ [Upload the Apache Flink Streaming Java Code](#get-started-exercise-fh-6)
+ [Create and Run the Kinesis Data Analytics Application](#get-started-exercise-fh-7)
+ [Clean Up AWS Resources](#getting-started-fh-cleanup)

## Create Dependent Resources<a name="get-started-exercise-fh-1"></a>

Before you create an Amazon Kinesis Data Analytics for Apache Flink for this exercise, you create the following dependent resources:
+ A Kinesis data stream \(`ExampleInputStream`\) 
+ A Kinesis Data Firehose delivery stream that the application writes output to \(`ExampleDeliveryStream`\)\. 
+ An Amazon S3 bucket to store the application's code \(`ka-app-code-<username>`\)

You can create the Kinesis stream, Amazon S3 buckets, and Kinesis Data Firehose delivery stream using the console\. For instructions for creating these resources, see the following topics:
+ [Creating and Updating Data Streams](https://docs.aws.amazon.com/kinesis/latest/dev/amazon-kinesis-streams.html) in the *Amazon Kinesis Data Streams Developer Guide*\. Name your data stream **ExampleInputStream**\.
+ [Creating an Amazon Kinesis Data Firehose Delivery Stream](https://docs.aws.amazon.com/firehose/latest/dev/basic-create.html) in the *Amazon Kinesis Data Firehose Developer Guide*\. Name your delivery stream **ExampleDeliveryStream**\. When you create the Kinesis Data Firehose delivery stream, also create the delivery stream's **S3 destination** and **IAM role**\. 
+ [How Do I Create an S3 Bucket?](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html) in the *Amazon Simple Storage Service User Guide*\. Give the Amazon S3 bucket a globally unique name by appending your login name, such as **ka\-app\-code\-*<username>***\.

## Write Sample Records to the Input Stream<a name="get-started-exercise-fh-2"></a>

In this section, you use a Python script to write sample records to the stream for the application to process\.

**Note**  
This section requires the [AWS SDK for Python \(Boto\)](https://aws.amazon.com/developers/getting-started/python/)\.

1. Create a file named `stock.py` with the following contents:

   ```
    import datetime
       import json
       import random
       import boto3
   
       STREAM_NAME = "ExampleInputStream"
   
   
       def get_data():
           return {
               'event_time': datetime.datetime.now().isoformat(),
               'ticker': random.choice(['AAPL', 'AMZN', 'MSFT', 'INTC', 'TBV']),
               'price': round(random.random() * 100, 2)}
   
   
       def generate(stream_name, kinesis_client):
           while True:
               data = get_data()
               print(data)
               kinesis_client.put_record(
                   StreamName=stream_name,
                   Data=json.dumps(data),
                   PartitionKey="partitionkey")
   
   
       if __name__ == '__main__':
           generate(STREAM_NAME, boto3.client('kinesis', region_name='us-west-2'))
   ```

1. Run the `stock.py` script: 

   ```
   $ python stock.py
   ```

   Keep the script running while completing the rest of the tutorial\.

## Download and Examine the Apache Flink Streaming Java Code<a name="get-started-exercise-fh-5"></a>

The Java application code for this example is available from GitHub\. To download the application code, do the following:

1. Clone the remote repository with the following command:

   ```
   git clone https://github.com/aws-samples/amazon-kinesis-data-analytics-java-examples
   ```

1. Navigate to the `amazon-kinesis-data-analytics-java-examples/FirehoseSink` directory\.

The application code is located in the `FirehoseSinkStreamingJob.java` file\. Note the following about the application code:
+ The application uses a Kinesis source to read from the source stream\. The following snippet creates the Kinesis source:

  ```
  return env.addSource(new FlinkKinesisConsumer<>(inputStreamName,
                  new SimpleStringSchema(), inputProperties));
  ```
+ The application uses a Kinesis Data Firehose sink to write data to a delivery stream\. The following snippet creates the Kinesis Data Firehose sink:

  ```
  private static KinesisFirehoseSink<String> createFirehoseSinkFromStaticConfig() {
          Properties sinkProperties = new Properties();
          sinkProperties.setProperty(AWS_REGION, region);
  
          return KinesisFirehoseSink.<String>builder()
                  .setFirehoseClientProperties(sinkProperties)
                  .setSerializationSchema(new SimpleStringSchema())
                  .setDeliveryStreamName(outputDeliveryStreamName)
                  .build();
      }
  ```

## Compile the Application Code<a name="get-started-exercise-fh-5.5"></a>

To compile the application, do the following:

1. Install Java and Maven if you haven't already\. For more information, see [Prerequisites](getting-started.md#setting-up-prerequisites) in the [Getting Started \(DataStream API\)](getting-started.md) tutorial\.

1. **In order to use the Kinesis connector for the following application, you need to download, build, and install Apache Maven\. For more information, see [Using the Apache Flink Kinesis Streams Connector with previous Apache Flink versions](earlier.md#how-creating-apps-building-kinesis)\.**

1. Compile the application with the following command: 

   ```
   mvn package -Dflink.version=1.15.3
   ```
**Note**  
The provided source code relies on libraries from Java 11\. 

Compiling the application creates the application JAR file \(`target/aws-kinesis-analytics-java-apps-1.0.jar`\)\.

## Upload the Apache Flink Streaming Java Code<a name="get-started-exercise-fh-6"></a>

In this section, you upload your application code to the Amazon S3 bucket that you created in the [Create Dependent Resources](#get-started-exercise-fh-1) section\.

**To upload the application code**

1. Open the Amazon S3 console at [https://console\.aws\.amazon\.com/s3/](https://console.aws.amazon.com/s3/)\.

1. In the console, choose the **ka\-app\-code\-*<username>*** bucket, and then choose **Upload**\.

1. In the **Select files** step, choose **Add files**\. Navigate to the `java-getting-started-1.0.jar` file that you created in the previous step\. 

1. You don't need to change any of the settings for the object, so choose **Upload**\.

Your application code is now stored in an Amazon S3 bucket where your application can access it\.

## Create and Run the Kinesis Data Analytics Application<a name="get-started-exercise-fh-7"></a>

You can create and run a Kinesis Data Analytics application using either the console or the AWS CLI\.

**Note**  
When you create the application using the console, your AWS Identity and Access Management \(IAM\) and Amazon CloudWatch Logs resources are created for you\. When you create the application using the AWS CLI, you create these resources separately\.

**Topics**
+ [Create and Run the Application \(Console\)](#get-started-exercise-fh-7-console)
+ [Create and Run the Application \(AWS CLI\)](#get-started-exercise-fh-7-cli)

### Create and Run the Application \(Console\)<a name="get-started-exercise-fh-7-console"></a>

Follow these steps to create, configure, update, and run the application using the console\.

#### Create the Application<a name="get-started-exercise-fh-7-console-create"></a>

1. Open the Kinesis Data Analytics console at [ https://console\.aws\.amazon\.com/kinesisanalytics](https://console.aws.amazon.com/kinesisanalytics)\.

1. On the Kinesis Data Analytics dashboard, choose **Create analytics application**\.

1. On the **Kinesis Analytics \- Create application** page, provide the application details as follows:
   + For **Application name**, enter **MyApplication**\.
   + For **Description**, enter **My java test app**\.
   + For **Runtime**, choose **Apache Flink**\.
**Note**  
Kinesis Data Analytics for Apache Flink uses Apache Flink version 1\.15\.2\.
   + Leave the version pulldown as **Apache Flink version 1\.15\.2 \(Recommended version\)**\.

1. For **Access permissions**, choose **Create / update IAM role `kinesis-analytics-MyApplication-us-west-2`**\.

1. Choose **Create application**\.

**Note**  
When you create the application using the console, you have the option of having an IAM role and policy created for your application\. The application uses this role and policy to access its dependent resources\. These IAM resources are named using your application name and Region as follows:  
Policy: `kinesis-analytics-service-MyApplication-us-west-2`
Role: `kinesis-analytics-MyApplication-us-west-2`

#### Edit the IAM Policy<a name="get-started-exercise-fh-7-console-iam"></a>

Edit the IAM policy to add permissions to access the Kinesis data stream and Kinesis Data Firehose delivery stream\.

1. Open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

1. Choose **Policies**\. Choose the **`kinesis-analytics-service-MyApplication-us-west-2`** policy that the console created for you in the previous section\. 

1. On the **Summary** page, choose **Edit policy**\. Choose the **JSON** tab\.

1. Add the highlighted section of the following policy example to the policy\. Replace all the instances of the sample account IDs \(*012345678901*\) with your account ID\.

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "ReadCode",
               "Effect": "Allow",
               "Action": [
                   "s3:GetObject",
                   "s3:GetObjectVersion"
               ],
               "Resource": [
                   "arn:aws:s3:::ka-app-code-username/java-getting-started-1.0.jar"
               ]
           },
           {
               "Sid": "DescribeLogGroups",
               "Effect": "Allow",
               "Action": [
                   "logs:DescribeLogGroups"
               ],
               "Resource": [
                   "arn:aws:logs:us-west-2:012345678901:log-group:*"
               ]
           },
           {
               "Sid": "DescribeLogStreams",
               "Effect": "Allow",
               "Action": [
                   "logs:DescribeLogStreams"
              ],
               "Resource": [
                   "arn:aws:logs:us-west-2:012345678901:log-group:/aws/kinesis-analytics/MyApplication:log-stream:*"
               ]
           },
           {
               "Sid": "PutLogEvents",
               "Effect": "Allow",
               "Action": [
                   "logs:PutLogEvents"
               ],
               "Resource": [
                   "arn:aws:logs:us-west-2:012345678901:log-group:/aws/kinesis-analytics/MyApplication:log-stream:kinesis-analytics-log-stream"
               ]
           },
           {
               "Sid": "ReadInputStream",
               "Effect": "Allow",
               "Action": "kinesis:*",
               "Resource": "arn:aws:kinesis:us-west-2:012345678901:stream/ExampleInputStream"
           },
           {
               "Sid": "WriteDeliveryStream",
               "Effect": "Allow",
               "Action": "firehose:*",
               "Resource": "arn:aws:firehose:us-west-2:012345678901:deliverystream/ExampleDeliveryStream"
           }
       ]
   }
   ```

#### Configure the Application<a name="get-started-exercise-fh-7-console-configure"></a>

1. On the **MyApplication** page, choose **Configure**\.

1. On the **Configure application** page, provide the **Code location**:
   + For **Amazon S3 bucket**, enter **ka\-app\-code\-*<username>***\.
   + For **Path to Amazon S3 object**, enter **java\-getting\-started\-1\.0\.jar**\.

1. Under **Access to application resources**, for **Access permissions**, choose **Create / update IAM role `kinesis-analytics-MyApplication-us-west-2`**\.

1. Under **Monitoring**, ensure that the **Monitoring metrics level** is set to **Application**\.

1. For **CloudWatch logging**, select the **Enable** check box\.

1. Choose **Update**\.

**Note**  
When you choose to enable CloudWatch logging, Kinesis Data Analytics creates a log group and log stream for you\. The names of these resources are as follows:   
Log group: `/aws/kinesis-analytics/MyApplication`
Log stream: `kinesis-analytics-log-stream`

#### Run the Application<a name="get-started-exercise-fh-7-console-run"></a>

The Flink job graph can be viewed by running the application, opening the Apache Flink dashboard, and choosing the desired Flink job\.

#### Stop the Application<a name="get-started-exercise-fh-7-console-stop"></a>

On the **MyApplication** page, choose **Stop**\. Confirm the action\.

#### Update the Application<a name="get-started-exercise-fh-7-console-update"></a>

Using the console, you can update application settings such as application properties, monitoring settings, and the location or file name of the application JAR\. 

On the **MyApplication** page, choose **Configure**\. Update the application settings and choose **Update**\.

**Note**  
To update the application's code on the console, you must either change the object name of the JAR, use a different S3 bucket, or use the AWS CLI as described in the [Update the Application Code](#get-started-exercise-fh-7-cli-update-code) section\. If the file name or the bucket does not change, the application code is not reloaded when you choose **Update** on the **Configure** page\.

### Create and Run the Application \(AWS CLI\)<a name="get-started-exercise-fh-7-cli"></a>

In this section, you use the AWS CLI to create and run the Kinesis Data Analytics application\.

#### Create a Permissions Policy<a name="get-started-exercise-fh-7-cli-policy"></a>

First, you create a permissions policy with two statements: one that grants permissions for the `read` action on the source stream, and another that grants permissions for `write` actions on the sink stream\. You then attach the policy to an IAM role \(which you create in the next section\)\. Thus, when Kinesis Data Analytics assumes the role, the service has the necessary permissions to read from the source stream and write to the sink stream\.

Use the following code to create the `KAReadSourceStreamWriteSinkStream` permissions policy\. Replace *username* with the user name that you will use to create the Amazon S3 bucket to store the application code\. Replace the account ID in the Amazon Resource Names \(ARNs\) \(`012345678901`\) with your account ID\.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": ["arn:aws:s3:::ka-app-code-username",
                "arn:aws:s3:::ka-app-code-username/*"
            ]
        },
        {
            "Sid": "ReadInputStream",
            "Effect": "Allow",
            "Action": "kinesis:*",
            "Resource": "arn:aws:kinesis:us-west-2:012345678901:stream/ExampleInputStream"
        },
        {
            "Sid": "WriteDeliveryStream",
            "Effect": "Allow",
            "Action": "firehose:*",
            "Resource": "arn:aws:firehose:us-west-2:012345678901:deliverystream/ExampleDeliveryStream"
        }
    ]
}
```

For step\-by\-step instructions to create a permissions policy, see [Tutorial: Create and Attach Your First Customer Managed Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_managed-policies.html#part-two-create-policy) in the *IAM User Guide*\.

**Note**  
To access other Amazon services, you can use the AWS SDK for Java\. Kinesis Data Analytics automatically sets the credentials required by the SDK to those of the service execution IAM role that is associated with your application\. No additional steps are needed\.

#### Create an IAM Role<a name="get-started-exercise-fh-7-cli-role"></a>

In this section, you create an IAM role that the Kinesis Data Analytics application can assume to read a source stream and write to the sink stream\.

Kinesis Data Analytics cannot access your stream if it doesn't have permissions\. You grant these permissions via an IAM role\. Each IAM role has two policies attached\. The trust policy grants Kinesis Data Analytics permission to assume the role\. The permissions policy determines what Kinesis Data Analytics can do after assuming the role\.

You attach the permissions policy that you created in the preceding section to this role\.

**To create an IAM role**

1. Open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

1. In the navigation pane, choose **Roles**, **Create Role**\.

1. Under **Select type of trusted identity**, choose **AWS Service**\. Under **Choose the service that will use this role**, choose **Kinesis**\. Under **Select your use case**, choose **Kinesis Analytics**\.

   Choose **Next: Permissions**\.

1. On the **Attach permissions policies** page, choose **Next: Review**\. You attach permissions policies after you create the role\.

1. On the **Create role** page, enter **KA\-stream\-rw\-role** for the **Role name**\. Choose **Create role**\.

   Now you have created a new IAM role called `KA-stream-rw-role`\. Next, you update the trust and permissions policies for the role\.

1. Attach the permissions policy to the role\.
**Note**  
For this exercise, Kinesis Data Analytics assumes this role for both reading data from a Kinesis data stream \(source\) and writing output to another Kinesis data stream\. So you attach the policy that you created in the previous step, [Create a Permissions Policy](#get-started-exercise-fh-7-cli-policy)\.

   1. On the **Summary** page, choose the **Permissions** tab\.

   1. Choose **Attach Policies**\.

   1. In the search box, enter **KAReadSourceStreamWriteSinkStream** \(the policy that you created in the previous section\)\.

   1. Choose the **KAReadSourceStreamWriteSinkStream** policy, and choose **Attach policy**\.

You now have created the service execution role that your application will use to access resources\. Make a note of the ARN of the new role\.

For step\-by\-step instructions for creating a role, see [Creating an IAM Role \(Console\)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html#roles-creatingrole-user-console) in the *IAM User Guide*\.

#### Create the Kinesis Data Analytics Application<a name="get-started-exercise-fh-7-cli-create"></a>

1. Save the following JSON code to a file named `create_request.json`\. Replace the sample role ARN with the ARN for the role that you created previously\. Replace the bucket ARN suffix with the suffix that you chose in the [Create Dependent Resources](#get-started-exercise-fh-1) section \(`ka-app-code-<username>`\.\) Replace the sample account ID \(*012345678901*\) in the service execution role with your account ID\.

   ```
   {
       "ApplicationName": "test",
       "ApplicationDescription": "my java test app",
       "RuntimeEnvironment": "FLINK-1_15",
       "ServiceExecutionRole": "arn:aws:iam::012345678901:role/KA-stream-rw-role",
       "ApplicationConfiguration": {
           "ApplicationCodeConfiguration": {
               "CodeContent": {
                   "S3ContentLocation": {
                       "BucketARN": "arn:aws:s3:::ka-app-code-username",
                       "FileKey": "java-getting-started-1.0.jar"
                   }
               },
               "CodeContentType": "ZIPFILE"
           }
         }
       }
   }
   ```

1. Execute the [https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_CreateApplication.html](https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_CreateApplication.html) action with the preceding request to create the application: 

   ```
   aws kinesisanalyticsv2 create-application --cli-input-json file://create_request.json
   ```

The application is now created\. You start the application in the next step\.

#### Start the Application<a name="get-started-exercise-fh-7-cli-start"></a>

In this section, you use the [https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StartApplication.html](https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StartApplication.html) action to start the application\.

**To start the application**

1. Save the following JSON code to a file named `start_request.json`\.

   ```
   {
       "ApplicationName": "test",
       "RunConfiguration": {
           "ApplicationRestoreConfiguration": { 
            "ApplicationRestoreType": "RESTORE_FROM_LATEST_SNAPSHOT"
            }
       }
   }
   ```

1. Execute the [https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StartApplication.html](https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StartApplication.html) action with the preceding request to start the application:

   ```
   aws kinesisanalyticsv2 start-application --cli-input-json file://start_request.json
   ```

The application is now running\. You can check the Kinesis Data Analytics metrics on the Amazon CloudWatch console to verify that the application is working\.

#### Stop the Application<a name="get-started-exercise-fh-7-cli-stop"></a>

In this section, you use the [https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StopApplication.html](https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StopApplication.html) action to stop the application\.

**To stop the application**

1. Save the following JSON code to a file named `stop_request.json`\.

   ```
   {
       "ApplicationName": "test"
   }
   ```

1. Execute the [https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StopApplication.html](https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_StopApplication.html) action with the following request to stop the application:

   ```
   aws kinesisanalyticsv2 stop-application --cli-input-json file://stop_request.json
   ```

The application is now stopped\.

#### Add a CloudWatch Logging Option<a name="get-started-exercise-fh-7-cli-cw"></a>

You can use the AWS CLI to add an Amazon CloudWatch log stream to your application\. For information about using CloudWatch Logs with your application, see [Setting Up Application Logging](cloudwatch-logs.md)\.

#### Update the Application Code<a name="get-started-exercise-fh-7-cli-update-code"></a>

When you need to update your application code with a new version of your code package, you use the [https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_UpdateApplication.html](https://docs.aws.amazon.com/kinesisanalytics/latest/apiv2/API_UpdateApplication.html) AWS CLI action\.

To use the AWS CLI, delete your previous code package from your Amazon S3 bucket, upload the new version, and call `UpdateApplication`, specifying the same Amazon S3 bucket and object name\.

The following sample request for the `UpdateApplication` action reloads the application code and restarts the application\. Update the `CurrentApplicationVersionId` to the current application version\. You can check the current application version using the `ListApplications` or `DescribeApplication` actions\. Update the bucket name suffix \(<*username*>\) with the suffix you chose in the [Create Dependent Resources](#get-started-exercise-fh-1) section\.

```
{
    "ApplicationName": "test",
    "CurrentApplicationVersionId": 1,
    "ApplicationConfigurationUpdate": {
        "ApplicationCodeConfigurationUpdate": {
            "CodeContentUpdate": {
                "S3ContentLocationUpdate": {
                    "BucketARNUpdate": "arn:aws:s3:::ka-app-code-username",
                    "FileKeyUpdate": "java-getting-started-1.0.jar"
                }
            }
        }
    }
}
```

## Clean Up AWS Resources<a name="getting-started-fh-cleanup"></a>

This section includes procedures for cleaning up AWS resources created in the Getting Started tutorial\.

**Topics**
+ [Delete Your Kinesis Data Analytics Application](#getting-started-fh-cleanup-app)
+ [Delete Your Kinesis Data Stream](#getting-started-fh-cleanup-stream)
+ [Delete Your Kinesis Data Firehose Delivery Stream](#getting-started-fh-cleanup-fh)
+ [Delete Your Amazon S3 Object and Bucket](#getting-started-fh-cleanup-s3)
+ [Delete Your IAM Resources](#getting-started-fh-cleanup-iam)
+ [Delete Your CloudWatch Resources](#getting-started-fh-cleanup-cw)

### Delete Your Kinesis Data Analytics Application<a name="getting-started-fh-cleanup-app"></a>

1. Open the Kinesis Data Analytics console at [ https://console\.aws\.amazon\.com/kinesisanalytics](https://console.aws.amazon.com/kinesisanalytics)\.

1. In the Kinesis Data Analytics panel, choose **MyApplication**\.

1. Choose **Configure**\.

1. In the **Snapshots** section, choose **Disable** and then choose **Update**\.

1. In the application's page, choose **Delete** and then confirm the deletion\.

### Delete Your Kinesis Data Stream<a name="getting-started-fh-cleanup-stream"></a>

1. Open the Kinesis console at [https://console\.aws\.amazon\.com/kinesis](https://console.aws.amazon.com/kinesis)\.

1. In the Kinesis Data Streams panel, choose **ExampleInputStream**\.

1. In the **ExampleInputStream** page, choose **Delete Kinesis Stream** and then confirm the deletion\.

### Delete Your Kinesis Data Firehose Delivery Stream<a name="getting-started-fh-cleanup-fh"></a>

1. Open the Kinesis console at [https://console\.aws\.amazon\.com/kinesis](https://console.aws.amazon.com/kinesis)\.

1. In the Kinesis Data Firehose panel, choose **ExampleDeliveryStream**\.

1. In the **ExampleDeliveryStream** page, choose **Delete delivery stream** and then confirm the deletion\.

### Delete Your Amazon S3 Object and Bucket<a name="getting-started-fh-cleanup-s3"></a>

1. Open the Amazon S3 console at [https://console\.aws\.amazon\.com/s3/](https://console.aws.amazon.com/s3/)\.

1. Choose the **ka\-app\-code\-*<username>* bucket\.**

1. Choose **Delete** and then enter the bucket name to confirm deletion\.

1. If you created an Amazon S3 bucket for your Kinesis Data Firehose delivery stream's destination, delete that bucket too\.

### Delete Your IAM Resources<a name="getting-started-fh-cleanup-iam"></a>

1. Open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

1. In the navigation bar, choose **Policies**\.

1. In the filter control, enter **kinesis**\.

1. Choose the **kinesis\-analytics\-service\-MyApplication\-*<your\-region>*** policy\.

1. Choose **Policy Actions** and then choose **Delete**\.

1. If you created a new policy for your Kinesis Data Firehose delivery stream, delete that policy too\.

1. In the navigation bar, choose **Roles**\.

1. Choose the **kinesis\-analytics\-MyApplication\-*<your\-region>*** role\.

1. Choose **Delete role** and then confirm the deletion\.

1. If you created a new role for your Kinesis Data Firehose delivery stream, delete that role too\.

### Delete Your CloudWatch Resources<a name="getting-started-fh-cleanup-cw"></a>

1. Open the CloudWatch console at [https://console\.aws\.amazon\.com/cloudwatch/](https://console.aws.amazon.com/cloudwatch/)\.

1. In the navigation bar, choose **Logs**\.

1. Choose the **/aws/kinesis\-analytics/MyApplication** log group\.

1. Choose **Delete Log Group** and then confirm the deletion\.