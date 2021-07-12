# CCF CUR Collection

These templates are used to collect cost and usage report data from AWS accounts into a central logging account. The reason for this is to collate cost and usage for one or more accounts in order to run Cloud Carbon Footprint while maintaining as light a touch as possible on the monitored account. This avoids having to jump through access/governance hoops and is a near zero risk way to measure carbon usage for production accounts. If you can do it, it would of course be easier to log your AWS Organization management account level cost and usage (for all accounts in the org). That will probably be the way to go in the future, but for now I'm trying this on a couple of accounts, not the whole org.

These notes assume experience of AWS and Cloud Carbon Footprint

# Overview

# Step 1 - Collector Account
Deploy template 1_cloud_carbon_collector_buckets.cf.yaml

The namespace parameter is used to prefix resources created so call it something like
"peteccf" or "snccf". If you have trouble with bucket name collision you might need
to add some randomness here.

This template creates two buckets - one for collecting the CUR data from each 
of the producer accounts and one for storing Athena query results. Both buckets
block public access and the CUR collection bucket has versioning on. This is required
for S3 replication.

Once this completes, go to "Outputs" tab and note the CCFCollectorBucketArn value. This is
used as a parameter on the producer accounts to configure where to replicate the CUR data.

# Step 2 - Producer Account(s)

## Create CUR bucket with replication
Deploy template 2_cloud_carbon_producer.yaml to the accounts you want to track.

You'll need:

* The collector account ID
* The destination bucket ARN (from step 1)

Namespace is used to prefix resource names as in step 1.

This template creates a bucket in the producer account. We need this as CUR data can
only be written to a bucket inside of the account creating the CUR data. It also creates
a role and creates an S3 bucket replication policy. As CUR data is written to this bucket,
it should be replicated to the bucket in the collector account.

Go to "Outputs" tab and note the "CloudCarbonBucketName" which we'll need in a moment. Also
note the "CloudCarbonFootprintReplicationRoleArn" value - we need that in step 3.

## Create CUR report
We have to do this manually. CloudFormation does not (at time of writing :-)) support Cost and
Usage report creation.

AWS documentation is available: https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html but in summary:

In the console, navigate to "Billing"
Select "Cost & Usage Reports" on the left hand side
Select Create report
Name it CCF and tick the "Include resource IDs" box (could be you don't need this, I didn't test it yet)
Leave Data refresh settings "Automatically refresh" ticked
Under S3 bucket, select the bucket you just created - the value of CloudCarbonBucketName (step 2) and confirm the policy is correct (that is, tick the box - you don't need to change it!)
For the prefix put your account ID or some other unique ID
Time granularity is "Daily"
Tick "Enable report data integration for *Amazon Athena*"

Hopefully you'll get the "Report created successfully" message.

It can take up to 24 hours to start to see CUR data being delivered. While you wait move on to step 3.

# Step 3 - Collector Account - Allow replication from producers
Back in the collector account, deploy template 3_cloud_carbon_collector_iam.cf.yaml

You'll need:

* The ARN of the user or role that will be running Cloud Carbon Footprint
* The ARNs of the replication roles from all of the producer accounts
* The name of the stack you created in step 1

Namespace is used as before.

CCF assumes a role to query CUR data. This template creates the role. Here you must provide an ARN for a user or this Athena role to query CUR data. To get this ARN, if you are running CCF as your own user account then go to IAM then users and select your user and copy the ARN displayed. (Or ask your admin if you can't see IAM). Alternately get the ARN of a role you can assume and use that. 

The ARNs for the replication roles need to specified as a comma-separated string:

> "arn:aws:iam::111111111111:role/myorg-ccf-producer-replication-role,arn:aws:iam::222222222222:role/myorg-ccf-producer-replication-role"

all one line, no spaces. I wish parameters supported a nice form for that. If you only have one
producer, you can just put that.

This template adds a resource policy to the collector bucket to enable the producer accounts to replicate into the collection bucket. It also creates the role used by CCF to run.

You can now test replication by uploading an object into the producer bucket (where the CUR reports will go) and verify that it replicates to the collection bucket. Delete the object after it appears (in both buckets) - if doing that in the console turn on viewing versions first to make sure the object is really deleted.

# Wait for the CUR data to arrive

Once all that is set up you need to wait for up to 24 hours before the CUR data starts to arrive.

Once CUR data starts arriving, it should be written into a "folder" (prefix) alongside a CloudFormation template. This template is created by AWS to setup Athena integration. However, this template is written assuming the CUR data lives in the producer account and integrates with the producer account bucket.

To get CCF working in the central collector account, the template needs a few updates - you can see an annotated example in `4_cloud_carbon_collector_ccf_int_example.cf.yaml`.

Update that CF template so that it references the collection bucket (with the appropriate path - collected account ID in the example) and resources in the collection account and you're done.

(I'll write more on this final step integrating CUR/Athena/CCF a bit later).










