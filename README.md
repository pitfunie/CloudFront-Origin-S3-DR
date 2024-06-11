# CloudFront-Origin-S3-DR

# AWS Account ID
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r .Account)


# Bucket Name - Primary Region
VIRGINIA_BUCKET=${ACCOUNT_ID}-mini-lab-cf-virginia


# Bucket Name - Secondary Region
CALIFORNIA_BUCKET=${ACCOUNT_ID}-mini-lab-cf-california 

# Create two buckets on Amazon S3 to be the sources
# Create two S3 buckets in the primary and secondary region, in this case US East (N. Virginia) and US West (N. California).
aws s3 mb s3://${VIRGINIA_BUCKET} --region us-east-1
aws s3 mb s3://${CALIFORNIA_BUCKET} --region us-west-1

# Create the files to send to buckets
echo "File in N. Virginia S3" > virginia.index.html
echo "File in N. California S3" > california.index.html

# Copy files to the bucket
aws s3 cp virginia.index.html s3://${VIRGINIA_BUCKET}/index.html
aws s3 cp california.index.html s3://${CALIFORNIA_BUCKET}/index.html

# Create a Origin Access Identity on Amazon CloudFront
OAI_ID=$(aws cloudfront create-cloud-front-origin-access-identity \
--cloud-front-origin-access-identity-config \
CallerReference="cloudfront-mini-lab-example",Comment="CloudFront Origin Group Example" \
| jq -r '.CloudFrontOriginAccessIdentity.Id')


# Create Access Policies on Buckets to Allow Amazon CloudFront Access PRIMARY REGION
$ Create bucket access policies in the primary region, in this case US East (N. Virginia).
cat <<EOT > virginia-s3-policy.json
{
"Version": "2008-10-17",
"Id": "PolicyForCloudFrontPrivateContent",
"Statement": [
{
"Sid": "1",
"Effect": "Allow",
"Principal": {
"AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${OAI_ID}"
},
"Action": "s3:GetObject","Resource": "arn:aws:s3:::${VIRGINIA_BUCKET}/*"
"Resource": "arn:aws:s3:::${VIRGINIA_BUCKET}/*"
]
}
EOT


# Create bucket access policies in the secondary region, in this case US West (N. California)
cat <<EOT > california-s3-policy.json
{
"Version": "2008-10-17",
"Id": "PolicyForCloudFrontPrivateContent",
"Statement": [
{
"Sid": "1",
"Effect": "Allow",
"Principal": {
"AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${OAI_ID}"
},
"Action": "s3:GetObject",
"Resource": "arn:aws:s3:::${CALIFORNIA_BUCKET}/*"
}
]
}
EOT




# Apply policies to both buckets.
# Apply policy to grant access to cloudfront distribution
aws s3api put-bucket-policy \
  --bucket ${VIRGINIA_LAB_BUCKET} --policy file://virginia-s3-policy.json

aws s3api put-bucket-policy \
  --bucket ${CALIFORNIA_LAB_BUCKET} --policy file://california-s3-policy.json


# Create a JSON Script for the Amazon CloudFront Distribution
# Create the distribution configuration file, this configuration will create both sources 
# (one for each bucket), and the source group with both sources, cache will be disables for testing only.

cat <<EOT > cf-distribution.json
{
  "CallerReference":"mini-lab-example",
  "Aliases":{
    "Quantity":0
  },
  "DefaultRootObject":"index.html",
  "Origins":{
    "Quantity":2,
    "Items":[
      {
        "Id":"mini-lab-origin-virginia",
        "DomainName":"${VIRGINIA_LAB_BUCKET}.s3.amazonaws.com",
        "S3OriginConfig":{
          "OriginAccessIdentity":"origin-access-identity/cloudfront/${OAI_ID}"
        },
        "ConnectionAttempts":1,
        "ConnectionTimeout":2
      },
      {
        "Id":"mini-lab-origin-california",
        "DomainName":"${CALIFORNIA_LAB_BUCKET}.s3.us-west-1.amazonaws.com",
        "S3OriginConfig":{
          "OriginAccessIdentity":"origin-access-identity/cloudfront/${OAI_ID}"
        },
        "ConnectionAttempts":1,
        "ConnectionTimeout":2
      }
    ]
  },
  "OriginGroups":{
    "Quantity":1,
    "Items":[
      {
        "Id":"mini-lab-origin-group-example",
        "FailoverCriteria":{
          "StatusCodes":{
            "Quantity":6,
            "Items":[
              404, # not found
              403, # forbidden
              500, # internal server error
              502, # bad gateway
              503, # service unavailable
              504  # gateway timeout
            ]
          }
        },
        "Members":{
          "Quantity":2,
          "Items":[
            {
              "OriginId":"mini-lab-origin-virginia"
            },
            {
              "OriginId":"mini-lab-origin-california"
            }
          ]
        }
      }
    ]
  },
  "DefaultCacheBehavior":{
    "TargetOriginId":"mini-lab-origin-group-example",
    "ForwardedValues":{
      "QueryString":false,
      "Cookies":{
        "Forward":"none"
      },
      "Headers":{
        "Quantity":0
      },
      "QueryStringCacheKeys":{
        "Quantity":0
      }
    },
    "TrustedSigners":{
      "Enabled":false,
      "Quantity":0
    },
    "TrustedKeyGroups":{
      "Enabled":false,
      "Quantity":0
    },
    "ViewerProtocolPolicy":"allow-all",
    "MinTTL":0,
    "MaxTTL":0,
    "DefaultTTL":0,
    "AllowedMethods":{
      "Quantity":2,
      "Items":[
        "HEAD",
        "GET"
      ],
      "CachedMethods":{
        "Quantity":2,
        "Items":[
          "HEAD",
          "GET"
        ]
      }
    }
  },
  "CacheBehaviors":{
    "Quantity":0
  },
  "Comment":"",
  "Enabled":true
}
EOT




# Apply policy to grant access to cloudfront distribution
aws s3api put-bucket-policy \
--bucket ${VIRGINIA_BUCKET} --policy file://virginia-s3-policy.json

aws s3api put-bucket-policy \
--bucket ${CALIFORNIA_BUCKET} --policy file://california-s3-policy.json


# Create the Amazon CloudFront Distribution
# Create the distribution configuration file, this configuration will already create 
# both sources (one for each bucket), and the source group with both sources, also 
# disables the cache for easy testing.

# ---> Execute CloudFront-Distribution.json


# Build the Distribution on Amazon CloudFront.

CLOUDFRONT_ID=$(aws cloudfront create-distribution \
--distribution-config file://cf-distribution.json | jq -r '.Distribution.Id')


# Check the distribution status with the command below.

STATUS=InProgress
while [ "$STATUS" == "InProgress" ];
do
STATUS=$(aws cloudfront get-distribution \
--id ${CLOUDFRONT_ID} | jq -r '.Distribution.Status')
echo "Status: "$STATUS
sleep 3
done

# Try CloudFront Distribution

CLOUDFRONT_URL=$(aws cloudfront get-distribution --id ${CLOUDFRONT_ID} | jq -r '.Distribution.DomainName')
echo ${CLOUDFRONT_URL} 

# Go to the URL to see the delivery of the primary region content. Optionally, you can type the Cloudfront URL on your web browser.

curl -s ${CLOUDFRONT_URL} # "File in Virginia S3"

# Test the Origin Group Failover
aws s3 rm s3://${VIRGINIA_BUCKET}/index.html


# Reaccess the URL of the distribution created on Amazon CloudFront to see delivery from the secondary region.
curl -s ${CLOUDFRONT_URL} # "File in California S3"

# copy the file back to the primary region but with another name
aws s3 cp virginia.index.html s3://${VIRGINIA_BUCKET}/virginia.html

# Access the file with the new name to see the return of the primary region
curl -s ${CLOUDFRONT_URL}/virginia.html # "File in Virginia S3"

