# Alexa-Skill-for-Voice-Powered-Analytics

### Introduction
This Alexa skill queries metrics from a data lake. The aim of building this skill is to understand how to uncover Key Performance Indicators (KPIs) from a data set, build and automate queries for measuring those KPIs, and access them via Alexa voice-enabled devices. Startups can make available voice powered analytics to query at any time, and enterprises can deliver these types of solutions to stakeholders so they can have easy access to the Business KPIs that are top of mind.
The main steps of this project include:

* Constructing the Data Lake
* Data Discovery using QuickSight
* Building Data Lake analytics in Athena (based on objects in S3) to generate answers for Alexa
* Building a custom Alexa skill to access the analytics queries from Athena

### Step 0: Lab Setup CloudFormation Stack
I followed the following steps to launch the CloudFormation Stack on the AWS Cloud
![](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/vpa-cloudformation-launch.gif)

### Step 1: Constructing the Data Lake
### Step 1 a: Generate Twitter Keys
1.  Go to http://twitter.com/oauth_clients/new
2.  Apply for a Twitter Developer Account. Takes ~15 minutes. Requires detailed justification, twitter approval, and email verification
3. Under Name, enter something descriptive, e.g., awstwitterdatalake *can only have alpha-numerics*
4. Enter a description
5. Under Website,  enter the website of your choice
6. Leave Callback URL blank
7. Read and agree to the Twitter Developer Agreement
8. Click "Create your Twitter application"

### Step 1 b: Deploy App In Repo

16.  Navigate to [Twitter-Poller to-Kinesis-Firehose](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:381943442500:applications~Twitter-Poller-to-Kinesis-Firehose) in the Serverless Application Repository.
17.  Click the Deploy button (top righthand corner)
18.  Login to your AWS account.  After doing so, scroll down to the Application Settings section where the folloing four values need to be entered:
 i.) The 4 Tokens received from Twitter
 ii.) You can keep the Kinesis Firehose resource name the same or change it to a preferred name 
 iii.) Customize the search text that twitter will bring back
19.  After deploying the Serverless Application, the application will begin polling automatically within the next 5 minutes.
</details>

Lastly, note the name of the S3 bucket because it would be required to create the Athena schema.  
