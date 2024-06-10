# Disable distribution using the update distribution configuration feature.


CLOUDFRONT_DIST=$(aws cloudfront get-distribution-config --id ${CLOUDFRONT_ID}) # Cloudfront Distribution info

CLOUDFRONT_ETAG=$(echo $CLOUDFRONT_DIST | jq -r '.ETag') # Cloudfront ETag

echo $CLOUDFRONT_DIST | jq -r '.DistributionConfig' | jq -r .Enabled="false" > mini-lab-example-disabled.json # Change Enabled parameter to "false"

CLOUDFRONT_ETAG=$(aws cloudfront update-distribution \
--id ${CLOUDFRONT_ID} \
--if-match ${CLOUDFRONT_ETAG} \
--distribution-config file://mini-lab-example-disabled.json \
| jq -r .ETag) 


# Wait a few minutes until the distribution is removed. You can check the deletion already finished using the command below:

STATUS=InProgress
while [ "$STATUS" == "InProgress" ];
do
STATUS=$(aws cloudfront get-distribution \
--id ${CLOUDFRONT_ID} | jq -r '.Distribution.Status')
echo "Status: "$STATUS
sleep 3
done

# Delete the Origin Access Identity

OAI_ETAG=$(aws cloudfront get-cloud-front-origin-access-identity --id ${OAI_ID} | jq -r '.ETag')

aws cloudfront delete-cloud-front-origin-access-identity \
--id ${OAI_ID} --if-match ${OAI_ETAG}

D# elete buckets used in the lab in both regions.

aws s3 rb s3://${VIRGINIA_LAB_BUCKET} --force
aws s3 rb s3://${CALIFORNIA_LAB_BUCKET} --force


# Delete the files created 

rm virginia.index.html \
california.index.html \
virginia-s3-policy.json \
california-s3-policy.json \
cf-distribution.json \
mini-lab-example-disabled.json



