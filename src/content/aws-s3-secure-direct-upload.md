---
layout: post
title: Direct uploads to AWS S3 from the browser (crazy performance boost)
subtitle: "Using pre signed URLs upload files securely to a S3 bucket"
author: santypk4
date: "2019-03-01T15:45:00.000Z"
image: img/fast-uploads.jpg
tags: ["AWS S3", "AWS", "DevOps", "Javascript", "Node.js", "Best"]
twittertags: ["aws", "s3", "devops", "nodejs", "javascript"]
draft: false
---

![Direct upload to s3 infrastructure](/img/s3-upload-guide/s3_post.png)

# Why you will want to upload files to a private S3 directly from the browser?

Well if your application is uploading a file to your server, and then your server uploads it to an AWS S3 Bucket, you have a bottleneck and performance trouble.

My clients were uploading **large** video **files**, 100mb average, from various locations Asia, Europe, and North America, my server is hosted on **Heroku** and located in **Northern Virginia** **but my main S3 Bucket** is in **Ireland**!

Will be easier and efficient if the web client has the possibility to upload directly to that AWS S3 Bucket.

![Object upload replication mess](img/s3-upload-guide/connections-mess.png)

Seem's trivial but you may confront several problems and the official AWS documentation don't tell you much.

# The procedure

You will need to generate `pre-signed` AWS S3 URLs, so a user can write an object **directly** with a **POST** or **PUT** call.

## Presigned URL (HTTP PUT Method)

A `pre-signed` URL is a URL that you generate with your AWS credentials and you provide to your users to grant temporary access to a specific AWS S3 object.

The presigned URLs are useful if you want your user/customer to be able to upload a specific object to your bucket, but you don't require them to have AWS security credentials or permissions.

When you create a presigned URL, you must provide your security credentials and then specify a bucket name, an object key, an HTTP method (PUT for uploading objects), and an expiration date and time.

The presigned URLs are valid only for the specified duration.

## Alternative (HTTP POST Form Method)

AWS S3 supports POST, which allows your users to upload content directly to AWS S3. 

POST is designed to simplify uploads, reduce upload latency, and save you money on applications where users upload data to store in AWS S3.


# Generate Credentials

- Open you AWS Console and Navigate to IAM

![AWS console looking for IAM service](img/s3-upload-guide/iam-1.png)

- Create a User with Programmatic access

![Creating AWS user with programatic access](img/s3-upload-guide/iam-2.png) 

- Click the attach existing policies directly tab

![Choosing attach existing policy](img/s3-upload-guide/iam-3.png)

- Click create your own policy and copy the following

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:Put*"
            ],
            "Resource": [
                "arn:aws:s3:::your-bucket-name/*",
            ]
        }
    ]
}
```
![Choosing attach existing policy](img/s3-upload-guide/iam-4.png)

- Click Review Policy and enter a name for the policy.

- Save the policy.

- Add it to your new user.

# Configuring S3 CORS policy

The `same-origin` policy is an important security concept implemented by web browsers to prevent Javascript code from making requests against a different domain than the one from which it was served.

Cross-Origin Resource Sharing (CORS) is a technique for relaxing the same-origin policy, allowing Javascript on a web page to making HTTP calls to a different origin.

CORS makes it easier for service providers to distribute content to users while adding interoperability to online services​​.
​​
- Go to your bucket

- Go to the permissions tab

- Click CORS configuration and copy and paste the following


```xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
<CORSRule>
    <AllowedOrigin>*</AllowedOrigin>
    <AllowedMethod>HEAD</AllowedMethod>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>POST</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
</CORSRule>
</CORSConfiguration>
```

CORS makes it easy for web services to quickly and easily integrate without exposing their users.

# Activating Transfer Acceleration Endpoint

AWS S3 Transfer Acceleration is a bucket-level feature that enables faster data transfers to and from AWS S3.

- Go to your bucket

- Choose properties

  ![Properties tab](img/s3-upload-guide/transfer-activate-1.png)

- Click on permissions
  ![Transfer acceleration](img/s3-upload-guide/transfer-activate-2.png)

- Scroll to transfer acceleration and active it
  ![Activate transfer acceleration](img/s3-upload-guide/transfer-activate-3.png)

# Server Code - PUT to a transfer acceleration endpoint

You have two choices for generating the pre-signed URL, depending on how your client code will upload the file.

This approach generates a PUT endpoint but you **can-not** use multi-part FormData to upload files. But you can benefit from using AWS Transfer acceleration endpoint
We rely on the `getSignedUrl` method from AWS-SDK. 

[Read more about it on the AWS S3 SDK](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrl-property)

```javascript
const AWS = require('aws-sdk');
const express = require('express');
const route = express.Router();

route.get('/signed-url-put-object', async (req, res) => {
  AWS.config.update({
    accessKeyId: 'AAAAAAAAAAAAAAAA', // Generated on step 1
    secretAccessKey: 'J21//xxxxxxxxxxx', // Generated on step 1
    region: 'eu-west-1', // Must be the same as your bucket
    signatureVersion: 'v4',
  });
  const params = {
    Bucket: 'your-bucket-name',
    Key: 'my-awesome-object.webm',
    Expires: 30 * 60, // 30 minutes
    ContentType: 'video/webm'
  };
  const options = {
    signatureVersion: 'v4',
    region: 'eu-west-1', // same as your bucket
    // highlight-next-line
    endpoint: new AWS.Endpoint('your-bucket-name.s3-accelerate.amazonaws.com'),
    // highlight-next-line
    useAccelerateEndpoint: true,
  }
  const client = new AWS.S3(options);
  const signedURL = await (new Promise((resolve, reject) => {
    // highlight-next-line
    client.getSignedUrl('putObject', params, (err, data) => {
      if (err) {
        reject(err)
      } else {
        resolve(data)
      }
      });
  }));
  return res.json({
    signedURL,
  })
}
```

# Server Code - POST Multi-Part FormData

Get a pre-signed POST policy to support uploading to S3 directly from an HTML form from the browser.

With this, you will generate a **FORM** and you must send all the fields in a FormData object in a **POST** request to the AWS S3 bucket.

**You can not use the transfer acceleration endpoint because is a CloudFront endpoint that it's not configured with the necessary CORS options and you cannot change it sadly.**

But this is useful if you are developing a react native application and you have the needed of using a **FormData** or any other scenario where you must use multi-part uploads.

For this method we rely on the `createPresignedPost` method from AWS-SDK please note the difference with the previous method.

[Read more about it on the AWS S3 SDK](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#createPresignedPost-property)

*You cannot use transfer acceleration with this method*

```javascript
const AWS = require('aws-sdk');
const express = require('express');
const route = express.Router();

route.get('/signed-form-upload', async (req, res) => {
  AWS.config.update({
    accessKeyId: 'AAAAAAAAAAAAAAAA', // Generated on step 1
    secretAccessKey: 'J21//xxxxxxxxxxx', // Generated on step 1
    region: 'eu-west-1', // Must be the same as your bucket
    signatureVersion: 'v4',
  });
  const params = {
    Bucket: 'your-bucket-name',
    Key: 'my-awesome-object.webm',
    Fields: {
      Key: 'my-awesome-object.webm',
    },
  };
  const options = {
    signatureVersion: 'v4',
    region: 'eu-west-1', // same as your bucket

    // highlight-next-line
    endpoint = new AWS.Endpoint('https://your-bucket-name.s3.amazonaws.com'),

    // highlight-next-line
    useAccelerateEndpoint = false,

    // highlight-next-line
    s3ForcePathStyle = true,
  }

  const client = new AWS.S3(options);
  const form = await (new Promise((resolve, reject) => {

    // highlight-next-line
    client.createPresignedPost(params, (err, data) => {

      if (err) {
        reject(err)
      } else {
        resolve(data)
      }
    });
  }));
  return res.json({
    form: { ...form, url: config.aws.s3.AWS_S3_ENDPOINT } 
  })
}
```

# Common Problems

**"SignatureDoesNotMatch"**

```xml

<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>SignatureDoesNotMatch</Code>
<Message>The request signature we calculated does not match the signature you provided. Check your key and signing method.</Message>
<StringToSignBytes>90 81 89 12 ...</StringToSignBytes>
<RequestId>G7AAF1689RC5909C</RequestId>
<HostId>q+r+2T5K6mMKLKTWw0R9/jm33LyIfZFACY8GEDznfmMrRxvaVJwPiu/hlofuJWbW</HostId>
<StringToSign>PUT
    video/webm
    456789067
    x-amz-acl:authenticated-read
    /your-bucket-name/</StringToSign>
<AWSAccessKeyId>youraccesskey</AWSAccessKeyId>
</Error>

```

S3 creates a signature by combining file type, the file key, content-type, and so on.

If you are having this problem check: 

- Make sure you are passing the correct Content Type Header.

- Check that you are using POST with the form upload method, or PUT with the transfer acceleration endpoint.

- The file type and file key **MUST** exactly match the one which was provided when the pre-signed URL was created.

- When using POST FormData method, check that you are sending all the form fields that were generated by AWS S3 SDK.


# Conclusion

There are several ways to upload files to a private AWS S3 bucket directly from browser, and can be challenging and confusing, but with a little effort, you will have a huge improvement in your performance.

In my case the performance upgrade was about 200% thanks to the AWS S3 Transfer Acceleration endpoint.

![AWS S3 Transfe Acceleration Performance](img/s3-upload-guide/s3_ta_speed_compare_2.png)

_You can try this awesome performance estimator tool [right here](http://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html)_


# Resources

- https://www.digitalocean.com/community/questions/signed-put-url-for-nodejs

- https://sanderknape.com/2017/08/using-pre-signed-urls-upload-file-private-s3-bucket/

- https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-UsingHTTPPOST.html

- https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html

- https://docs.aws.amazon.com/AmazonS3/latest/user-guide/enable-transfer-acceleration.html

- https://aws.amazon.com/blogs/aws/aws-storage-update-amazon-s3-transfer-acceleration-larger-snowballs-in-more-regions/