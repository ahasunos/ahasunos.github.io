---
layout: post
title:  "AWS Lambda & S3 Interaction"
date:   2022-05-22 10:46:54 +0530
categories: aws
---

## S3 bucket
Amazon Simple Storage Service (Amazon S3) is an object storage service offering industry-leading scalability, data availability, security, and performance.

## Activity
In this activity, we have two S3 buckets and a lambda function deployed with image. Whenever any object is uploaded to the first S3 bucket, lambda function is triggered, then some business operation is performed with the object inside the containerized lambda function and finally the object is uploaded to the second S3 bucket.

For this activity my `app.rb` would look something as below:
```Ruby
require "json"
require "shellwords"
require "aws-sdk-s3"
require "logger"

# Global logger
$logger = Logger.new($stdout)

# Method: object_downloaded?
# Downloads an object from an Amazon Simple Storage Service (Amazon S3) bucket.
# @param s3_client [Aws::S3::Client] An initialized S3 client.
# @param bucket_name [String] The name of the bucket containing the object.
# @param object_key [String] The name of the object to download.
# @param local_path [String] The path on your local computer to download the object
# @return [Boolean] true if the object was downloaded; otherwise, false.

def object_downloaded?(s3_client, src_bucket_name, object_key, local_path)
  $logger.info("Downloading object from S3 bucket initiated.")
  s3_client.get_object(
    response_target: local_path,
    bucket: src_bucket_name,
    key: object_key
  )
  $logger.info("Object '#{object_key}' in bucket '#{src_bucket_name}' downloaded to '#{local_path}'.")
  true
rescue StandardError => e
  $logger.info("Error getting object: #{e.message}")
end

# ============================================================================ #
# Method: object_upload
# Uploads an object to Amazon S3 bucket.
# @param s3_client [Aws::S3::Client] An initialized S3 client.
# @param destination_bucket_name [String] The name of the destination bucket.
# @param object_key [String] The name of the object to upload.
# @param local_path [String] The path on your local computer where the file is present.

def object_upload(s3_client, dest_bucket_name, object_key, local_path)
  # upload file from disk in a single request, may not exceed 5GB
  File.open(local_path, 'rb') do |file|
    s3_client.put_object(bucket: dest_bucket_name, key: object_key, body: file)
  end
  $logger.info("#{object_key} uploaded to #{dest_bucket_name}")
rescue StandardError => e
  $logger.info("Error uploading object: #{e.message}")
end

# ============================================================================ #
# Method: lambda_handler
# Driver function of the Lambda
# @param event: the data sent during the Lambda function invocation. It contains information about the event.
# @param context: it contains information about the invocation, function configuration, environment, request id, log etc.
def lambda_handler(event:, context:)
  # Extract the record from the event
  $logger.info("Extract bucket name and object key from event initiated.")
  first_record = event["Records"].first
  src_bucket_name = first_record["s3"]["bucket"]["name"]
  object_key = first_record["s3"]["object"]["key"]
  $logger.info("Bucket: #{src_bucket_name}, object: #{object_key} is extracted from event successfully.")

  # Set local path, destination bucket, region and initialize s3 client.
  $logger.info("Setting of path, destination bucket, region and s3 client initiated.")
  local_path = "/tmp/#{object_key}"
  region = "us-east-2"
  s3_client = Aws::S3::Client.new(region: region)
  dest_bucket_name = "my-destination-bucket"
  $logger.info("Setting of path, destination bucket, region and s3 client completed.")

  # Download the object and perform operation if success.
  if object_downloaded?(s3_client, src_bucket_name, object_key, local_path)

    # Here we perform our business operation and then upload to new S3 bucket
    object_upload(s3_client, dest_bucket_name, object_key, local_path)
  else
    $logger.info("Object '#{object_key}' in bucket '#{src_bucket_name}' not downloaded.")
  end

  {
    statusCode: 200,
    body: JSON.generate("Lambda is invoked successfully!")
  }
end
```

And my Gemfile would be something like:
```Ruby
source "https://rubygems.org"
gem 'aws-sdk-s3', '~> 1'
```


## References
1. [AWS Developer guide for Ruby](https://docs.amazonaws.cn/en_us/sdk-for-ruby/v3/developer-guide/welcome.html)