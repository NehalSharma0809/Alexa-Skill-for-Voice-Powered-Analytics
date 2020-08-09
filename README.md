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
#### Step 1 a: Generate Twitter Keys
1.  Go to http://twitter.com/oauth_clients/new
2.  Apply for a Twitter Developer Account. Takes ~15 minutes. Requires detailed justification, twitter approval, and email verification
3. Under Name, enter something descriptive, e.g., awstwitterdatalake *can only have alpha-numerics*
4. Enter a description
5. Under Website,  enter the website of your choice
6. Leave Callback URL blank
7. Read and agree to the Twitter Developer Agreement
8. Click "Create your Twitter application"

#### Step 1 b: Deploy App In Repo

1.  Navigate to [Twitter-Poller to-Kinesis-Firehose](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:381943442500:applications~Twitter-Poller-to-Kinesis-Firehose) in the Serverless Application Repository.
2.  Click the Deploy button (top righthand corner)
3.  Login to your AWS account.  After doing so, scroll down to the Application Settings section where the folloing four values need to be entered:
 i.) The 4 Tokens received from Twitter
 ii.) You can keep the Kinesis Firehose resource name the same or change it to a preferred name 
 iii.) Customize the search text that twitter will bring back
4.  After deploying the Serverless Application, the application will begin polling automatically within the next 5 minutes.

Lastly, note the name of the S3 bucket because it would be required to create the Athena schema.  

### Step 2: AWS QuickSight for Data Visualization
In this step, we will use QuickSight to explore our dataset and visualize a few interesting metrics of the twitter dataset. 

#### Step 2 a: Understand The Raw Data 

This step is intended to give you a better understanding of the data we are using for the lab. 
Each file in S3 has a collection of JSON objects stored within the file.
In addition, the files have been gziped by [Kinesis FIrehose](https://aws.amazon.com/kinesis/firehose/) which saves cost and improves performance.

I have used a dataset created from Twitter data related to AWS re:Invent 2020. 
This dataset includes tweets with the #reinvent hashtag or to/from @awsreinvent. 

#### Getting the data into S3
The data is acquired starting with a CloudWatch Event triggering an AWS Lambda function every 5 minutes. Lambda is making calls to Twitter's APIs for data, and then ingesting the data into [Kinesis Firehose](https://aws.amazon.com/kinesis/firehose/).   
Firehose then micro-batches the results into S3 as shown in the following diagram:
![Workshop dataset](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/Athena_Arch_1.png)

This dataset has been acquired from the following bucket:
EU-WEST-1 | ```s3://aws-vpa-tweets-euw1/```


Amazon Kinesis Firehose delivers the data into S3 as a GZIP file format.
A variety of methods can be used to download one of the files in the dataset. If you use the AWS CLI today, this is likely the easiest method to take a look at the data.

List one of the files with (Note use **s3://aws-vpa-tweets-euw1...** for Ireland):
```bash
aws s3 ls s3://aws-vpa-tweets/tweets/sample/2017/11/06/04/aws-vpa-tweets-sample.gz
```
Download this file to your local directory (Note use **s3://aws-vpa-tweets-euw1...** for Ireland):
```bash
aws s3 cp s3://aws-vpa-tweets/tweets/sample/2017/11/06/04/aws-vpa-tweets-sample.gz .
```

Since the files are compressed, you will need to unzip it. In addition the data is stored in raw text form. Make sure you rename the file to either *.json or *.text.
The data format look like this:
```json
{  
    "id": 914642318044655600,  
    "text": "Two months until #reInvent! Check out the session calendar & prepare for reserved seating on Oct. 19! http://amzn.to/2fxlVg7  ",  
    "created": "2017-10-02 00:04:56",  
    "screen_name": " AWSReinvent ",  
    "screen_name_followers_count": 49864,  
    "place": "none",  
    "country": "none",  
    "retweet_count": 7,  
    "favorite_count": 21
}
```

#### Step 2b - Create an Athena table

We need to create a table in Amazon Athena. This will allow us to query the data at rest in S3 from QuickSight. 
The twitter data is stored as JSON documents and then compressed in s3. 
Athena supports reading of gzip files and includes json SerDe's to make parsing the data easy.

**Create Athena table**

i.) We need to ensure that we are in the **same region** that launched the CloudFormation stack 
ii.) For the **Ireland region**, modify the location field below with the following location:
LOCATION
  's3://aws-vpa-tweets-euw1/tweets/'
iii.) In the AWS account navigate to the **Athena** service
iv.) In the top left menu, choose *Query Editor*
v.) Use this code to create the Athena table. Once added, click **Run Query**

```SQL
CREATE EXTERNAL TABLE IF NOT EXISTS default.tweets(
  id bigint COMMENT 'Tweet ID', 
  text string COMMENT 'Tweet text', 
  created timestamp COMMENT 'Tweet create timestamp', 
  screen_name string COMMENT 'Tweet screen_name',
  screen_name_followers_count int COMMENT 'Tweet screen_name follower count',
  place string COMMENT 'Location full name',
  country string COMMENT 'Location country',
  retweet_count int COMMENT 'Retweet count', 
  favorite_count int COMMENT 'Favorite count')
ROW FORMAT SERDE 
  'org.openx.data.jsonserde.JsonSerDe' 
WITH SERDEPROPERTIES ( 
  'paths'='id,text,created,screen_name,screen_name_followers_count,place_fullname,country,retweet_count,favorite_count') 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://aws-vpa-tweets/tweets/'
```

**Verify the table created correctly** 
i.) We will see an Athena table called *tweets* in the *default* database 
ii.) If we click on the *tweets* table, we can see the fields that we saw earlier.    
iii.) Let's test that the tweets table works.  In the same Query Editor run the following `SELECT` statement (clear the previous statement):

```SQL
SELECT COUNT(*) AS TOTAL_TWEETS FROM tweets;
```


#### Step 2 c :Explore the data using Quicksight
We've created an Athena table directly on top of our S3 Twitter data, let's explore some insights on the data.  
While this can be achieved through Athena itself or compatible query engines, Amazon Quicksight enables us to connect directly to Athena and quickly visualize it into charts and graphs without writing any SQL code.  
Let's explore:      

**Import Permissions note** 
1. Launch the [QuickSight portal](https://us-east-1.quicksight.aws.amazon.com/).  This may ask you to register your email address for Quicksight access.  
1. If not already configured, Quicksight may need special permissions to access Athena:   
a. (These settings can only be changed in the **N.Virginia region**) In the upper right corner, ensure **US East N. Virginia** is selected, then to the right of the *region* in the upper right corner, choose the profile name, and from the dropdown menu, choose *Manage Quicksight*.    
b. On the left menu, click *Account Settings*  
c. Click the *Edit AWS permissions* button  
d. Ensure the box *Amazon Athena* is checked.  
e. Click *Choose S3 Buckets*, **Choose Select All**.   
f. Click the Tab *S3 Buckets you can access across AWS*, under *Use Different Bucket*, Type: ```aws-vpa-tweets``` (Note For Ireland: ```aws-vpa-tweets-euw1```) Then click **Add S3 Bucket**, then click **Select Buckets**, then click **Apply**    

![Setting Quicksight Permissions](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/Quicksight_Permissions.gif)
 

1. **(if running out of Ireland)** In the main Quicksight portal page, switch back to the **EU Ireland Region**)
2. In the upper right choose **Manage data**
3. Now in the upper left choose **New data set**
4. We see tiles for each of the QuickSight supported data sources. From this page select the **Athena** tile. 
5. When asked for the dataset name we can choose anything you like, for our example I used **tweets-dataset**.We can choose to validate that SSL will be used for data between Athena and QuickSight. Finish by selecting **Create data source**
6. Now we need to choose the Athena table we created in **Step 1**. For our example we used the **Default** database, with a table name of **tweets**. Finish by clicking on **Select**. 
7. SPICE is not needed for this project. If asked, select **Directly query your data**. Click Visualize when done. 
8. QuickSight will now import the data. Wait until we see **Import Complete**. Then close the summary window. 
9. Add the **created** field from the Athena table by dragging it from the Field list to the empty AutoGraph window.
10. From the *Visual types* in the bottom left corner, select **Vertical bar chart**
11. Add another Visual by selecting in the top left corner, the **+ Add** button and then **Add visual**
12. On this new graph, lets add the **country** field. 
13. As we can see, lots of tweets do not include which country the tweet was created in. Let’s filter these results out. Click on the large bar labeled **none**, then select **exclude "none"** from the pop up window. Now we can see the tweets without a location were excluded.
14. Lets change the visual from a bar chart to a pie chart. Select the entire visual, then from the bottom right select the **pie chart** visual.  Add **Group By: "country"**

** Interesting insights from the data in QuickSight**

![](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/VPA_Quicksight_Submission_1.jpg) 
 
![](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/VPA_Quicksight_Submission_2.jpg) 
 
![](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/VPA_Quicksight_Submission_3.jpg)

