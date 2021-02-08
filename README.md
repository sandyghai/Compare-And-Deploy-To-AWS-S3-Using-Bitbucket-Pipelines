Compare and Deploy To AWS S3 Using BitBucket Pipeline
===========================================
Deploying to AWS S3 is simple just execute a one line command and it will sync all files from directory to S3 bucket e.g. aws s3 sync . s3://${S3_BUCKET}/

This works fine until one day we have to automate the process and start deploying multiple times in a day. It starts becoming difficult with thousands of files pushing to S3 which create lot of put requests which requires time and cost money at the same time.

Background
==========
Our frontend application code releases which comprise of mostly static files was an easy deployment and working fine until thousands of files are pushed. We realised libraries, images, css are unnecessary getting pushed with every deployment when only fewer files are getting changed which takes longer time in deployment.

We need to fix this by following compare and deploy approach for our releases. 

Problem
=======
How do we compare files and run aws sync using BitBucket Pipeline?

Prerequisites:
=============
Before we start creating pipeline for deployment we need to make sure that we have sufficient permissions to copy files on S3 bucket.

* Set up an AWS S3 bucket where deployment artifacts will be copied.
* An IAM configured with sufficient permissions to upload artifacts to the AWS S3 bucket.

Solution
========
- Setup your CI/CD configuration using Bitbucket Pipeline (bitbucket-pipelines.yml) file. 
- Use Python3 image
- Install awscli in pipeline
- Setup AWS credentials as environment variables

  AWS_ACCESS_KEY_ID (*): Your AWS access key.
  AWS_SECRET_ACCESS_KEY (*): Your AWS secret access key. Make sure that you save it as a secured variable.
  AWS_DEFAULT_REGION (*):  The AWS region code (us-east-1, us-west-2, etc.) of the region containing the AWS resource(s). 

- Create shell script pipe.sh to perform git diff and deploy to aws bucket.
- Run CloudFront invalidation. 

* Create bitbucket-pipelines.yml file as below.

<pre>
image: python:3

pipelines:
  branches:
    master:
      - step:
          script:
              - pip install awscli
              - export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
              - export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
              - export AWS_DEFAULT_REGION=us-east-1
              - chmod +x pipe.sh
              - ./pipe.sh [Replace-With-Your-S3-Bucket]
              - aws cloudfront create-invalidation --distribution-id [Replace-With-Your-CloudFront-Distribution-Id] --paths '/*'
</pre>

* Create the shell script pipe.sh file to be run from bitbucket pipeline.

<pre>

#!/bin/bash

S3_BUCKET=$1

# We will get the files based on last commit which will be avaialble as in env variable
echo "Last commit: $BITBUCKET_COMMIT"

# We will get diff based on last commit to add into AWS CLI sync command
ChangedFiles=()
for i in $( git diff-tree --no-commit-id --name-only -r $BITBUCKET_COMMIT ); do
    if [ "$i" != "bitbucket-pipelines.yml" -a "$i" != "pipe.sh" ]; then
      ChangedFiles+=( "--include=$i" )
    fi
done

echo "${ChangedFiles[@]}"

# --exclude * will exclude all files
# --delete will remove files that exist in the destination but not in the source
# Input ChangedFiles to xargs to to run aws sync command
echo "${ChangedFiles[@]}" | xargs aws s3 sync . s3://${S3_BUCKET}/ --delete --exclude "*"

</pre>

Discussion
==========
This will help to setup compare and deploy files on S3 using Bitbucket Pipelines.  

Have you implemented this solution?

Yes, I implemented the solution and extend it by adding tag based deployment.

References
==========
Helpful resources from Bitbucket Pipelines documentations

[Deploy to AWS with S3](https://support.atlassian.com/bitbucket-cloud/docs/deploy-to-aws-with-s3/)

[Write a pipe for Bitbucket Pipelines](https://support.atlassian.com/bitbucket-cloud/docs/write-a-pipe-for-bitbucket-pipelines/)
