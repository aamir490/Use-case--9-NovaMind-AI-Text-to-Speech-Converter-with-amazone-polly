# 🎙️ NovaMind AI — Text-to-Speech Converter
### Powered by Amazon Polly | Built on AWS Serverless Architecture

> Convert any text into natural-sounding MP3 audio using Amazon Polly, delivered through a fully serverless AWS backend and a clean web frontend hosted on S3.

🔗 **Live App:** http://audiopostssss.s3-website-us-east-1.amazonaws.com/

Built by **Aamir** | [LinkedIn](https://www.linkedin.com/in/aamir-imran)

---

## 🖥️ Application Screenshots

### Dashboard — Main Interface
![NovaMind AI Dashboard](project-pic/NovaMind%20Ai%20Text%20to%20Speech%20Dashboard1.png)

### Submit Text & Generate Audio
![NovaMind AI Submit](project-pic/NovaMind%20Ai%20Text%20to%20Speech%20Dashboard2.png)

### Audio Player — Playback Result
![NovaMind AI Playback](project-pic/NovaMind%20Ai%20Text%20to%20Speech%20Dashboard3.png)

---

## 📌 Project Overview

**NovaMind AI Text-to-Speech Converter** is a cloud-native, serverless application that transforms written text into spoken audio using **Amazon Polly**. Users submit text through a web interface, select a voice, and receive a playable MP3 audio file — all processed asynchronously in the cloud with zero server management.

---

## 🏗️ Architecture

```
Browser (S3 Static Website)
        │
        ▼
   API Gateway (REST API)
   ┌────────────────────┐
   │  POST /  →  GET /  │
   └────────────────────┘
        │                 │
        ▼                 ▼
Lambda: NewPost     Lambda: GetPost
        │                 │
        ▼                 ▼
    DynamoDB ◄──────── DynamoDB
    (posts)            (posts)
        │
        ▼
    SNS Topic
        │
        ▼
Lambda: ConvertToAudio
        │
        ▼
  Amazon Polly
        │
        ▼
    S3 Bucket
   (MP3 files)
```

---

## ☁️ AWS Services Used

| Service | Purpose |
|---|---|
| **Amazon Polly** | Converts text to MP3 speech |
| **AWS Lambda** | Serverless compute (3 functions) |
| **Amazon DynamoDB** | Stores post metadata and audio URLs |
| **Amazon SNS** | Async trigger between Lambda functions |
| **Amazon S3** | Stores MP3 files + hosts the web frontend |
| **API Gateway** | REST API exposing POST and GET endpoints |
| **IAM** | Role-based permissions for Lambda |

---

## 📸 AWS Console Screenshots

### Lambda Functions (3 Functions)
![AWS Lambda Functions](project-pic/aws-3-lambda-functions.png)

### API Gateway Configuration
![AWS API Gateway](project-pic/aws-api-gateway.png)

### DynamoDB Table
![AWS DynamoDB](project-pic/aws-dynamodb.png)

### SNS Topic
![AWS SNS Topic](project-pic/aws-sns.png)

### Frontend S3 Bucket
![AWS Frontend S3 Bucket](project-pic/aws-fronted-s3-bucket.png)

---

## ⚙️ Lambda Functions

### 1. `PostReader_NewPost`
- Triggered by: API Gateway `POST /`
- Accepts `{ "voice": "Joanna", "text": "Hello world" }`
- Creates a DynamoDB record with status `PROCESSING`
- Publishes the record ID to SNS topic
- Returns the unique post ID

### 2. `ConvertToAudio`
- Triggered by: SNS topic
- Reads the post from DynamoDB
- Splits long text into ~2500-char blocks (Polly limit)
- Calls Amazon Polly to synthesize MP3
- Uploads MP3 to S3 with `public-read` ACL
- Updates DynamoDB record: status → `UPDATED`, url → S3 URL

### 3. `PostReader_GetPost`
- Triggered by: API Gateway `GET /?postId=`
- Accepts `postId` — use `*` to retrieve all posts
- Queries DynamoDB and returns item(s)

---

## 🗄️ DynamoDB Schema

**Table name:** `posts`  
**Partition key:** `id` (String)

| Attribute | Type | Description |
|---|---|---|
| `id` | String | UUID — unique post identifier |
| `text` | String | Input text submitted by user |
| `voice` | String | Polly voice (e.g. `Joanna`) |
| `status` | String | `PROCESSING` or `UPDATED` |
| `url` | String | Public S3 URL of the MP3 file |

---

## 🌐 Frontend Features

- Select from 15 Amazon Polly voices across 10+ languages
- Submit text and receive a post ID instantly
- Search by post ID (or `*` for all posts)
- Built-in HTML5 audio player for each generated clip
- Character counter on text input
- Responsive design — works on mobile and desktop
- API endpoint pre-configured — works instantly on open

---

## 📁 Project Structure

```
PollyGenAI/
├── API_GW/
│   └── PostReaderAPI-prod-swagger-apigateway.json   # Exported API spec
├── ConvertToAudio/
│   └── lambda_function.py                           # Lambda 2: Polly → S3
├── get_audio_post/
│   └── lambda_function.py                           # Lambda 3: Read DynamoDB
├── Load_On_S3_or_ELB/
│   ├── index.html                                   # Frontend web app
│   └── mylogoo.png                                  # NovaMind AI logo
├── PostReader_NewPost/
│   └── PostReader_NewPost.py                        # Lambda 1: Create post
├── project-pic/                                     # Screenshots
│   ├── NovaMind Ai Text to Speech Dashboard1.png
│   ├── NovaMind Ai Text to Speech Dashboard2.png
│   ├── NovaMind Ai Text to Speech Dashboard3.png
│   ├── aws-3-lambda-functions.png
│   ├── aws-api-gateway.png
│   ├── aws-dynamodb.png
│   ├── aws-sns.png
│   └── aws-fronted-s3-bucket.png
├── ReadersAreTheLeaders/
│   ├── Serverless Text-to-Speech...txt              # Original step guide
│   └── awsCLI.rtf                                   # AWS CLI cleanup commands
├── README.md                                        # This file
├── interview.md                                     # Interview preparation guide
├── github.txt                                       # Git commands reference
└── steps_to_do.md                                   # Full deployment guide
```

---

## 🚀 Deployment Summary

> Full step-by-step instructions are in [`steps_to_do.md`](./steps_to_do.md)

1. Create DynamoDB table `posts` (partition key: `id`)
2. Create S3 bucket for audio files (ACLs enabled, public access on)
3. Create SNS Standard topic
4. Create IAM role with Polly, DynamoDB, SNS, S3, CloudWatch permissions
5. Deploy Lambda 1: `PostReader_NewPost` — set env vars `SNS_TOPIC` + `DB_TABLE_NAME`
6. Deploy Lambda 2: `ConvertToAudio` — set env vars `DB_TABLE_NAME` + `BUCKET_NAME`, add SNS trigger
7. Deploy Lambda 3: `PostReader_GetPost` — set env var `DB_TABLE_NAME`
8. Create API Gateway REST API with POST and GET methods + CORS + mapping template
9. Create frontend S3 bucket with static website hosting + bucket policy
10. Upload `index.html` + `mylogoo.png` → test the live app

---

## 🔐 IAM Permissions Required

The Lambda execution role needs:
- `polly:SynthesizeSpeech`
- `dynamodb:Query`, `Scan`, `PutItem`, `UpdateItem`
- `sns:Publish`
- `s3:PutObject`, `PutObjectAcl`, `GetBucketLocation`
- `logs:CreateLogGroup`, `CreateLogStream`, `PutLogEvents`

---

## 🧪 Testing

**Test Lambda 1 (create post):**
```json
{
  "voice": "Joanna",
  "text": "Hello, this is NovaMind AI speaking."
}
```

**Test Lambda 3 (get all posts):**
```json
{
  "postId": "*"
}
```

**Test via API Gateway:**
```bash
# Create a new post
curl -X POST https://<api-id>.execute-api.us-east-1.amazonaws.com/prod \
  -H "Content-Type: application/json" \
  -d '{"voice":"Joanna","text":"Hello from NovaMind AI"}'

# Retrieve all posts
curl "https://<api-id>.execute-api.us-east-1.amazonaws.com/prod?postId=*"
```

---

## 🧹 Cleanup

```bash
# Empty and delete the audio bucket
aws s3 rb s3://audiopostssss-aamir --force

# Empty and delete the frontend bucket
aws s3 rb s3://www-audioposts-frontend --force
```

Also delete manually: Lambda functions, DynamoDB table, SNS topic, API Gateway, IAM role.

---

## 👤 Author

**Aamir**  
Cloud & AI Engineer  
🔗 [linkedin.com/in/aamir-imran](https://www.linkedin.com/in/aamir-imran)

---

## 📄 License

This project is for educational and portfolio purposes.
