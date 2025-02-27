# Actions for AWS IoT Rule
<!--BEGIN STABILITY BANNER-->

---

![cdk-constructs: Experimental](https://img.shields.io/badge/cdk--constructs-experimental-important.svg?style=for-the-badge)

> The APIs of higher level constructs in this module are experimental and under active development.
> They are subject to non-backward compatible changes or removal in any future version. These are
> not subject to the [Semantic Versioning](https://semver.org/) model and breaking changes will be
> announced in the release notes. This means that while you may use them, you may need to update
> your source code when upgrading to a newer version of this package.

---

<!--END STABILITY BANNER-->

This library contains integration classes to send data to any number of
supported AWS Services. Instances of these classes should be passed to
`TopicRule` defined in `@aws-cdk/aws-iot`.

Currently supported are:

- Invoke a Lambda function
- Put objects to a S3 bucket
- Put logs to CloudWatch Logs
- Capture CloudWatch metrics
- Put records to Kinesis Data Firehose stream

## Invoke a Lambda function

The code snippet below creates an AWS IoT Rule that invoke a Lambda function
when it is triggered.

```ts
import * as iot from '@aws-cdk/aws-iot';
import * as actions from '@aws-cdk/aws-iot-actions';
import * as lambda from '@aws-cdk/aws-lambda';

const func = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_14_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = (event) => {
      console.log("It is test for lambda action of AWS IoT Rule.", event);
    };`
  ),
});

new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323("SELECT topic(2) as device_id, timestamp() as timestamp, temperature FROM 'device/+/data'"),
  actions: [new actions.LambdaFunctionAction(func)],
});
```

## Put objects to a S3 bucket

The code snippet below creates an AWS IoT Rule that put objects to a S3 bucket
when it is triggered.

```ts
import * as iot from '@aws-cdk/aws-iot';
import * as actions from '@aws-cdk/aws-iot-actions';
import * as s3 from '@aws-cdk/aws-s3';

const bucket = new s3.Bucket(this, 'MyBucket');

new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323("SELECT topic(2) as device_id FROM 'device/+/data'"),
  actions: [new actions.S3PutObjectAction(bucket)],
});
```

The property `key` of `S3PutObjectAction` is given the value `${topic()}/${timestamp()}` by default. This `${topic()}`
and `${timestamp()}` is called Substitution templates. For more information see 
[this documentation](https://docs.aws.amazon.com/iot/latest/developerguide/iot-substitution-templates.html).
In above sample, `${topic()}` is replaced by a given MQTT topic as `device/001/data`. And `${timestamp()}` is replaced
by the number of the current timestamp in milliseconds as `1636289461203`. So if the MQTT broker receives an MQTT topic
`device/001/data` on `2021-11-07T00:00:00.000Z`, the S3 bucket object will be put to `device/001/data/1636243200000`. 

You can also set specific `key` as following:

```ts
new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323(
    "SELECT topic(2) as device_id, year, month, day FROM 'device/+/data'",
  ),
  actions: [
    new actions.S3PutObjectAction(bucket, {
      key: '${year}/${month}/${day}/${topic(2)}',
    }),
  ],
});
```

If you wanna set access control to the S3 bucket object, you can specify `accessControl` as following:

```ts
new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323("SELECT * FROM 'device/+/data'"),
  actions: [
    new actions.S3PutObjectAction(bucket, {
      accessControl: s3.BucketAccessControl.PUBLIC_READ,
    }),
  ],
});
```

## Put logs to CloudWatch Logs

The code snippet below creates an AWS IoT Rule that put logs to CloudWatch Logs
when it is triggered.

```ts
import * as iot from '@aws-cdk/aws-iot';
import * as actions from '@aws-cdk/aws-iot-actions';
import * as logs from '@aws-cdk/aws-logs';

const logGroup = new logs.LogGroup(this, 'MyLogGroup');

new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323("SELECT topic(2) as device_id FROM 'device/+/data'"),
  actions: [new actions.CloudWatchLogsAction(logGroup)],
});
```

## Capture CloudWatch metrics

The code snippet below creates an AWS IoT Rule that capture CloudWatch metrics
when it is triggered.

```ts
import * as iot from '@aws-cdk/aws-iot';
import * as actions from '@aws-cdk/aws-iot-actions';

const topicRule = new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323(
    "SELECT topic(2) as device_id, namespace, unit, value, timestamp FROM 'device/+/data'",
  ),
  actions: [
    new actions.CloudWatchPutMetricAction({
      metricName: '${topic(2)}',
      metricNamespace: '${namespace}',
      metricUnit: '${unit}',
      metricValue: '${value}',
      metricTimestamp: '${timestamp}',
    }),
  ],
});
```

## Put records to Kinesis Data Firehose stream

The code snippet below creates an AWS IoT Rule that put records to Put records
to Kinesis Data Firehose stream when it is triggered.

```ts
import * as iot from '@aws-cdk/aws-iot';
import * as actions from '@aws-cdk/aws-iot-actions';
import * as s3 from '@aws-cdk/aws-s3';
import * as firehose from '@aws-cdk/aws-kinesisfirehose';
import * as destinations from '@aws-cdk/aws-kinesisfirehose-destinations';

const bucket = new s3.Bucket(this, 'MyBucket');
const stream = new firehose.DeliveryStream(this, 'MyStream', {
  destinations: [new destinations.S3Bucket(bucket)],
});

const topicRule = new iot.TopicRule(this, 'TopicRule', {
  sql: iot.IotSql.fromStringAsVer20160323("SELECT * FROM 'device/+/data'"),
  actions: [
    new actions.FirehoseStreamAction(stream, {
      batchMode: true,
      recordSeparator: actions.FirehoseStreamRecordSeparator.NEWLINE,
    }),
  ],
});
```
