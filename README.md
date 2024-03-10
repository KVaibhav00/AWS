# Airline Incremental Data Ingestion
ARCHITECTURE
 

 

 

Open image-20240310-061906.png
image-20240310-061906.png
 

AWS  services Used
S3 - Object Storage service in AWS.

Cloudtrail Data Events - To capture any changes in the S3 bucket 

AWS eventbridge - Based on a event trigger a service (for example lambda function or step function)

AWS Glue - Managed ETL service on AWS

Crawler - To get the metadata about the data of source data and redshift tables.

AWS Redshift - Managed warehouse service in AWS. 

Step Function - Used for integrating mutilple services in AWS with each other.

 

Architecture Overview
Here we want to automate our pipeline so that as soon as new file is available in S3 bucket our pipeline get triggered and data gets ingested from S3 into redshift table in AWS.

 

Redshift tables

airports_dim- Contains details like airport name, city name. Sample data given below.

airport_id,city,state,name
10165,Adak Island,AK,Adak
10299,Anchorage,AK,Ted Stevens Anchorage International
10304,Aniak,AK,Aniak Airport
10754,Barrow,AK,Wiley Post/Will Rogers Memorial
10551,Bethel,AK,Bethel Airport
10926,Cordova,AK,Merle K Mudhole Smith

daily_flights_fact - Final Table in Redshift.

CREATE TABLE airlines.daily_flights_fact (
    carrier VARCHAR(10),
    dep_airport VARCHAR(200),
    arr_airport VARCHAR(200),
    dep_city VARCHAR(100),
    arr_city VARCHAR(100),
    dep_state VARCHAR(100),
    arr_state VARCHAR(100),
    dep_delay BIGINT,
    arr_delay BIGINT
);

 

 

Daily Raw Data - flights.csv Sample data shown below.

 

Carrier,OriginAirportID,DestAirportID,DepDelay,ArrDelay
DL,11433,13303,-3,1
DL,14869,12478,0,-8
DL,14057,14869,-4,-15


DL,15016,11433,28,24
DL,11193,12892,-6,-11
DL,10397,15016,-1,-19
DL,15016,10397,0,-1

 

 

A file gets uploaded into the S3 bucket into the partition current_business_date=2024-01-01/flights.csv

As soon as the file is uploaded, the S3 cloudtrail that has been set up for the bucket will send notification to CloudTrail from where the event bridge is continuously looking for the below pattern.



{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["s3.amazonaws.com"],
    "eventName": ["PutObject", "CompleteMultipartUpload"],
    "requestParameters": {
      "bucketName": ["airline-dataset"],
      "key": [{
        "suffix": "/flights.csv"
      }]

    }
  }
}
 

As soon as eventBridge captures the above pattern it triggers the Step Function.

 

Once step Function is triggered the first service it runs is the crawler which crawls the source data.

Reason to run the crawler - Since the daily raw file gets uploaded into the current business date partition which comes under structural change we need to run the crawler so the metadata about the change in strucuture gets uploaded into the AWS Glue Catalog which then can be used during the GlueJobRun phase.

 

We are checking the status of the crawler in every 10 seconds to confirm if it has run successfully only then we will move to the GlueJobRun.

 

Once the crawler run is successfull the glueJobRun starts. It uses the AWS Glue catalog to refer the details of the raw data schema and the tables in redshift and accordingly after doing some joins it finally ingests data into the final table. To achieve icremental load we have enabled job bookmarking for the Glue Job.

 

Once Glue Job runs successfully we get an email notification that data has been ingested successfully.

 

In addition to orchestrating the workflow and integration of services, the project also involved configuring a Virtual Private Cloud (VPC) endpoint to facilitate secure communication between the AWS services utilized. Ensuring that each component had appropriate permissions to interact seamlessly within the AWS environment was a crucial aspect of the project's infrastructure setup.
