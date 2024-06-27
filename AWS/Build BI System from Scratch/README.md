# <a name="top"></a>Implementation of BI system on AWS

The purpose of this lab is to implement a Businesss Intelligence(BI) System using AWS Analytics Services.<br/>

This lab will set up a `Data Collection -> Store -> Analysis/Processing -> Visualization` pipeline.


## Table of Contents
* [Solutions Architecture Overview](#solutions-architecture-overview)
* [Lab setup](#settingups)
* [\[Step-1a\] Create Kinesis Data Streams to receive input data](#kinesis-data-streams)
* [\[Step-1b\] Create Kinesis Data Firehose to store data in S3](#kinesis-data-firehose)
* [\[Step-1c\] Verify data pipeline operation](#kinesis-data-pipeline)
* [\[Step-1d\] Analyze data using Athena](#athena)
* [\[Step-1e\] Data visualization with QuickSight](#amazon-quicksight-visualization)
* [\[Step-1f\] Combine small files stored in S3 into large files using AWS Lambda Function](#athena-ctas-lambda-function)
* [\[Step-2a\] Create Amazon OpenSearch Service for Real-Time Data Analysis](#amazon-es)
* [\[Step-2b\] Ingest real-time data into OpenSearch using AWS Lambda Functions](#amazon-lambda-function)
* [\[Step-2c\] Data visualization with Kibana](#amazon-es-kibana-visualization)
* [Recap and Review](#recap-and-review)
* [Resources](#resources)
* [Reference](#reference)
* [Deployment by AWS CDK](#deployment-by-aws-cdk)

## <a name="solutions-architecture-overview"></a>Solutions Architecture Overview
![aws-analytics-system-architecture](./Arch-diagram.png)


## <a name="settingups"></a>Setting up the environment
Before starting the implementation, create and configure EC2, the IAM user, and VPC with security group
 - [Prepare the environment](./SettingUps.md)


## <a name="kinesis-data-streams"></a>Create Kinesis Data Streams to receive real-time input data

![bi-system-step1](./arch-first-diagram.png)

Navigate to **Kinesis** service on the AWS Management Console to create data stream. <br/>
**On-demand** capacity mode automatically scale to handle gigabytes of write and read throughput per minute.
![kinesis-creation](./part1/kinesis-ds.png)

## <a name="kinesis-data-firehose"></a>Create Amazon Data Firehose to format and store data in S3
Amazon Data Firehose allow ingesting data in real-time from data stream and format it to load into a storage location such as Amazon S3, Amazon Redshift or OpenSearch Service.<br/>
We will load data into an S3 bucket later on. <br/>

![bi-system-step1](./arch-first-diagram.png)

Navigate to **Amazon Data Firehose** from the console and **Create FrieHose stream**
  + Under **Choose a source**, select the **Kinesis Data Stream** and choose `retail-trans` which was created earlier.
  + Select Amazon S3 as **Destination** and click `Create new` to create a new S3 bucket.
  + Quickly created a S3 bucket named `bi-bkt` and navigate back to FireHorse creation.
  + Under **S3 Prefix**, </br>
    Enter S3 prefix as follows:
    ```shell script
    json-data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/
    ```
    Enter S3 error prefix as follows:
    ```shell script
    error-json/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/!{firehose:error-output-type}
    ```
    :information_source: **The S3 prefixes and S3 Error Prefix are used to define the directory structure for storing data in Amazon S3. These prefixes use placeholders that dynamically insert the current timestamp and error type into the S3 key (path) where the data is stored**. (**ref.** [Custom Prefixes for Amazon S3 Objects](https://docs.aws.amazon.com/firehose/latest/dev/s3-prefixes.html))
  + Set buffer size to `1` MB and buffer interval to `60` seconds in **S3 buffer conditions**. Leave everything else as default. </br>
FireHose stream configurations: </br>
 ![firehose-creation](./part1/filehorse-configs.png)


## <a name="kinesis-data-pipeline"></a>Simulate real-time data pipeline operation.
 Generate sample data flow from EC2 host and verify it is being buffered, processed and stored as follows- `EC2 Bastion host -> Kinesis Data Streams -> Amazon FireHose stream -> S3`.

![bi-system-step1](./arch-first-diagram.png)

1. Connect SSH to the previously created EC2 instance.
2. Run `gen_kinesis_data.py` script on the EC2 instance by entering the following command -
    ```shell script
    python3 gen_kinesis_data.py \
      --region-name us-east-2 \
      --service-name kinesis \
      --stream-name retail-trans
    ```
3. Data is generated every second. and buffered into the stream.
4. Go to **S3** service and open the bucket, the data has been delivered by **Data Firehose** and stored in a formatted folder structure by year, month, day, and hour.
Buffered data stored in S3. Noticed the formated folder structure: </br>
 ![s3-object-stored](./part1/s3-object-stored.png)


## <a name="athena"></a>Query data using Athena
**Amazon Athena** allow on-demand execution of SQL queries </br>
We will create tables based on data stored in S3, query those tables using SQL, and view query results.

![bi-system-step1](./arch-first-diagram.png)

### Step 1: Create a database
1. Navigate to **Athena** from the list of services on the AWS Management console.
3. First, set an S3 location to save Athena's query results. We will create a new folder in the same S3 bucket acording to the architechture </br>
 ![aws-athena-setup-query-results-location-01](./part1/athena-q-loc-1.png) </br>
4. Navigate back to **Query editor**
  + Create a new database called `mydatabase` bu execute the SQL command:
    ```shell script
    CREATE DATABASE IF NOT EXISTS mydatabase
    ```
  + Select the newly created database `mydatabase` to peform SQL script on it.
 ![athena-db-select](./part1/athena-db-select.jpeg)

### Step 2: Create a table
Peform basic SQL script to structure a table named `retail_trans_json` in database `mydatabase`.</br>
:information_source: click the **Run Query** button to execute the script. S3 location need to be specified to the designated S3 path.<br>
  ```shell script
    CREATE EXTERNAL TABLE IF NOT EXISTS `mydatabase.retail_trans_json`(
      `invoice` string COMMENT 'Invoice number',
      `stockcode` string COMMENT 'Product (item) code',
      `description` string COMMENT 'Product (item) name',
      `quantity` int COMMENT 'The quantities of each product (item) per transaction',
      `invoicedate` timestamp COMMENT 'Invoice date and time',
      `price` float COMMENT 'Unit price',
      `customer_id` string COMMENT 'Customer number',
      `country` string COMMENT 'Country name')
    PARTITIONED BY (
      `year` int,
      `month` int,
      `day` int,
      `hour` int)
    ROW FORMAT SERDE
      'org.openx.data.jsonserde.JsonSerDe'
    STORED AS INPUTFORMAT
      'org.apache.hadoop.mapred.TextInputFormat'
    OUTPUTFORMAT
      'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
    LOCATION
      's3://aws-analytics-immersion-day-xxxxxxxx/json-data'
  ```
After creating the table, run the query to load the partition data. </br>
  ```shell script
    MSCK REPAIR TABLE mydatabase.retail_trans_json
  ```
Table created using the script: </br>
![athena-db-select](./part1/athena-tbl-creation.jpeg)

### Step 3: Query Data
Use simple SQL statement to query 10 transactions from the table. </br>
  ```shell script
    SELECT *
    FROM retail_trans_json
    LIMIT 10
  ```
  The result is returned:
  ![aws_athena_select_all_limit_10](./part1/athena-q-10-rslt.png)


## <a name="athena-ctas-lambda-function"></a>Combine small files stored in S3 into large files using AWS Lambda Function

:information_source: When real-time incoming data is stored in S3 using  Data Firehose, files with small data size are created.
To improve the query performance of Amazon Athena, it is recommended to combine small files into one large file.
To run these tasks periodically, we are going to create an AWS Lambda function function that executes Athena's Create Table As Select (CTAS) query.

![aws-analytics-system-build-steps-extra](./arch-first-diagram.png)

### Step 1: Create a table to store CTAS query results
1. Access **Athena Query Editor**, set an S3 location to save Athena's CTAS query results output
![athena-ctas-loc](./part1/athena-q-loc2.jpeg)
3. Select `mydatabase` to peform SQL script on it. The SQL script change the json format data of the original `retal_tran_json` table into parquet format and store it in a table called `ctas_retail_trans_parquet`.<br/>
    ```shell script
    CREATE EXTERNAL TABLE `mydatabase.ctas_retail_trans_parquet`(
      `invoice` string COMMENT 'Invoice number',
      `stockcode` string COMMENT 'Product (item) code',
      `description` string COMMENT 'Product (item) name',
      `quantity` int COMMENT 'The quantities of each product (item) per transaction',
      `invoicedate` timestamp COMMENT 'Invoice date and time',
      `price` float COMMENT 'Unit price',
      `customer_id` string COMMENT 'Customer number',
      `country` string COMMENT 'Country name')
    PARTITIONED BY (
      `year` int,
      `month` int,
      `day` int,
      `hour` int)
    ROW FORMAT SERDE
      'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
    STORED AS INPUTFORMAT
      'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
    OUTPUTFORMAT
      'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
    LOCATION
      's3://aws-analytics-immersion-day-xxxxxxxx/parquet-retail-trans'
    TBLPROPERTIES (
      'has_encrypted_data'='false',
      'parquet.compression'='SNAPPY')
    ;
    ```

### Step 2: Create an AWS Lambda Function
:information_source: This function is to merge small file every 5 minutes using the transition script in the `athena_ctas.py`. We also grant nesseary access role to the function so that it can tranform the data using Athena, and store in S3.
1. From the **AWS Lambda Console**, create a python function named `MergeSmallFiles` 
2. Make **CloudWatch Events/EventBridge** as a trigger to execute the function.
3. for event **Trigger configuration**,
  + `Schedule expression` as the rule type
  + enter `cron(5 * * * *)` </br>
  This will run the task every 5 minutes as a scheduled expression. </br>
  cron(min, hour, D, M, DoW )
4. In **Trigger configuration**, click **\[Add\]**.
5. Copy and paste the script from the `athena_ctas.py` [file](./athena_ctas.py) into the code editor of the Function code. Click **Deploy**.
6. Click **\[Add environment variables\]** to register the following environment variables.
  ```shell script
    OLD_DATABASE=mydatabase
    OLD_TABLE_NAME=retail_trans_json
    NEW_DATABASE=mydatabase
    NEW_TABLE_NAME=ctas_retail_trans_parquet
    WORK_GROUP=primary
    OLD_TABLE_LOCATION_PREFIX=s3://aws-analytics-immersion-day-xxxxxxxx/json-data
    OUTPUT_PREFIX=s3://aws-analytics-immersion-day-xxxxxxxx/parquet-retail-trans
    STAGING_OUTPUT_PREFIX=s3://aws-analytics-immersion-day-xxxxxxxx/tmp
    COLUMN_NAMES=invoice,stockcode,description,quantity,invoicedate,price,customer_id,country
  ``` 
7. Grant IAM Policy required for the`MergeSmallFiles-role-XXXXXXXX` under execution role configuration of the fuction
 ![aws-athena-ctas-lambda-execution-iam-role](./part1/athena-ctas-role.jpeg)
8. Attach policies to the role, namely **AmazonAthenaFullAccess** and **AmazonS3FullAccess** in order.
 ![aws-athena-ctas-lambda-iam-role-policies](./part1/merge-sm-files-perm.png)
9. For **Basic settings** of the function. set Timout to `5 min`.

## <a name="amazon-quicksight-visualization"></a>Data visualization with QuickSight
As the final goal, we use Amazon QuickSight to visualize the data that was collected by Kinesis, stored in S3, and analyzed using Athena previously.

![aws-analytics-system-build-steps](./assets/aws-analytics-system-build-steps.svg)

1. Go to [QuickSight Console](https://quicksight.aws.amazon.com).
2. Click the **Sign up for QuickSight** button to sign up for QuickSight.
3. Select Standard Edition and click the **Continue** button.
4. Specify a QuickSight account name. This name should be unique to you, so use the unique string in the account name similar to how you did for the S3 bucket name earlier. Enter your personal email address under **Notification email address**.
5. QuckSight needs access to S3 to be able to read data. Check the **Amazon S3** box, and select `aws-analytics-immersion-day-xxxxxxxx` bucket from the list. Click **Finish**.
   ![aws-quicksight-choose-s3-bucket](./assets/aws-quicksight-choose-s3-bucket.png)
6. After the account is created, click the **Go to Amazon QuickSight** button. Confirm that you are in `US West (Oregon)` region. Click on the account name on the top right corner and select **US West (Oregon)** if it is not already set to Oregon. Click the **New Analysis** button and click on **New dataset** on the next screen.
   ![aws-quicksight-new_data_sets](./assets/aws-quicksight-new_data_sets.png)
7. Click `Athena` and enter `retail-quicksight` in the Data source name in the pop-up window.
Click **Validate connection** to change to `Validated`, then click the **Create data source** button.
   ![aws-quicksight-athena_data_source](./assets/aws-quicksight-athena_data_source.png)
8. On the **Choose your table** screen, select Catalog `AwsDataCatalog`, Database `mydatabase` and Tables `retail_trans_json`. Click the **Select** button.
   ![aws-quicksight-athena-choose_your_table](./assets/aws-quicksight-athena-choose_your_table.png)
9. On the **Finish dataset creation** screen, choose `Directly query your data` and click the **Visualize** button.
   ![aws-quicksight-finish-dataset-creation](./assets/aws-quicksight-finish-dataset-creation.png)
10. Let's visualize the `Quantity` and `Price` by `InvoiceDate`. Select vertical bar chart from the **Visual types** box on the bottom left. In the **field wells**, drag `invoicedate` from the left panel into **X axis**, drag `price`, and `quantity` into **Value**. You will see a chart get populated as shown below.
   ![aws-quicksight-bar-chart](./assets/aws-quicksight-bar-chart.png)
11. Let's share the Dashboard we just created with other users. Click on the account name on the top right corner and select **Manage QuickSight**.
12. Click the `+` button on the right side, and enter an email address of the person with whom you want to share the visualization. Click the **Invite** button and close the popup window.</br>
   ![aws-quicksight-user-invitation](./assets/aws-quicksight-user-invitation.png)
13. Users you invite will receive the following Invitation Email. They can click the button to accept invitation.
   ![aws-quicksight-user-email](./assets/aws-quicksight-user-email.png)
14. Return to the QuickSight home screen, select your analysis, and click **Share> Share analysis** from the upper right corner.
   ![aws-quicksight-share-analysis](./assets/aws-quicksight-share-analysis.png)
15. Select `BI_user01` and click the Share button.
   ![aws-quicksight-share-analysis-users](./assets/aws-quicksight-share-analysis-users.png)
16. Users receive the following email: You can check the analysis results by clicking **Click to View**.
   ![aws-quicksight-user-email-click-to-view](./assets/aws-quicksight-user-email-click-to-view.png)





## <a name="amazon-es"></a>Create Amazon OpenSearch Service for Real-Time Data Analysis

An OpenSearch cluster is created to store and analyze data in real time. An OpenSearch Service domain combined of clusters with the settings, instance types, instance counts, and storage resources.

![aws-analytics-system-build-steps](./Arch-diagram.png)

1. Navigate to **Amazon OpenSearch Service** and **Create a new domain**.
2. Provide a name for the domain. I'm using `salesv1`.
3. For the deployment type, choose **Production**.
4. For **Version**, I used the latest version. 
5. Under **Data nodes**, select a instant type as needed.
6. Under **Network**
  + choose **VPC access (recommended)**. I use the same VPC that is created earlier
  + I select `es-cluster-sg` as Security Groups, which allows traffic from `use-es-cluster-sg` (attached to Lambda Function and EC2)
7. In the fine-grained access control settings, **Create master user** with `username` and `password`
8. Under **Access policy**, i choose **Only use fine-grained access control** to only access to the service using my `username` and `password` </br>

Open search domain configs:
![ES-cluster-config](./part2/open-search-cluster-conf.png)


## <a name="amazon-lambda-function"></a>Ingest real-time data into OpenSearch using AWS Lambda Functions

We can index data into Amazon OpenSearch Service in realtime using different ways, including, Kinesis injestion, using a Lambda function, or from any storage services
For this architechture diagram, I will use the method of Lambda function since we will need to format raw data from the stream.

![aws-analytics-system-build-steps](./Arch-diagram.png)

### Generate and add a library to Layers for use by the Lambda functions,
1. Generate `es-lib` library as `.zip` and store it in S3
  + From the CLI of `Bastion` EC2 instance, run the following command:
    ```shell script
      $ python3 -m venv es-lib
      $ cd es-lib
      $ source bin/activate
      (es-lib)$ mkdir -p python_modules
      (es-lib)$ pip install 'elasticsearch < 7.14' requests requests-aws4auth -t python_modules
      (es-lib)$ mv python_modules python
      (es-lib)$ zip -r es-lib.zip python/
      (es-lib)$ aws s3 mb s3://my-bucket-for-lambda-layer-packages
      (es-lib)$ aws s3 cp es-lib.zip s3://my-bucket-for-lambda-layer-packages/python/
    ```
  + Replace the S3 path to actual bucket path.
2. From **AWS Lambda Console**, create a new layer called `es-lib`
  + Select `Upload a file from Amazon S3` and enter the s3 path of `es-lib.zip` file created earlier
  + Use `Python 3.11` as `Compatible runtimes`.

### Create a Lambda function to format data from the stream and injest it to OpenSearch Service
1. From **AWS Lambda Console**, **Create a function** called `UpsertToES`.
  + Select `Python 3.11` in Runtime.
2. In the Designer tab. **Add a layer** and choose the `es-lib` as **Custom Layers**
3. Select `UpsertToES` in the Designer tab to return to Function code and Configuration.
4. Copy and paste the code from the `upsert_to_es.py` [file](./upsert_to_es.py) and **Deploy**.
5. Add necessary environment variables to register. Replace variables as needed </br>
  ```shell script
    ES_HOST=vpc-retail-xkl5jpog76d5abzhg4kyfilymq.us-west-1.es.amazonaws.com
    ES_INDEX=retail
    ES_TYPE=trans
    REQUIRED_FIELDS=Invoice,StockCode,Customer_ID
    REGION_NAME=us-west-2
    DATE_TYPE_FIELDS=InvoiceDate
  ```
6. In order to execute the lambda function in the VPC and read data from Kinesis Data Streams, add the IAM Policy required for the Execution role required to execute the lamba function.
7. Modify the execution role of the function and Attach policies: **AWSLambdaVPCAccessExecutionRole** and **AmazonKinesisReadOnlyAccess** in order.
 ![aws-lambda-iam-role-policies](./part2/labmda-opensearch-role-perm.png)
8. Add a inline policy statements to enables the lambda function to ingest data into the `salesv1` index in the opensearch service.
    <pre>
    {
        "Action": [
            "es:DescribeElasticsearchDomain",
            "es:DescribeElasticsearchDomainConfig",
            "es:DescribeElasticsearchDomains",
            "es:ESHttpPost",
            "es:ESHttpPut"
        ],
        "Resource": [
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/*"
        ],
        "Effect": "Allow"
    },
    {
        "Action": "es:ESHttpGet",
        "Resource": [
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/_all/_settings",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/_cluster/stats",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/_nodes",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/_nodes/*/stats",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/_nodes/stats",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/_stats",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/retail*/_mapping/trans",
            "arn:aws:es:<i>region</i>:<i>account-id</i>:domain/salesv1/retail*/_stats"
        ],
        "Effect": "Allow"
    }
    </pre>
9. **Edit** the VPC of the function and select a `Custom VPC` for VPC connection.
Use the same VPC and subnets where the domain for the OpenSearch service is created, and choose the security groups that are allowed access to the OpenSearch service domain, in this case, `use-es-cluster-sg`
10. Select **Edit** in Basic settings. Adjust Memory and Timeout appropriately. In this lab, we set Timout to `5 min`.
11. **Add trigger** and pick **Kinesis** as a trigger.
12. Select the Kinesis Data Stream (`retail-trans`) created earlier in **Kinesis stream**.
Overall `UpsertToES` Function should look like this:
 ![aws-lambda-kinesis](./part2/lambda-f-overview.png)

### <a name="create-firehose-role"></a>Enable the Lambda function to ingest records into Amazon OpenSearch

The lambda function uses the delivery role to sign HTTP (Signature Version 4) requests before sending the data to the Amazon OpenSearch Service endpoint. This section describes how to create roles and set permissions for the lambda function. </br>
The Amazon OpenSearch cluster is provisioned in a VPC. Hence, the Amazon OpenSearch endpoint and the Kibana endpoint are not available over the internet. In order to access the endpoints, we have to create a ssh tunnel and do local port forwarding. <br/>

1. Using SSH Tunneling via CLI command from a local PC. This tunnel will connect to the `Bastion` host then do port forwarding to the OpenSearch service domain.</br>
  ```shell script
    ssh -i ~/.ssh/<key.pem> ec2-user@<ip-of-bastion> -N -L 9200:<VPC-enpoint-of-domain>:443
  ```
  Replace`<ip-of-bastion>` and `<VPC-enpoint-of-domain>` as neccesary, `<key.pem>` is used to access the `Bastion` and is stored in the  directory of `~/.ssh/`.Onced opened, the tunnel will run indefinitely, use **Ctrl+C** to disrupt it.
2. Access open search service via `https://localhost:9200/_dashboards/app/login?` in a web browser.
3. Enter the master `username` and `password` for Amazon OpenSearch Service endpoint.
4. From the dashboaRd, configure **Security** in the toolbar.
5. Under **Security**, create a new role called `firehose_role`.
    + For cluster permissions, add `cluster_composite_ops` and `cluster_monitor`.
    + Under **Index permissions**, choose **Index Patterns** and enter <i>index-name*</i>; for example, `retail*`.
    + Under **Permissions**, add three action groups: `crud`, `create_index`, and `manage`.
  ![ops-create-firehose_role](./part2/filehose-role.png)

Next, map the execution IAM role of `UpsertToES`function to the `filehose-role`
6. Under **Mapped users** tab
    + Choose **Manage mapping** and under **Backend roles**,
    + For **Backend Roles**, enter the IAM role ARN of `UpsertToES:
    `arn:aws:iam::123456789012:role/UpsertToESServiceRole709-xxxxxxxxxxxx`.


## <a name="amazon-es-kibana-visualization"></a>Data visualization with Kibana

Visualize data collected from Amazon OpenSearch Service using Kibana.

![aws-analytics-system-build-steps](./Arch-diagram.png)

The Amazon OpenSearch cluster is provisioned in a VPC. Hence, the Amazon OpenSearch endpoint and the Kibana endpoint are not available over the internet. In order to access the endpoints, we have to create a ssh tunnel and do local port forwarding. <br/>
1. Using SSH Tunneling via CLI command from a local PC. This tunnel will connect to the `Bastion` host then do port forwarding to the OpenSearch service domain.</br>
    ```shell script
      ssh -i ~/.ssh/<key.pem> ec2-user@<ip-of-bastion> -N -L 9200:<VPC-enpoint-of-domain>:443
    ```
    Replace`<ip-of-bastion>` and `<VPC-enpoint-of-domain>` as neccesary, `<key.pem>` is used to access the `Bastion` and is stored in the  directory of `~/.ssh/`.Onced opened, the tunnel will run indefinitely, use **Ctrl+C** to disrupt it.
2. Access open search service via `https://localhost:9200/_dashboards/app/login?` in a web browser. </br>
3. Enter the master `username` and `password` for Amazon OpenSearch Service endpoint.</br>
5. From the left toolbar, (Management / Create index pattern)  **Create index pattern**, enter `retail*` in Index pattern.
   ![ops-create-index-pattern](./assets/ops-create-index-pattern.png)
6. (Management / Create index pattern) Choose **Next step**.
7. (Management / Create index pattern) Select `InvoiceDate` for the **Time Filter field name** in **Step 2 of 2: Configure settings** of the Create index pattern.
   ![ops-create-index-pattern-configure-setting](./assets/ops-create-index-pattern-configure-setting.png)
8. (Management / Create index pattern) Click **Create index pattern**.
9. (Management / Advanced Settings) After selecting **Advanced Settings** from the left sidebar menu, set **Timezone for date formatting** to `Etc/UTC`. Since the log creation time of the test data is based on `UTC`, **Kibana**'s **Timezone** is also set to `UTC`.
    ![kibana-02d-management-advanced-setting](./assets/ops-management-advanced-setting.png)
10. (Discover) After completing the creation of **Index pattern**, select **Discover** to check the data collected in OpenSearch.
    ![kibana-03-discover](./assets/kibana-03-discover.png)
11. (Discover) Let's visualize the `Quantity` by `InvoicdDate`. Select **invoicdDate** from **Available fields** on the left, and click **Visualize** at the bottom
    ![kibana-04-discover-visualize](./assets/kibana-04-discover-visualize.png)
12. (Visualize) After selecting **Y-Axis** in **Metrics** on the Data tab, apply `Sum` for **Aggregation**, and `Quantity` for **Field** as shown below.
    ![kibana-05-discover-change-metrics](./assets/kibana-05-discover-change-metrics.png)
13. (Visualize) Click **Save** in the upper left corner, write down the name of the graph you saved, and then click **Confirm Save**.
    ![kibna-08-visualize-save](./assets/kibana-08-visualize-save.png)
14. (Dashboards) Click **Dashboard** icon on the left and click the **Create new dashboard** button.
    ![kibana-09-dashboards](./assets/kibana-09-dashboards.png)
15. (Dashboards) You can see the following Dashboards.
    ![kibana-13-complete](./assets/kibana-13-complete.png)

\[[Top](#top)\]

## Recap and Review

:warning: **At the end of this lab, you should delete the resources you used to avoid incurring additional charges for the AWS account you used.**

Through this lab, we have built a Business Intelligent System with Lambda Architecture such that consists of real-time data processing and batch data processing layers.

\[[Top](#top)\]

## Resources
+ slide: [AWS Analytics Immersion Day - Build BI System from Scratch](http://tinyurl.com/serverless-bi-on-aws)
+ data source: [Online Retail II Data Set](https://archive.ics.uci.edu/ml/datasets/Online+Retail+II)

\[[Top](#Top)\]

## Reference
### AWS Developer Guide By Services
+ [Amazon Simple Storage Service (Amazon S3)](https://docs.aws.amazon.com/AmazonS3/latest/dev/Introduction.html)
+ [Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
+ [Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html)
+ [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
+ [Amazon Kinesis Data Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
+ [Amazon Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
+ [Amazon QuickSight](https://docs.aws.amazon.com/quicksight/latest/user/welcome.html)
+ [AWS Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html#configuration-layers-path)
    + <a name="aws-lambda-layer-python-packages"></a>Example of creating a python package to register with AWS Lambda layer: **elasticsearch**

      :warning: **You should create the python package on Amazon Linux, otherwise create it using a simulated Lambda environment with Docker.**
      <pre>
      [ec2-user@ip-172-31-6-207 ~] $ python3 -m venv es-lib
      [ec2-user@ip-172-31-6-207 ~] $ cd es-lib
      [ec2-user@ip-172-31-6-207 ~] $ source bin/activate
      (es-lib) $ mkdir -p python_modules
      (es-lib) $ pip install opensearch-py==2.0.1 requests==2.31.0 requests-aws4auth==1.1.2 -t python_modules
      (es-lib) $ mv python_modules python
      (es-lib) $ zip -r es-lib.zip python/
      (es-lib) $ aws s3 mb s3://my-bucket-for-lambda-layer-packages
      (es-lib) $ aws s3 cp es-lib.zip s3://my-bucket-for-lambda-layer-packages/var/
      (es-lib) $ deactivate
      </pre>
    + [How to create a Lambda layer using a simulated Lambda environment with Docker](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-layer-simulated-docker/)
      ```
      $ cat <<EOF > requirements.txt
      > opensearch-py==2.0.1
      > requests==2.31.0
      > requests-aws4auth==1.1.2
      > EOF
      $ docker run -v "$PWD":/var/task "public.ecr.aws/sam/build-python3.11" /bin/sh -c "pip install -r requirements.txt -t python/lib/python3.11/site-packages/; exit"
      $ zip -r es-lib.zip python > /dev/null
      $ aws s3 mb s3://my-bucket-for-lambda-layer-packages
      $ aws s3 cp es-lib.zip s3://my-bucket-for-lambda-layer-packages/var/
      ```

### <a name="SSH-Tunnel-with-PuTTy-on-Windows"></a>SSH Tunnel for Kibana Instructions with PuTTy on Windows
+ [Windows SSH / Tunnel for Kibana Instructions - Amazon Elasticsearch Service](https://search-sa-log-solutions.s3-us-east-2.amazonaws.com/logstash/docs/Kibana_Proxy_SSH_Tunneling_Windows.pdf)
+ [Use an SSH Tunnel to access Kibana within an AWS VPC with PuTTy on Windows](https://amazonmsk-labs.workshop.aws/en/mskkdaflinklab/createesdashboard.html)

\[[Top](#top)\]

### Further readings
##### Amazon S3
+ [New – Automatic Cost Optimization for Amazon S3 via Intelligent Tiering](https://aws.amazon.com/ko/blogs/aws/new-automatic-cost-optimization-for-amazon-s3-via-intelligent-tiering/)

##### Amazon Athena
+ [Top 10 Performance Tuning Tips for Amazon Athena](https://aws.amazon.com/ko/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)
+ [Extract, Transform and Load data into S3 data lake using CTAS and INSERT INTO statements in Amazon Athena](https://aws.amazon.com/ko/blogs/big-data/extract-transform-and-load-data-into-s3-data-lake-using-ctas-and-insert-into-statements-in-amazon-athena/)
+ [Query Amazon S3 analytics data with Amazon Athena](https://aws.amazon.com/blogs/storage/query-amazon-s3-analytics-data-with-amazon-athena/)

##### Amazon Elasticsearch Service
+ [Elasticsearch tutorial: a quick start guide](https://aws.amazon.com/blogs/database/elasticsearch-tutorial-a-quick-start-guide/)
+ [Run a petabyte scale cluster in Amazon Elasticsearch Service](https://aws.amazon.com/blogs/database/run-a-petabyte-scale-cluster-in-amazon-elasticsearch-service/)
+ [Analyze user behavior using Amazon Elasticsearch Service, Amazon Kinesis Data Firehose and Kibana](https://aws.amazon.com/blogs/database/analyze-user-behavior-using-amazon-elasticsearch-service-amazon-kinesis-data-firehose-and-kibana/)

##### AWS Lambda
+ [Introduction to Messaging for Modern Cloud Architecture](https://aws.amazon.com/blogs/architecture/introduction-to-messaging-for-modern-cloud-architecture/)
+ [Understanding the Different Ways to Invoke Lambda Functions](https://aws.amazon.com/blogs/architecture/understanding-the-different-ways-to-invoke-lambda-functions/)

##### Amazon Kinesis Data Firehose
+ [Amazon Kinesis Data Firehose custom prefixes for Amazon S3 objects](https://aws.amazon.com/blogs/big-data/amazon-kinesis-data-firehose-custom-prefixes-for-amazon-s3-objects/)
+ [Amazon Kinesis Firehose Data Transformation with AWS Lambda](https://aws.amazon.com/blogs/compute/amazon-kinesis-firehose-data-transformation-with-aws-lambda/)

##### Amazon Kinesis Data Streams
+ [Under the hood: Scaling your Kinesis data streams](https://aws.amazon.com/blogs/big-data/under-the-hood-scaling-your-kinesis-data-streams/)
+ [Scale Amazon Kinesis Data Streams with AWS Application Auto Scaling](https://aws.amazon.com/blogs/big-data/scaling-amazon-kinesis-data-streams-with-aws-application-auto-scaling/)

##### Amazon Kinesis Data Analytics
+ [Streaming ETL with Apache Flink and Amazon Kinesis Data Analytics](https://aws.amazon.com/ko/blogs/big-data/streaming-etl-with-apache-flink-and-amazon-kinesis-data-analytics/)

##### Amazon QuickSight
+ [10 visualizations to try in Amazon QuickSight with sample data](https://aws.amazon.com/blogs/big-data/10-visualizations-to-try-in-amazon-quicksight-with-sample-data/)
+ [Visualize over 200 years of global climate data using Amazon Athena and Amazon QuickSight](https://aws.amazon.com/blogs/big-data/visualize-over-200-years-of-global-climate-data-using-amazon-athena-and-amazon-quicksight/)
+ [Advanced analytics with table calculations in Amazon QuickSight](https://aws.amazon.com/ko/blogs/big-data/advanced-analytics-with-table-calculations-in-amazon-quicksight/)

##### Etc
+ [Optimize downstream data processing with Amazon Kinesis Data Firehose and Amazon EMR running Apache Spark](https://aws.amazon.com/blogs/big-data/optimizing-downstream-data-processing-with-amazon-kinesis-data-firehose-and-amazon-emr-running-apache-spark/)
+ [Serverless Scaling for Ingesting, Aggregating, and Visualizing Apache Logs with Amazon Kinesis Firehose, AWS Lambda, and Amazon Elasticsearch Service](https://aws.amazon.com/blogs/database/serverless-scaling-for-ingesting-aggregating-and-visualizing-apache-logs-with-amazon-kinesis-firehose-aws-lambda-and-amazon-elasticsearch-service/)
+ [Analyze Apache Parquet optimized data using Amazon Kinesis Data Firehose, Amazon Athena, and Amazon Redshift](https://aws.amazon.com/blogs/big-data/analyzing-apache-parquet-optimized-data-using-amazon-kinesis-data-firehose-amazon-athena-and-amazon-redshift/)
+ [Our data lake story: How Woot.com built a serverless data lake on AWS](https://aws.amazon.com/blogs/big-data/our-data-lake-story-how-woot-com-built-a-serverless-data-lake-on-aws/)

##### Securely Connect Bastion Hosts
+ [Securing your bastion hosts with Amazon EC2 Instance Connect](https://aws.amazon.com/blogs/infrastructure-and-automation/securing-your-bastion-hosts-with-amazon-ec2-instance-connect/)

  ```
  $ # (1) Create a new ssh key.
  $ ssh-keygen -t rsa -f my_rsa_key

  $ # (2) Push your SSH public key to the instance.
  $ aws ec2-instance-connect send-ssh-public-key \
    --instance-id $BASTION_INSTANCE \
    --availability-zone $DEPLOY_AZ \
    --instance-os-user ec2-user \
    --ssh-public-key file:///path/to/my_rsa_key.pub

  $ # (3) Connect to the instance using your private key.
  $ ssh -i /path/to/my_rsa_key ec2-user@$BASTION_DNS_NAME
  ```

+ [Connect using the EC2 Instance Connect CLI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html#ec2-instance-connect-connecting-ec2-cli)
   <pre>
   $ sudo pip install ec2instanceconnectcli
   $ mssh ec2-user@i-001234a4bf70dec41EXAMPLE # ec2-instance-id
   </pre>

\[[Top](#top)\]

## Deployment by AWS CDK

:warning: **At the end of this lab, you should delete the resources you used to avoid incurring additional charges for the AWS account you used.**

Introducing how to deploy using the AWS CDK.

### Prerequisites
1. Install AWS CDK Toolkit.

    ```shell script
    npm install -g aws-cdk
    ```

2. Verify that cdk is installed properly by running the following command:
    ```
    cdk --version
    ```
   ex)
    ```shell script
    $ cdk --version
    2.41.0 (build 56ba2ab)
    ```

##### Useful commands

 * `cdk ls`          list all stacks in the app
 * `cdk synth`       emits the synthesized CloudFormation template
 * `cdk deploy`      deploy this stack to your default AWS account/region
 * `cdk diff`        compare deployed stack with current state
 * `cdk docs`        open CDK documentation

\[[Top](#top)\]

### Deployment

When deployed as CDK, `1(a), 1(b), 1(c), 1(f), 2(b), 2(a)` in the architecture diagram below are automatically created.

![aws-analytics-system-build-steps-extra](./assets/aws-analytics-system-build-steps-extra.svg)

1. Refer to [Getting Started With the AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html) to install cdk.
Create an IAM User to be used when running cdk and register it in `~/.aws/config`. (**cf.** [Creating an IAM User](#preliminaries))<br/>
For example, after creating an IAM User called cdk_user, add it to `~/.aws/config` as shown below.

    ```shell script
    $ cat ~/.aws/config
    [profile cdk_user]
    aws_access_key_id=AKIAIOSFODNN7EXAMPLE
    aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    region=us-west-2
    ```

2. Create a Python package to register in the Lambda Layer and store it in the s3 bucket. For example, create an s3 bucket named `lambda-layer-resources` so that you can save the elasticsearch package to register in the Lambda Layer as follows.

    ```shell script
    $ aws s3 ls s3://lambda-layer-resources/var/
    2019-10-25 08:38:50          0
    2019-10-25 08:40:28    1294387 es-lib.zip
    ```

3. After downloading the source code from git, enter the s3 bucket name where the package to be registered in the lambda layer is stored in an environment variable called `S3_BUCKET_LAMBDA_LAYER_LIB`.
After setting, deploy using the `cdk deploy` command.

    ```shell script
    $ git clone https://github.com/aws-samples/aws-analytics-immersion-day.git
    $ cd aws-analytics-immersion-day
    $ python3 -m venv .env
    $ source .env/bin/activate
    (.env) $ pip install -r requirements.txt
    (.env) $ export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
    (.env) $ export CDK_DEFAULT_REGION=us-west-2
    (.env) $ cdk bootstrap aws://${CDK_DEFAULT_ACCOUNT}/${CDK_DEFAULT_REGION}
    (.env) $ export S3_BUCKET_LAMBDA_LAYER_LIB=lambda-layer-resources
    (.env) $ cdk --profile cdk_user deploy --require-approval never --all
    ```

   :white_check_mark: `cdk bootstrap ...` command is executed only once for the first time to deploy **CDK toolkit stack**, and for subsequent deployments, you only need to execute `cdk deploy` command without distributing **CDK toolkit stack**.

    ```shell script
    (.env) $ export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
    (.env) $ export CDK_DEFAULT_REGION=us-west-2
    (.env) $ export S3_BUCKET_LAMBDA_LAYER_LIB=lambda-layer-resources
    (.env) $ cdk --profile cdk_user deploy --require-approval never --all
    ```

3. [Enable the Lambda function to ingest records into Amazon OpenSearch.](#create-firehose-role)

### Clean Up

To delete the deployed application, execute the `cdk destroy` command as follows.

    (.env) $ cdk --profile cdk_user destroy --force --all

\[[Top](#top)\]

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.