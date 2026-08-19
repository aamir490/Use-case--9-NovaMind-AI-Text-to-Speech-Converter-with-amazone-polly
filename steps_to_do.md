# Serverless Text-to-Speech Application with Amazon Polly
## Complete Deployment Guide

---

## Architecture Overview

```
User (Browser)
    │
    ▼
index.html  ──►  API Gateway (PostReaderAPI)
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
   POST /  (new text)        GET /?postId=*
              │                    │
              ▼                    ▼
   Lambda: PostReader_NewPost   Lambda: PostReader_GetPost
              │                    │
              ▼                    ▼
         DynamoDB ◄──────────── DynamoDB
         (AudioPost)           (AudioPost)
              │
              ▼ (SNS publish)
         SNS Topic: audiopostsss
              │
              ▼ (trigger)
   Lambda: ConvertToAudio
              │
         Amazon Polly
              │
              ▼
         S3 Bucket (audio .mp3 files)
```

**Services Used:** DynamoDB · S3 · SNS · Lambda (×3) · IAM · API Gateway · Amazon Polly

---

## Step 1 — Create DynamoDB Table

1. Go to **AWS Console → DynamoDB → Create table**
2. Set the following:
   - **Table name:** `posts`
   - **Partition key:** `id` (type: String)
   - **Table settings:** Default settings
3. Click **Create table**

> **Table schema** (auto-populated by the app):
> | Attribute | Description |
> |-----------|-------------|
> | `id`      | UUID — unique identifier for each audio post |
> | `text`    | The input text submitted by the user |
> | `voice`   | Amazon Polly voice used (e.g. `Joanna`) |
> | `status`  | `PROCESSING` → `UPDATED` after MP3 is generated |
> | `url`     | Public S3 URL of the generated `.mp3` file |

---

## Step 2 — Create S3 Bucket for Audio Files

1. Go to **AWS Console → S3 → Create bucket**
2. Set the following:
   - **Bucket name:** `audiopostssss` (or your preferred unique name, e.g. `ml-polly-gen-ai`)
   - **Region:** Choose your preferred region (e.g. `us-east-1`)
   - **Object Ownership:** ACLs enabled
   - **Block Public Access:** Uncheck "Block all public access"
   - Acknowledge the public access warning
3. Click **Create bucket**

> This bucket stores all generated `.mp3` audio files. The `ConvertToAudio` Lambda uploads files here with `public-read` ACL.

---

## Step 3 — Create SNS Topic

1. Go to **AWS Console → SNS → Topics → Create topic**
2. Set the following:
   - **Type:** Standard
   - **Name:** `audiopostssss`
   - **Display name:** `New posts`
3. Click **Create topic**
4. **Copy the Topic ARN** — you will need it for Lambda environment variables in Step 6. 
 ARN - `arn:aws:sns:us-east-1:472360887517:audiopostssss`

> The `PostReader_NewPost` Lambda publishes the new record's UUID to this topic. SNS then triggers `ConvertToAudio` to process the text asynchronously.

---

## Step 4 — Create IAM Role for Lambda

1. Go to **AWS Console → IAM → Roles → Create role**
2. **Trusted entity:** AWS service → Lambda
3. **Role name:** `cloudage-lambda-roleee`
4. Attach an **inline policy** with the following JSON (update the ARNs to match your account ID and region):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": ["polly:SynthesizeSpeech"],
            "Resource": ["*"],
            "Effect": "Allow"
        },
        {
            "Action": [
                "dynamodb:Query",
                "dynamodb:Scan",
                "dynamodb:PutItem",
                "dynamodb:UpdateItem"
            ],
            "Resource": [
                "arn:aws:dynamodb:<REGION>:<ACCOUNT_ID>:table/posts"
            ],
            "Effect": "Allow"
        },
        {
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:<REGION>:<ACCOUNT_ID>:log-group:/aws/lambda/PostReader_*",
                "arn:aws:logs:<REGION>:<ACCOUNT_ID>:log-group:/aws/lambda/ConvertToAudio*"
            ],
            "Effect": "Allow"
        },
        {
            "Action": ["sns:Publish"],
            "Resource": [
                "arn:aws:sns:<REGION>:<ACCOUNT_ID>:audiopostsss"
            ],
            "Effect": "Allow"
        },
        {
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::audiopostsss",
                "arn:aws:s3:::audiopostsss/*"
            ],
            "Effect": "Allow"
        }
    ]
}
```

> Replace `<REGION>` and `<ACCOUNT_ID>` with your actual AWS region and 12-digit account ID.

---

## Step 5 — Lambda Function 1: PostReader_NewPost

### Create the Function

1. Go to **AWS Console → Lambda → Create function → Author from scratch**
2. Set the following:
   - **Function name:** `PostReader_NewPost`
   - **Runtime:** Python 3.13
   - **Execution role:** Use an existing role → `cloudage-lambda-roleee`
3. Click **Create function**

### Deploy the Code

Replace the default code with the contents of `PostReader_NewPost/PostReader_NewPost.py`:

```python
import boto3
import os
import uuid

def lambda_handler(event, context):

    recordId = str(uuid.uuid4())
    voice = event["voice"]
    text = event["text"]

    print('Generating new DynamoDB record, with ID: ' + recordId)
    print('Input Text: ' + text)
    print('Selected voice: ' + voice)

    # Creating new record in DynamoDB table
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['DB_TABLE_NAME'])
    table.put_item(
        Item={
            'id' : recordId,
            'text' : text,
            'voice' : voice,
            'status' : 'PROCESSING'
        }
    )

    # Sending notification about new post to SNS
    client = boto3.client('sns')
    client.publish(
        TopicArn = os.environ['SNS_TOPIC'],
        Message = recordId
    )

    return recordId
```

Click **Deploy** (or press `Ctrl+Shift+U`).

### Configure Environment Variables

Go to **Configuration → Environment variables → Edit** and add:

| Key | Value |
|-----|-------|
| `SNS_TOPIC` | Paste your SNS Topic ARN (from Step 3) |
| `DB_TABLE_NAME` | `posts` |

Click **Save**.

### Set Timeout

Go to **Configuration → General configuration → Edit** → set **Timeout** to `10 seconds` → Save.

### Test the Function

1. Go to the **Test** tab
2. **Event name:** `Joanna`
3. Replace the event body with:
```json
{
  "voice": "Joanna",
  "text": "This is working!"
}
```
4. Click **Test** — verify a successful execution log and a UUID return value.

---

## Step 6 — Lambda Function 2: ConvertToAudio

### Create the Function

1. Go to **AWS Console → Lambda → Create function → Author from scratch**
2. Set the following:
   - **Function name:** `ConvertToAudio`
   - **Runtime:** Python 3.13
   - **Execution role:** Use an existing role → `cloudage-lambda-roleee`
3. Click **Create function**

### Deploy the Code

Replace the default code with the contents of `ConvertToAudio/lambda_function.py`:

```python
import boto3
import os
from contextlib import closing
from boto3.dynamodb.conditions import Key, Attr

def lambda_handler(event, context):

    postId = event["Records"][0]["Sns"]["Message"]

    print ("Text to Speech function. Post ID in DynamoDB: " + postId)

    # Retrieving information about the post from DynamoDB table
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['DB_TABLE_NAME'])
    postItem = table.query(
        KeyConditionExpression=Key('id').eq(postId)
    )

    text = postItem["Items"][0]["text"]
    voice = postItem["Items"][0]["voice"]

    rest = text

    # Split text into ~2500-char blocks (Polly limit is ~3000 chars per call)
    textBlocks = []
    while (len(rest) > 2600):
        begin = 0
        end = rest.find(".", 2500)
        if (end == -1):
            end = rest.find(" ", 2500)
        textBlock = rest[begin:end]
        rest = rest[end:]
        textBlocks.append(textBlock)
    textBlocks.append(rest)

    # Call Polly for each block and combine into a single MP3
    polly = boto3.client('polly')
    for textBlock in textBlocks:
        response = polly.synthesize_speech(
            OutputFormat='mp3',
            Text = textBlock,
            VoiceId = voice
        )
        if "AudioStream" in response:
            with closing(response["AudioStream"]) as stream:
                output = os.path.join("/tmp/", postId)
                with open(output, "wb") as file:
                    file.write(stream.read())

    # Upload MP3 to S3 with public-read ACL
    s3 = boto3.client('s3')
    s3.upload_file('/tmp/' + postId, os.environ['BUCKET_NAME'], postId + ".mp3")
    s3.put_object_acl(ACL='public-read', Bucket=os.environ['BUCKET_NAME'], Key=postId + ".mp3")

    # Build the public URL
    location = s3.get_bucket_location(Bucket=os.environ['BUCKET_NAME'])
    region = location['LocationConstraint']
    if region is None:
        url_beginning = "https://s3.amazonaws.com/"
    else:
        url_beginning = "https://s3-" + str(region) + ".amazonaws.com/"

    url = url_beginning + str(os.environ['BUCKET_NAME']) + "/" + str(postId) + ".mp3"

    # Update DynamoDB record: status → UPDATED, url → S3 URL
    response = table.update_item(
        Key={'id': postId},
        UpdateExpression="SET #statusAtt = :statusValue, #urlAtt = :urlValue",
        ExpressionAttributeValues={':statusValue': 'UPDATED', ':urlValue': url},
        ExpressionAttributeNames={'#statusAtt': 'status', '#urlAtt': 'url'},
    )

    return
```

Click **Deploy**.

### Configure Environment Variables

Go to **Configuration → Environment variables → Edit** and add:

| Key | Value |
|-----|-------|
| `DB_TABLE_NAME` | `posts` |
| `BUCKET_NAME` | Your audio S3 bucket name (e.g. `audiopostsss`) |

Click **Save**.

### Set Timeout

Go to **Configuration → General configuration → Edit** → set **Timeout** to `5 minutes` → Save.

### Add SNS Trigger

1. Go to the **Triggers** section → click **Add trigger**
2. **Source:** SNS
3. **SNS topic:** Select `audiopostssss`
4. Click **Add**

> This wires SNS → Lambda automatically. When `PostReader_NewPost` publishes a record ID to SNS, this function is invoked to generate and upload the MP3.

---

## Step 7 — Lambda Function 3: PostReader_GetPost

### Create the Function

1. Go to **AWS Console → Lambda → Create function → Author from scratch**
2. Set the following:
   - **Function name:** `PostReader_GetPost`
   - **Runtime:** Python 3.13
   - **Execution role:** Use an existing role → `CloudAge-Lambda-Role`
3. Click **Create function**

### Deploy the Code

Replace the default code with the contents of `get_audio_post/lambda_function.py`:

```python
import boto3
import os
from boto3.dynamodb.conditions import Key, Attr

def lambda_handler(event, context):

    postId = event["postId"]

    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['DB_TABLE_NAME'])

    if postId == "*":
        items = table.scan()      # Return ALL posts
    else:
        items = table.query(
            KeyConditionExpression=Key('id').eq(postId)
        )

    return items["Items"]
```

Click **Deploy**.

### Configure Environment Variables

Go to **Configuration → Environment variables → Edit** and add:

| Key | Value |
|-----|-------|
| `DB_TABLE_NAME` | `posts` |

Click **Save**.

### Test the Function

1. Go to the **Test** tab
2. **Event name:** `AllPosts`
3. Replace the event body with:
```json
{
  "postId": "*"
}
```
4. Click **Test** — verify the execution returns a list (may be empty if no posts yet, or contain your test post from Step 5).

---

## Step 8 — Create REST API (API Gateway)

### Create the API

1. Go to **AWS Console → API Gateway → Create API → REST API → Build**
2. Set the following:
   - **API name:** `PostReaderAPI`
   - **Endpoint Type:** Regional
3. Click **Create API**

### Create POST Method (new posts)

1. In the **Resources** pane, select `/`
2. Click **Create method**
3. Set the following:
   - **Method type:** POST
   - **Integration type:** Lambda Function
   - **Lambda function:** `PostReader_NewPost`
4. Click **Create method**

### Create GET Method (retrieve posts)

1. In the **Resources** pane, select `/`
2. Click **Create method**
3. Set the following:
   - **Method type:** GET
   - **Integration type:** Lambda Function
   - **Lambda function:** `PostReader_GetPost`
4. Click **Create method**

### Enable CORS

1. In the **Resources** pane, select `/`
2. Click **Enable CORS**
3. Configure:
   - **Gateway responses:** Check `Default 4XX` and `Default 5XX`
   - **Access-Control-Allow-Methods:** Check `GET` and `POST`
4. Click **Save**

### Configure GET Query Parameter

1. Select the **GET** method
2. Go to **Method request settings → Edit**
3. Expand **URL query string parameters**
4. Click **Add query string** → name it `postId`
5. Click **Save**

### Configure GET Integration Request Mapping

The GET method needs to map the `postId` query parameter into the JSON body that `PostReader_GetPost` expects.

1. Select the **GET** method → click the **Integration request** tab
2. Click **Edit**
3. Set **Request body passthrough:** `When there are no templates defined (recommended)`
4. Expand **Mapping templates** → click **Add mapping template**
5. Set **Content type:** `application/json`
6. Set **Template body:**
```json
{
    "postId" : "$input.params('postId')"
}
```
7. Click **Save**

### Deploy the API

1. Click **Deploy API**
2. **Stage:** Create a new stage, name it `prod`
3. Click **Deploy**
4. **Copy the Invoke URL** — it will look like:
   `https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod`
   `https://r9wqe87zei.execute-api.us-east-1.amazonaws.com/prodd/`

> You will paste this URL into `index.html` in Step 10.

---

## Step 9 — Create S3 Bucket for Website Hosting

1. Go to **AWS Console → S3 → Create bucket**
2. Set the following:
   - **Bucket name:** Choose a unique name (e.g. `www-audioposts-frontend`)
   - **Object Ownership:** ACLs enabled
   - **Block Public Access:** Uncheck "Block all public access"
3. Click **Create bucket**
4. Open the bucket → go to the **Permissions** tab → **Bucket policy → Edit**
5. Paste the following policy (replace the bucket name): `arn:aws:s3:::audiopostssss`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::YOUR-FRONTEND-BUCKET-NAME/*"]
        }
    ]
}
```

6. Click **Save changes**

---

## Step 10 — Configure and Upload the Frontend

### Update the API Endpoint in index.html

1. Open `Load_On_S3_or_ELB/index.html`
2. Find the input field with `id="apiEndpoint"` — the user enters the URL via the UI, so no hardcoding is needed. However, you can pre-fill the `value=""` attribute with your Invoke URL for convenience:

```html
<input type="text" id="apiEndpoint"
  value="https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod"
  placeholder="Enter your API endpoint">
```

### Upload Files to the Frontend Bucket

Upload both files to the website S3 bucket:

- `Load_On_S3_or_ELB/index.html`
- `Load_On_S3_or_ELB/CloudAge-Logo.png`

### Enable Static Website Hosting

1. Go to the frontend bucket → **Properties** tab
2. Scroll to **Static website hosting → Edit**
3. Set:
   - **Static website hosting:** Enable
   - **Index document:** `index.html`
   - **Error document:** `index.html`
4. Click **Save changes**
5. Note the **Bucket website endpoint URL** displayed at the bottom of the Static website hosting section.

---

## Step 11 — Test the Full Application

1. Open the **S3 website endpoint URL** in a browser
2. Enter your **API Gateway Invoke URL** in the endpoint field
3. Select a voice (e.g. **Joanna [English]**)
4. In the **Post ID** field enter `*` and click **Search** — this retrieves all existing posts
5. Type something in the text area and click **Say it!**
   - A Post ID will appear confirming the record was created with status `PROCESSING`
   - After a few seconds, search again — the status should be `UPDATED` and an audio player will appear
6. Click play on the audio player to hear your text spoken aloud

---

## Flow Summary

```
[User types text] → POST to API Gateway
                            ↓
                  PostReader_NewPost Lambda
                  • Saves record to DynamoDB (status: PROCESSING)
                  • Publishes record ID to SNS
                            ↓
                  SNS triggers ConvertToAudio Lambda
                  • Fetches text + voice from DynamoDB
                  • Calls Amazon Polly → generates MP3
                  • Uploads MP3 to S3 (public-read)
                  • Updates DynamoDB record (status: UPDATED, url: S3 URL)
                            ↓
             [User searches by Post ID] → GET to API Gateway
                            ↓
                  PostReader_GetPost Lambda
                  • Queries DynamoDB
                  • Returns item(s) including MP3 URL
                            ↓
                  [Browser renders audio player with S3 URL]
```

---

## Environment Variables Reference

| Lambda | Key | Value |
|--------|-----|-------|
| `PostReader_NewPost` | `SNS_TOPIC` | SNS Topic ARN |
| `PostReader_NewPost` | `DB_TABLE_NAME` | `posts` |
| `ConvertToAudio` | `DB_TABLE_NAME` | `posts` |
| `ConvertToAudio` | `BUCKET_NAME` | Your audio S3 bucket name |
| `PostReader_GetPost` | `DB_TABLE_NAME` | `posts` |

---

## Cleanup (AWS CLI)

When you are done, use these commands to tear down resources and avoid charges.

```bash
# Configure AWS CLI (if not already done)
aws configure

# Empty the audio S3 bucket
aws s3 rm s3://audiopostsss --recursive

# Verify bucket is empty
aws s3 ls s3://audiopostsss

# Delete the audio bucket
aws s3 rb s3://audiopostsss --force

# Empty and delete the frontend bucket
aws s3 rb s3://YOUR-FRONTEND-BUCKET-NAME --force

# List buckets in a specific region to verify cleanup
aws s3api list-buckets --query "Buckets[].Name" --output text | tr '\t' '\n' | \
while read bucket; do
    region=$(aws s3api get-bucket-location --bucket "$bucket" --query "LocationConstraint" --output text)
    if [ "$region" = "None" ] || [ "$region" = "us-east-1" ]; then
        echo "$bucket"
    fi
done
```

> Also remember to delete the Lambda functions, DynamoDB table, SNS topic, API Gateway, and IAM role from the AWS Console to fully clean up.

---

## Project File Reference

```
PollyGenAI/
├── API_GW/
│   └── PostReaderAPI-prod-swagger-apigateway.json   # Exported API spec
├── ConvertToAudio/
│   └── lambda_function.py                           # Lambda 2: Polly → S3
├── get_audio_post/
│   └── lambda_function.py                           # Lambda 3: Fetch from DynamoDB
├── Load_On_S3_or_ELB/
│   ├── index.html                                   # Frontend web app
│   └── CloudAge-Logo.png                            # Logo asset
├── PostReader_NewPost/
│   └── PostReader_NewPost.py                        # Lambda 1: Create post + SNS
└── ReadersAreTheLeaders/
    ├── Serverless Text-to-Speech Application...txt  # Original step guide
    ├── awsCLI.rtf                                   # AWS CLI cleanup commands
    └── Practice.png                                 # Architecture diagram
```
