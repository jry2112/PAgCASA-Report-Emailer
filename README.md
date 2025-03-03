# PAgCASA Report Emailer
Produced by Robert Behring, Eric Riemer, and Jada Young in association with Jim Cupples of PAgCASA.
<br>For More information about PAgCASA and the work they do, go to https://www.pagcasa.org/

## Overview
---
The PAgCASA Report Emailer provides users a simple way to visualize the data collected from M-Lab's Murakami testing tool (https://github.com/m-lab/murakami). The Murakami testing tool natively outputs data in a JSON new line format (.jsonl) which is hard to read and just as difficult when comparing data from different tests. The PAgCASA Report Emailer provides an email report to a user with either daily or weekly frequency. 


## Configuration
---
The PAgCASA Report Emailer is intended for use within the Google Cloud Platform (GCP) environment and thus requires several Google Cloud Service (GCS) tools and the Murakami testing tool. 

1. Murakami Testing Tool (https://github.com/m-lab/murakami)
    - data must be collected using M-Lab's Murakami testing tool in either NDT-7-client-go or Ookla's speedtest-cli test format.
2. GCS Buckets
    - Your Murakami testing must provide the .jsonl files directly to a GCS bucket
3. GCS BigQuery
    - .jsonl data from GCS Buckets will be exported to a BigQuery dataset
4. GCS Cloud Functions gen 2.0
    - The automation tools within the PAgCASA Report Emailer are all built in the GCS Cloud Functions gen 2.0. 


## Deployment onto your GCP
---
The deployment of the PAgCASA Report Emailer is broken into three steps. It is assumed that the user has already deployed M-Lab's Murakami testing tool onto their own GCP.

The below guide provides two installation options for creating the required GCP Resources.
### **1) Quick Install Guide**

**Getting Started**  
Prior to beginning, ensure that you have downloaded the required source files and are in the correct GCP project using the following instructions:
1. Navigate to the directory you want to store your source code
2. Clone the associated source code git repository from GitHub using the link above or the following line of code in your terminal

```bash
git clone https://github.com/RobertBehring/PAgCASA-Report-Emailer.git
```

3. Make sure you are in the correct GCP project in the gcloud CLI. The following code will allow you to set your code according to the PROJECT_ID you provide.

```bash
gcloud config set project PROJECT_ID
```

**GCP Services and Permissions**  
The PAgCASA Report Emailer Functions requires the following GCP IAM Permissions to be enabled:
- iam.serviceAccountUser
- cloudfunctions.admin
- pubsub.publisher

The PAgCASA Report Emailer Functions requires the following GCP Resources:
- run.googleapis.com 
- logging.googleapis.com 
- cloudbuild.googleapis.com 
- storage.googleapis.com 
- pubsub.googleapis.com 
- eventarc.googleapis.com

Users may enable the required services and permissions using either the GCP Console or by uncommenting and updating the required fields in config/setup.sh.

**Installation**  
Prior to running the command navigate to `config/setup.sh` and each function's `.env` file and update all required project information. In any bucket of your choosing, include a .csv file with all the email addresses that you'd like to recieve the data report, ensuring that the first column of that .csv file contains only email addresses.  
In the gcloud CLI, enable file execution then run the configuration file as shown, substituing DATA_BUCKET_NAME with the name of your GCP Data Bucket:
```bash
python chmod u+x ./config/setup.sh
./config.setup.sh DATA_BUCKET_NAME
```
> **Note**  
> The Export-to-BQ Cloud Function only monitors for the creation of new objects and the editing of existing objects stored in the designated GCS Bucket. If you would like to incorporate older data, it is recommended to setup the project using a new bucket, then transfer the pre-existing data. 

### **2) Step-By-Step Guide**
### BigQuery Dataset
BigQuery is a cloud service offered by Google LLC and is available within their Google Cloud Platform (GCP). Before installation of BigQuery, one must first sign up for a Google account and gain access to their own private/shared GCP. This installation guide will cover the creation of the BigQuery dataset (DeviceBroadbandData) and the associated tables (Multistream and NDT-7). 

1. Navigate to the directory you want to store your source code
2. Clone the associated source code git repository from GitHub using the link above or the following line of code in your terminal

```bash
git clone https://github.com/RobertBehring/PAgCASA-Report-Emailer.git
```

3. Make sure you are in the correct GCP project in the gcloud CLI. The following code will allow you to set your code according to the PROJECT_ID you provide.

```bash
gcloud config set project PROJECT_ID
```

4. Finally run the DDL.py

```bash
python DDL.py
```

> **Note**
> The preceding code will create a BigQuery dataset named DeviceBroadbandData and two tables, Multistream and NDT-7.


### Export to BigQuery
The GCS Bucket Migration Tool is an automation designed to detect when broadband test data is uploaded to a target GCP Storage Bucket and transfers the data to the proper BigQuery Table. This tool assumes that you have created the BigQuery Dataset as described in the previous section. If you have not done so, please create the BigQuery Dataset prior to deploying this function. 

1. (If required) Create a new bucket to hold speed test json data by running the following command in the Google CLI:

```bash
gcloud storage buckets create gs://YOUR_DATA_BUCKET_NAME
```

2. Next, make sure that you are in the source code folder, then deploy the function by running the following command:
```bash
gcloud functions deploy get-new-upload \
--gen2 \
--region=us-west1 \
--runtime=python310 \
--source=./src/export_to_bq \
--entry-point=get_new_data \
--trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
--trigger-event-filters="bucket=YOUR_DATA_BUCKET_NAME" \
--service-account=YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com
```

> **Note**
> This command will deploy the function and create an Eventarc trigger that automatically detects when new files have been added to the source data bucket. After confirmation, users may view/update the function using the GCP Console.


### Email CSV Report
The CSV Report Emailer (email_csv folder) tool is a cloud function program that utilizes other cloud services to run. Globally, you will need to have a Google Cloud Platform (GCP) environment along with the associated Google Cloud Service BigQuery and the respective dataset. Do not continue if you have not set up your BigQuery dataset as outlined above in the “Creating the BigQuery Dataset” section. This section will go through how to deploy the email_csv.py program in your GCP environment. 

1. Create a Pub/Sub Topic by running the following command:

```bash
gcloud pubsub topics create export_to_csv
```
This will create a new GCP Pub/Sub Topic that receives messages from the Cloud Scheduler jobs and is subscribed to by the Cloud Function. 

2. Deploy the function by running the following command:

```bash
gcloud functions deploy email-csv \
--gen2 \
--region=us-west1 \
--runtime=python310 \
--source=./src/email_csv \
--entry-point send_csv_email \
--trigger-topic=export_to_csv \
--service-account=YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com
```

This command deploys the function and creates a trigger that subscribes to the topic created in the previous step.

3. Create the Cloud Scheduler job to automatically send the email results. 

```bash
gcloud scheduler jobs create pubsub weekly_export_to_csv \
--schedule="00 7 * * *" \
--location="us-west1" \
--topic export_to_csv \
--message-body="Sent Weekly Email"
```

```bash
gcloud scheduler jobs create pubsub daily_export_to_csv \
--schedule="0 9 * * *" \
--location="us-west1" \
--topic export_to_csv \
--message-body="Sent Daily Email"
```

> **Note** 
>The schedule flag can be substituted with any cron expression to customize the sending intervals. The Daily and Weekly intervals have been provided.

4. Add a .csv file to a data bucket with all the contacts you would like to receive the emails created by this function. There is an example of the format of the .csv file in the email_csv folder titled "contacts.csv". Indicate which bucket has the .csv file and the name of that file in the .env file in the email_csv folder. 

## Looker Studio

After creating the required BigQuery Tables and importing data, users may visualize their data using the provided [Looker Studio Template](https://lookerstudio.google.com/u/3/reporting/ed16a23b-d713-455c-9556-bacdf57b7021/page/p_rj30zb6j3c/preview).

The report includes the following pre-made charts:
1) List of Tests That Do Not Meet 25/3 Mbps Standard (By Date)
2) Chart of Upload/Download Speeds (By IP and Date)


<!-- ## Appendix -->


## Credits
---
- Jim Cupples at PAgCASA for providing the opportunity for a couple of Oregon State University Students to contribute to open-source broadband data measurement. https://www.pagcasa.org/
- M-Lab's Murakami Testing Tool available at https://github.com/m-lab/murakami
