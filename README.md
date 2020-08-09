# Alexa-Skill-for-Voice-Powered-Analytics

### Introduction
This Alexa skill queries metrics from a data lake. The aim of building this skill is to understand how to uncover Key Performance Indicators (KPIs) from a data set, build and automate queries for measuring those KPIs, and access them via Alexa voice-enabled devices. Startups can make available voice powered analytics to query at any time, and enterprises can deliver these types of solutions to stakeholders so they can have easy access to the Business KPIs that are top of mind.
The main steps of this project include:

* Constructing the Data Lake
* Data Discovery using QuickSight
* Building Data Lake analytics in Athena (based on objects in S3) to generate answers for Alexa
* Building a custom Alexa skill to access the analytics queries from Athena

### Step 0: Lab Setup CloudFormation Stack
I followed the following steps to launch the CloudFormation Stack on the AWS cloud
![CloudFormation Stack] (https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/vpa-cloudformation-launch.gif)

