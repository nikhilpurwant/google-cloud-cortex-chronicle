# User Guide - Palo Alto Corex XDR Event Forwarding to Google SecOps


### Introduction

This document provides detailed steps for a customer to achieve Cortex XDR Events / Telemetry forwarding to Google Security Operations (SIEM/Chronicle)
And this will be community supported, if there is any questions, please reach out to the community or contact: [google-tech@paloaltonetworks.com](mailto:google-tech@paloaltonetworks.com), or you can access the community page at [https://live.paloaltonetworks.com/t5/general-articles/palo-alto-cortex-xdr-event-forwarding-to-google-secops-chronicle/ta-p/600621](https://live.paloaltonetworks.com/t5/general-articles/palo-alto-cortex-xdr-event-forwarding-to-google-secops-chronicle/ta-p/600621)



### Solution



The telemetry data is pulled into an intermediary bucket in the customer tenant and the native integration is set up from there.



The following diagram demonstrates this.



![Image](./images/image16.png)


### Steps To Setup The Integration (with an example)

1. Create Required GCS Bucket

The customer creates a GCS bucket in the Customer/MDR tenant. Let’s call it Project1

For this example, we are using the following bucket (your bucket name will be different)

cortex-xdr-events-destination-09302024 - Used to hold the XDR telemetry data temporarily.

Make sure that the bucket is in the same region as the customer’s Chronicle Region.

   

![Image](./images/image18.png)

2. Set up Cortex XDR Event Forwarding 

Setup the Cortex XDR event forwarding (an [Add-on Feature](https://docs-cortex.paloaltonetworks.com/r/Cortex-XDR/Cortex-XDR-Pro-Administrator-Guide/Manage-Event-Forwarding) of Cortex XDR) and download the service account key. It will save as event_forwarding_credentials.json(name could be different in your case). For the complete guide to event forwarding please refer to [this link](https://docs-cortex.paloaltonetworks.com/r/Cortex-XDR/Cortex-XDR-Pro-Administrator-Guide/Parsing-Rules-Raw-Dataset). The following screenshot shows the customer performing this action. At the end of this step the customer must have 

- Storage Path (GCS Bucket URL)

- Service account JSON key.



![Image](./images/image4.png)





3. Secret Manager Setup

Create a secret called EVENT_FRWD_CRTX_KEY and add the contents of the SA

JSON event_forwarding_credentials.json as the value of the secret

   

![Image](./images/image23.png)

   

4. Set up Native Chronicle Feed Integration

- Create a new feed by navigating to SIEM Settings - Feeds - ADD NEW. 



![Image](./images/image1.png)

- Provide a feed name and select options as shown below. Click GET A SERVICE ACCOUNT. Click Next.



![Image](./images/image15.png)



- Provide the bucket name and select the options as shown below. Please add a namespace if that’s relevant to you/ your customer. It is recommended to add an ingestion label. Copy the service account name.



![Image](./images/image17.png)

- Review the details added and Submit



![Image](./images/image14.png)



- The feed should be available in feeds now (the feed name in the example below is different as we are using a feed created earlier)



![Image](./images/image11.png)



- For now, disable the feed. We will enable it later.



![Image](./images/image13.png)



![Image](./images/image9.png)



5. IAM Setup

The Cortex XDR service account created during the event forwarding setup already has access to the source bucket. Now go ahead and provide the Storage Object Admin and Storage Legacy Bucket Reader access to this service account on the bucket created in Step 1.

- You can get the service account name by:


```bash
CORTEX_EMAIL=$(jq -r '.client_email' <(gcloud secrets versions access latest --secret=EVENT_FRWD_CRTX_KEY))
CHRONICLE_BUCKET="cortex-xdr-events-destination-09302024" #the GCS bucket you created in the beginning
```

- Grant the permissions through gcloud:

      
```bash
gcloud storage buckets add-iam-policy-binding gs://$CHRONICLE_BUCKET \
    --member=serviceAccount:$CORTEX_EMAIL \
    --role='roles/storage.objectAdmin'


gcloud storage buckets add-iam-policy-binding gs://$CHRONICLE_BUCKET \
    --member=serviceAccount:$CORTEX_EMAIL \
    --role='roles/storage.legacyBucketReader'
    
```

- Also, provide the chronicle service  account created during feed creation of the Storage Object Viewer (roles/storage.objectViewer), use the below gcloud command, and replace the <chronicle-service-account> with the value of the service account name you got from the Chronicle feed creation.


```bash
gcloud storage buckets add-iam-policy-binding $CHRONICLE_BUCKET \

    --member='serviceAccount:<chronicle-service-account>' \

    --role='roles/storage.objectViewer'
```

### Set up the Solution (One-time setup)

1. Enable Following APIs

- Cloud Run

- Artifact Registry

2. As the project owner Open Cloud Shell and download the code using 

```bash
git clone https://github.com/PaloAltoNetworks/google-cloud-cortex-chronicle.git
```
```bash
cd google-cloud-cortex-chronicle/
```
The contents of this directory are shown below



![Image](./images/image7.png)

3. Open the file env.properties with the editor of your choice. Update the values of the variables as shown below. Job Schedule minutes can be adjusted based on the size and frequency of data pushed by Cortex.




```shell
REGION=<your region> # update this to the region you want

REPO_NAME=panw-chronicle # The repo name to create

IMAGE_NAME=sync_cortex_bucket # The image name to create

GCP_PROJECT_ID=xdrxxxxxtion # update this to your project ID

JOB_NAME=cloud-run-job-cortex-data-sync # The Cloud Job name to create

PROJECT_NUMBER=80xxxxx9 # update this to your project number

# JOB ENV VARIABLES

SRC_BUCKET=xdr-us-xxxxx-event-forwarding # update this to Cortex XDR GCS bucket

DEST_BUCKET=cortex-xdr-events-destination # Update to the GCS name you created

SECRET_NAME=EVENT_FRWD_CRTX_KEY # Need to match exactly the secret you created

JOB_SCHEDULE_MINS=30

```

4. Provide execute permissions to the deploy.sh using the command 

```bash
chmod 744 deploy.sh
```
5. Run the deploy.sh using command 
```bash
./deploy.sh
```
This steps does following

- Creates an artifact registry repository

- Build an image for a cloud run job

- Pushes the image to the Artifact registry

- Creates a Cloud Run Job using this image

- Creates a trigger for this cloud run job every JOB_SCHEDULE_MINS minutes (configured in env.properties)



6. After the script finishes, you would need to grant permission to access the Secret Manager Secret you created before to the service account (you can see the service account used by the Cloud Jobs from the script output; see below).



![Image](./images/image2.png)

- Grant permission through the Secret Manager -> Permissions (Secret Manager Secret Accessor):



![Image](./images/image6.png)

### Verify setup


- Verify if artifacts mentioned in 5.1 (above step) are created.

- You can wait for JOB_SCHEDULE_MINS minutes or perform the following steps to force execute the job. This is required/recommended to be done only once to test.

- Go to the Cloud run job. In your case, it might show “no executions” in the status.



![Image](./images/image25.png)

- Force Execute



![Image](./images/image28.png)

- Check logs



![Image](./images/image27.png)

- Now check the destination bucket

It should have the files downloaded from XDR Bucket



![Image](./images/image26.png)

- Download one of the files and unzip it and note down one or more event ids as shown below



![Image](./images/image12.png)





- Now goto Chronicle - SIEM Settings - [Your Feed Name] -> Enable Feed



![Image](./images/image8.png)



![Image](./images/image3.png)

- Search for the event id in Chronicle with RAW search / UDM search (you may have to wait for a few minutes for UDM search). You should find the event in the Chronicle.



![Image](./images/image5.png)

### Regular Usage & Monitoring

You do not have to change anything going forward for this integration.

The feed should remain enabled from this point forward unless you want to troubleshoot.

- You can change the schedule based on your requirement. Recommended 30 minutes initially when there’s too much data in the source bucket and gradually reducing to about 5 minutes as things stabilize. You can change that directly on the trigger as shown below



![Image](./images/image20.png)

- Monitor the Job Run execution History in Cloud run.



![Image](./images/image24.png)

### Cloud Billing Costs

Predominantly this solution uses the following resources that are billed.



- GCS cloud bucket

- Cloud Run Batch job

- Artifact Registry

Assuming deployment region to be us-central1 and about 10GB of data persistently stored every day and a cloud run job that runs every 10 minutes for 30 days - here is the cost estimate (It’s about USD 10 per month).

These are approximate costs. Your costs may vary based on the amount of data and the frequency of the job.

### Troubleshooting

1. If the data is not available in the Chronicle

- Check if the feed is enabled

- Check the cloud run job logs

- For the very first time the job may have to deal with GBs of data if you had the forwarding setup enabled for many days. Please check the job logs and wait for it to finish.

- Wait for a few minutes if the event is from a recent file

- Sometimes Cortex sends older files, so try to expand the search time range by a few hours.

2. Troubleshooting the Cloud run job

- Check for logs for any errors

- If there are no files logged by the job please check with Cortex XDR support as the files may not be present in the source bucket.





### How to reduce GCS costs?

You can set up the Lifecycle Management rule to delete the object after 14 days. Follow the steps shown in the screenshots below



![Image](./images/image21.png)



![Image](./images/image19.png)



![Image](./images/image22.png)





![Image](./images/image10.png)











### Uninstallation



Go to the installation folder, provide execute permissions to the uninstall.sh, and execute it by using the command 

```bash
chmod 744 uninstall.sh && ./uninstall.sh
```
> [!NOTE]
> Please be aware that you need to manually delete the GCS bucket created for temp telemetry storage in the very beginning (cortex-xdr-events-destination-09302024 in this example), and also need to manually delete the Secret Manager secret (EVENT_FRWD_CRTX_KEY in this example)

