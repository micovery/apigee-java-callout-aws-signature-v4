## AWS Signature V4 in Apigee Java Callout

This repo shows how to call Amazons's REST APIs using Apigee's out of the box Service Callout policy. 
When calling Amazon's REST APIs you have to authenticate each API call with a `key` and a `secret key`.

However, you also need to digitally sign each request using [AWS Signature V4 algorithm](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html). The process of computing the
signature is non trivial. To help with this task, this repo has an Apigee Java Callout that can add
the necessary signature headers.


## How it works

The Java Callout policy takes an existing HTTP request object and adds the following headers:

* x-amz-content-sha256
* x-amz-date
* authorization

The value of the headers is computed dynamically based on the content of the request object as well as the
AWS key and secret key. Behind the scenes it leverages [Amazon's SDK for Java](https://aws.amazon.com/sdk-for-java/) 
to compute these values.


### Pre-built distribution

You can find the pre-built jar file for the Java Callout in the dist/ directory.


### Using in Apigee

Here is a sample flow of the policies you would need to add an entry to an AWS S3 bucket.

First create an Apigee request object using the [Assign Message](https://docs.apigee.com/api-platform/reference/policies/assign-message-policy) policy.


```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AssignMessage async="false" continueOnError="false" enabled="true" name="AM-S3Request">
    <DisplayName>AM-S3Request</DisplayName>
    <Set>
        <Verb>PUT</Verb>
        <Path>newS3ObjectKey</Path>
        <Headers>
            <Header name="content-type">application/octet-stream</Header>
        </Headers>
        <Payload>New S3 value</Payload>
    </Set>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
    <AssignTo createNew="new" transport="http" type="request">s3Callout</AssignTo>
</AssignMessage>
```

In the above example, we have an HTTP `PUT` request object (called **s3Callout**). The request path is to `/newS3ObjectKey`, and payload `"New S3 Value"`.
Those are the S3 Object's key and value respectively. 

Next we need to add the AWS signature headers. So, lets do that using the Java Callout policy:

```xml
<JavaCallout async="false" continueOnError="false" enabled="true" name="JC-AWSSignV4">
    <DisplayName>JC-AWSSignV4</DisplayName>
    <Properties>
        <Property name="debug">true</Property>
        <Property name="service">s3</Property>
        <Property name="endpoint">https://my-bucket-name.s3.amazonaws.com</Property>
        <Property name="region">us-west-1</Property>
        <Property name="key">{private.aws-key}</Property>
        <Property name="secret">{private.aws-secret-key}</Property>
        <Property name="message-variable-ref">s3Callout</Property>
    </Properties>
    <ClassName>com.google.apigee.edgecallouts.AWSSignatureV4Callout</ClassName>
    <ResourceURL>java://edge-callout-aws-signature-v4.jar</ResourceURL>
</JavaCallout>
```

Note, that both the AWS `key` and `secret key` are coming from private flow variables. This is a best practice
so that these values do not show in the Apigee trace. You could populate these values using an [Apigee Key-Value-Map
policy](https://docs.apigee.com/api-platform/reference/policies/key-value-map-operations-policy).

At this point we have the signed HTTP request object. The next step is to actually execute it.
We can do that using Apigee's Service Callout policy:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ServiceCallout async="true" continueOnError="false" enabled="true" name="SC-CallS3">
    <DisplayName>SC-CallS3</DisplayName>
    <Properties/>
    <Request clearPayload="false" variable="s3Callout">
        <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
    </Request>
    <Response>s3CalloutResponse</Response>
    <HTTPTargetConnection>
        <Properties/>
        <URL>ttps://my-bucket-name.s3.amazonaws.com</URL>
    </HTTPTargetConnection>
</ServiceCallout>
```


### Sample Apigee API Proxy

I've included a [Sample Apigee Proxy](https://github.com/micovery/apigee-java-callout-aws-signature-v4/raw/master/downloads/apigee-s3-sample-apiproxy.zip) (in the downloads directory) you can use to quickly try out the Java Callout (This proxy assumes that you have an Apigee KVM named "aws-s3-credentials" with the "key", and "secretKey" entries).

If you are going to be using this across from multiple Apigee proxies, consider creating an [Apigee Shared-Flow](https://docs.apigee.com/api-platform/fundamentals/shared-flows) instead.


### Build Prerequisites


  * [Maven 3.6.1 or later](https://maven.apache.org/download.cgi)
  * [Java SE 9 or later](https://www.oracle.com/technetwork/java/javase/downloads/index.html)
  * bash (Linux shell)
  * cURL
  

### Building it


If you want to build the Java Callout yourself, follow these instructions.

First, we will run the `buildsetup.sh` script to download Apigee's Java Callout libraries:

```bash
$ ./buildsetup.sh
```

This script downloads a couple of JAR files and installs them in maven.

Then, we need to compile and package the actual Java Callout code:

```bash
$ cd callout
$ mvn package
```

Once this is done you will see a new jar file  "edge-callout-aws-signature-v4.jar" within the target directory. 
That is the build output.

The build process itself runs inside docker, so it should work well across different operating
systems a long as you have both bash, and docker installed in your system.


### Not Google Product Clause

This is not an officially supported Google product.
