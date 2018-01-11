![](https://s3-us-west-2.amazonaws.com/us-west-2-aws-training/awsu-spl/sts-sign-in-images/media/aws-logo.png)

# Introduction to AWS Lambda
## Overview

The lab provides a basic explanation of AWS Lambda. It will demonstrate the steps required to get started to create a Lambda function in an event-driven environment.

**AWS Lambda** is a compute service that runs your code in response to events and automatically manages the compute resources for you, making it easy to build applications that respond quickly to new information. AWS Lambda starts running your code within milliseconds of an event such as an image upload, in-app activity, website click, or output from a connected device. You can also use AWS Lambda to create new back-end services where compute resources are automatically triggered based on custom requests.

## Topics covered

By the end of this lab you will be able to:

*   Create an AWS Lambda function
*   Configure an Amazon S3 bucket as a Lambda Event Source
*   Trigger a Lambda function by uploading an object to Amazon S3
*   Monitor AWS Lambda S3 functions through Amazon CloudWatch Log

## Scenario

This lab demonstrates AWS Lambda by creating a serverless **image thumbnail application**.

The following diagram illustrates the application flow:

![Overview 1](https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-88/scripts/overview1.png)

*   (1) A user uploads an object to the source bucket in **Amazon S3** (object-created event).
*   (2) Amazon S3 detects the object-created event.
*   (3) Amazon S3 publishes the object-created event to AWS Lambda by invoking the Lambda function and passing event data as a function parameter.
*   (4) AWS Lambda executes the Lambda function.
*   (5) From the event data it receives, the Lambda function knows the source bucket name and object key name. The Lambda function reads the object and creates a thumbnail using graphics libraries, then saves the thumbnail to the target bucket.

Upon completing this tutorial, you will have the following resources in your account:

![Overview 2](https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-88/scripts/overview2.png)

The steps in this lab will show you how to create the Amazon S3 buckets and the Lambda function. You will then test the service by uploading images for resizing.

* * *

## Task 1: Create the Amazon S3 Buckets

In this task, you will create two Amazon S3 buckets -- one for input and one for output.

1.  In the **AWS Management Console**, on the **Services** menu, click **S3**.

2.  Click **Create bucket**.

Every bucket in Amazon S3 requires a unique name, so you will use a name based on a name and random number of your choosing.

1.  In the **Create bucket** window, configure the following:

*   For **Bucket name**, enter <input readonly="" class="copyable-inline-input" size="13" type="text" value="images-NUMBER">
*   Replace **NUMBER** with a random number. (e.g. _images-123_)
*   Copy the name of your bucket to a text editor. You will use this in the next step and also later in the lab.
*   Click **Create**.

If you receive an error stating **The requested bucket name is not available**, then click the first **Edit** link, change the bucket name and try again until it works.

You will now create another bucket for output.

1.  Click **Create bucket**.

2.  In the **Create bucket** window, configure the following:

*   For **Bucket name**, paste the name of your images bucket.
*   At the end of the bucket name, append <input readonly="" class="copyable-inline-input" size="8" type="text" value="-resized">
    *   (e.g. _images-123-resized_)
*   Click **Create**.

You should now have buckets named similar to:

*   _images-123_
*   _images-123-resized_

You will now upload a picture for testing purposes.

1.  Right-click this link and download the picture to your computer: [HappyFace.jpg](https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-88/scripts/HappyFace.jpg)

Firefox users: Make sure the saved filename is _HappyFace.jpg_ (not _.jpeg)_.

1.  Open the image on your computer. It is a large picture, with dimensions of 1280 x 853.

2.  In the S3 Console, click the **images*** bucket. (Not the *-resized bucket)

3.  Click **Upload**.

4.  In the **Upload** window, click **Add files**.

5.  Browse to and select the **HappyFace.jpg** picture you downloaded.

6.  Click **Upload**.

Later in this lab you will invoke the Lambda function manually by passing sample event data to the function. The sample data will refer to this _HappyFace.jpg_ image.

* * *

## Task 2: Create an AWS Lambda Function

In this task, you will create an AWS Lambda function that reads an image from Amazon S3, resizes the image and then stores the new image in Amazon S3.

1.  On the **Services** menu, click **Lambda**.

2.  Click **Create a function**.

**Blueprints** are code templates for writing Lambda functions. Blueprints are provided for standard Lambda triggers such as creating Alexa skills and processing Amazon Kinesis Firehose streams. This lab provides you with a pre-written Lambda function, so you will **Author from scratch**.

1.  In the **Create function** window, configure the following:

*   **Name**: enter <input readonly="" class="copyable-inline-input" size="16" type="text" value="Create-Thumbnail">
*   **Runtime**: select **Python 3.6**.
*   **Existing role**: select **lambda-execution-role**.

This **role** grants permission to the Lambda function to access Amazon S3 to read and write the images.

*   Click **Create function**.

A page will be displayed with your function configuration.

AWS Lambda functions can be **triggered** automatically by activities such as data being received by Amazon Kinesis or data being updated in an Amazon DynamoDB database. For this lab, you will trigger the Lambda function whenever a new object is created in your Amazon S3 bucket.

1.  Under **Add triggers**, click **S3**.

2.  Scroll down to **Configure triggers** and enter these settings:

*   **Bucket:** select your **images-*** (e.g. _images-123_) bucket.
*   **Event type:** select **Object Created (All)** (_not_ Object Removed)
*   Click **Add**.

1.  Scroll back to the diagram at the top of the page.

2.  Click **Create-Thumbnail** at the top of the diagram:

![](https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-88/scripts/create-thumbnail-button.png)

You will now configure the Lambda function.

1.  Scroll down to the **Function code** section and configure the following settings (and ignore any settings that aren't listed):

*   **Code entry type:** select **Upload a file from Amazon S3**.
*   **Runtime:** select **Python 3.6**.
*   **Handler:** enter <input readonly="" class="copyable-inline-input" size="23" type="text" value="CreateThumbnail.handler">
*   **S3 link URL*** Copy and paste this URL into the field:

    https://s3-us-west-2.amazonaws.com/us-west-2-aws-training/awsu-spl/spl-88/scripts/CreateThumbnail.zip

The _CreateThumbnail.zip_ file contains the following Lambda function:

Do not copy this code -- it is just showing you what is in the Zip file.

    import boto3
    import os
    import sys
    import urllib
    from PIL import Image
    import PIL.Image

    s3_client = boto3.client('s3')

    def resize_image(image_path, resized_path):
        with Image.open(image_path) as image:
            image.thumbnail((128,128))
            image.save(resized_path)

    def handler(event, context):
        for record in event['Records']:
            bucket = record['s3']['bucket']['name']
            key = record['s3']['object']['key']
            raw_key = urllib.parse.unquote_plus(key)
            download_path = '/tmp/{}'.format(key)
            upload_path = '/tmp/resized-{}'.format(key)

            s3_client.download_file(bucket, raw_key, download_path)
            resize_image(download_path, upload_path)
            s3_client.upload_file(upload_path,
                 '{}-resized'.format(bucket),
                 'thumbnail-{}'.format(raw_key),
                 ExtraArgs={'ContentType': 'image/jpeg'})

1.  Examine the above code. It is performing the following steps:

*   Receives an Event, which contains the name of the incoming object (Bucket, Key)
*   Downloads the image to local storage
*   Resizes the image using the _Pillow_ library
*   Uploads the resized image to the _-resized_ bucket

1.  In the **Basic settings** section at the bottom of the page, for **Description:** enter <input readonly="" class="copyable-inline-input" size="30" type="text" value="Create a thumbnail-sized image">

You will leave the other settings as default, but here is a brief explanation of these settings:

*   **Memory** defines the resources that will be allocated to your function. Increasing memory also increases CPU allocated to the function.
*   **Timeout** sets the maximum duration for function execution, up to five minutes.
*   **VPC** (under _Network_) provides the Lambda function access to resources within a Virtual Private Cloud (VPC) network.
*   **Dead Letter Queue (DLQ) Resource** (under _Debugging and error handling_) defines how to handle failed function executions.
*   **Enable active tracing** allows tracing and monitoring of distributed code via AWS X-Ray.

1.  Click **Save** at the top of the window.

Your Lambda function has now been configured.

* * *

## Task 3: Test Your Function

In this task, you will test your Lambda function. This is done by simulating an event with the same information normally sent from Amazon S3 when a new object is uploaded.

1.  Click the **drop-down** arrow to the left of Test.

2.  In the menu, select **Configure test events**.

3.  In the **Configure test event** window, configure the following

*   Select **Create new test event**.
*   **Event template** select **S3 Put**.
*   **Event name** enter <input readonly="" class="copyable-inline-input" size="6" type="text" value="Upload">

A sample template will be displayed that shows the event data sent to a Lambda function when it is triggered by an upload into Amazon S3\. You will need to edit the bucket name so that it uses the bucket you created earlier.

*   Replace **sourcebucket** with the name of your images bucket (e.g. _images-123_) that you copied to your text editor.

![](https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-88/images/before.png)  
![](https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-88/images/after.png)

1.  Click **Create**.

2.  Click **Test**.

AWS Lambda will now trigger your function, using _HappyFace.jpg_ as the input image.

Towards the top of the page you should see the message: _Execution result: succeeded_

If your test did not succeed, the error message will explain the cause of failure. For example, a _Forbidden_ message means that the image was not found possibly due to an incorrect bucket name. Review the previous steps to confirm that you have configured the function correctly.

1.  Click **Details** to expand it.

You will be shown information including:

*   Execution duration
*   Resources consumed
*   Maximum memory used
*   Log output

You can now view the resized image that was stored in Amazon S3.

1.  On the **Services** menu, click **S3**.

2.  Click the name of your **_-resized_** bucket (which is the second bucket you created).

3.  Click **thumbnail-HappyFace.jpg**

4.  Click **Open**. (If the image does not open, disable your pop-up blocker.)

The image should now be a smaller thumbnail of the original image.

You are welcome to upload your own images to the _images-_ bucket and then check for thumbnails in the _-resized_ bucket.

* * *

## Task 4: Monitoring and Logging

You can monitor AWS Lambda functions to identify problems and view log files to assist in debugging.

1.  On the **Services** menu, click **Lambda**.

2.  Click your **Create-Thumbnail** function.

3.  Click the **Monitoring** tab.

The console displays graphs showing:

*   **Invocations:** The number of times the function has been invoked.
*   **Duration:** How long the function took to execute (in milliseconds).
*   **Errors:** How many times the function failed.
*   **Throttles:** When too many functions are invoked simultaneously, they will be throttled. The default is 1000 concurrent executions.
*   **Iterator age:** Measures the age of the last record processed from streaming triggers (Amazon Kinesis and Amazon DynamoDB Streams).
*   **DLQ Errors:** Failures when sending messages to the Dead Letter Queue.

Log messages from Lambda functions are retained in **Amazon CloudWatch Logs**.

1.  Click **Jump to Logs**.

You will be taken to the CloudWatch Management Console.

1.  Click the link in the **Log Streams** column.

Log information is displayed. Click the triangles to view the individual lines.

The Event Data includes the Request Id, the duration (in milliseconds), the billed duration (rounded up to the nearest 100 ms, the Memory Size of the function and the Maximum Memory that the function used. In addition, any logging messages or print statements from the functions are displayed in the logs. This assists in debugging Lambda functions.

* * *

## Conclusion

Congratulations! You have now learned how to:

*   Create an AWS Lambda function
*   Configure an Amazon S3 bucket as a Lambda Event Source
*   Trigger a Lambda function by uploading an object to Amazon S3
*   Monitor AWS Lambda S3 functions through Amazon CloudWatch Log
