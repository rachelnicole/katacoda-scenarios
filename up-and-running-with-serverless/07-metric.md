We can add a custom metric to our Lambda function.

Replace the Lambda function's code with the following:

```
const AWS = require("aws-sdk");
const uuid = require("uuid/v4");

const axios = require("axios");

// Import sendDistributionMetric() from datadog-lambda-js
// This library is provided by the Serverless framework plugin, so we don't have to install it.
const sendDistributionMetric = require("datadog-lambda-js").sendDistributionMetric;

module.exports.handler = async (event, context) => {
  const s3 = new AWS.S3();
  const bucket = process.env.BUCKET_NAME;
  const stage = process.env.STAGE;

  const id = `${uuid()}.jpg`;
  const key = `${stage}/unprocessed/${id}`;

  const unprocessedImageUrl = `http://${bucket}.s3.amazonaws.com/${stage}/unprocessed/${id}`;
  const processedImageUrl = `http://${bucket}.s3.amazonaws.com/${stage}/processed/${id}`;

  const signedUrlExpireSeconds = 60 * 3;

  const options = {
    Bucket: bucket,
    Key: key,
    ACL: "public-read",
    Expires: signedUrlExpireSeconds,
    ContentType: "image/jpeg"
  };

  const url = s3.getSignedUrl("putObject", options);

  const unprocessedImageTinyUrlResponse = await axios.get(
    `https://tinyurl.com/api-create.php?url=${unprocessedImageUrl}`
  );
  const processedImageTinyUrlResponse = await axios.get(
    `https://tinyurl.com/api-create.php?url=${processedImageUrl}`
  );

  // Send the distribution metric here
  sendDistributionMetric(
    "create_image_upload_url.request", // Metric name
    1,                                 // Metric value
    `bucket:${bucket}`,                // Metric tag
  );

  return {
    statusCode: 202,
    body: JSON.stringify({
      uploadUrl: url,
      unprocessedImageUrl: unprocessedImageTinyUrlResponse.data,
      processedImageUrl: processedImageTinyUrlResponse.data
    })
  };
};
```{{copy}}
