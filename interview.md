# 🎤 Interview Preparation Guide
## NovaMind AI — Serverless Text-to-Speech Converter with Amazon Polly

**Prepared by: Aamir**
🔗 [linkedin.com/in/aamir-imran](https://www.linkedin.com/in/aamir-imran)

---

## 🏗️ Architecture Diagram

![NovaMind AI Architecture](project-pic/architecture.jpg)

---

## 🚀 Project Overview

**NovaMind AI** is a production-ready, production-grade Serverless Generative AI Text-to-Speech (TTS) Web Application. It leverages Amazon Web Services (AWS) managed services to convert user-submitted text into highly natural, human-like audio files in real time.

Because the backend is entirely serverless, it requires zero server management, automatically scales from zero to millions of requests, and charges strictly on a pay-per-use basis.

---

## 🛠️ Core Technology Stack

- **Frontend:** HTML5, CSS3, and JavaScript hosted as a static site on Amazon S3
- **API Management:** Amazon API Gateway handles incoming HTTP REST API requests with CORS enabled
- **Compute:** AWS Lambda executes the microservice backend business logic asynchronously
- **Database:** Amazon DynamoDB serves as a fast, NoSQL key-value store to track text post states and storage URLs
- **Event Orchestration:** Amazon SNS manages the decoupled asynchronous execution flow
- **Generative AI Engine:** Amazon Polly powers the high-fidelity neural text-to-speech synthesis
- **Storage:** Amazon S3 Buckets securely save the final generated audio outputs (`.mp3` format)
- **Security & Monitoring:** AWS IAM handles strict fine-grained execution roles, while Amazon CloudWatch acts as the central hub for logging, error tracking, and metrics

---

## 🔄 Step-by-Step Architecture Flow

The system operates using two distinct operational paths: the **Write Path** (Audio Creation) and the **Read Path** (Audio Retrieval).

### 1. The Frontend Dashboard
A user visits the NovaMind AI website. Through the user interface, they enter custom text, pick a specific target neural voice profile (e.g., standard vs. neural conversational, gender, language accent), and hit **"Generate Audio"**.

### 2. The Create Audio Path — Asynchronous Write Pipeline

- **API Submission:** The browser sends a `POST /` request containing the text payload to the PostReaderAPI (API Gateway)
- **Request Intake (PostReader_NewPost):** A lightweight Lambda function generates a unique UUID for the post, initializes a status record tracking it as `PROCESSING`, and saves this metadata to the DynamoDB `posts` table
- **Event Broadcast:** Before finishing, the Lambda function publishes the new post event to an Amazon SNS Topic. This decouples the client response from the heavy-lifting audio compilation, immediately freeing the API to accept new traffic
- **Audio Synthesis (ConvertToAudio):** Triggered by the SNS topic, a second worker Lambda function reads the full text entry. If the text is long, it breaks it into smaller chunks, passes it to the Amazon Polly API, receives raw high-quality MP3 streams back, assembles them, and writes the output file directly to the Amazon S3 Audio Bucket
- **State Update:** Once the audio upload is successful, it hits DynamoDB a second time to switch the post state to `UPDATED` and attach the public S3 object URL

### 3. The Retrieve Audio Path — Synchronous Read Pipeline

- While the write path runs in the background, the client dashboard fires a `GET /?postId=` request to the API Gateway
- The `PostReader_GetPost` Lambda checks the status of that specific `id` in DynamoDB
- If the status is still `PROCESSING`, it tells the UI to show a loader. Once it changes to `UPDATED`, it returns the final MP3 URL
- The browser's native media player then streams the audio file directly from S3

---

## 🧠 Why This Qualifies as Generative AI

Instead of relying on rigid, pre-recorded audio files or robotic phoneme splicing, the heart of this application — **Amazon Polly** — utilizes deep learning models to synthesize human-like speech. It dynamically generates entirely new audio content from raw textual sequences, adjusting for contextual nuances, punctuation pacing, accents, and inflection to simulate a natural human speaker.

---

## 1. PROJECT INTRODUCTION (Elevator Pitch)

> *"Tell me about a project you built on AWS."*

**What to say:**

"I built a fully serverless Text-to-Speech application on AWS called **NovaMind AI**. The application takes text input from a web interface, converts it to natural-sounding MP3 audio using Amazon Polly, stores the audio in S3, and makes it available for playback directly in the browser — all without managing a single server.

The entire backend is event-driven: API Gateway receives the request, Lambda processes it, SNS decouples the conversion step, and DynamoDB tracks the state. I also hosted the frontend as a static website on S3, making the whole architecture 100% serverless."

---

## 2. ARCHITECTURE EXPLANATION

> *"Walk me through your architecture."*

**Full data flow to explain:**

```
Step 1: User opens index.html (hosted on S3 Static Website)
Step 2: User types text, selects a voice, clicks "Say it!"
Step 3: Browser sends HTTP POST to API Gateway
Step 4: API Gateway triggers Lambda 1 (PostReader_NewPost)
Step 5: Lambda 1 saves a record to DynamoDB with status = PROCESSING
Step 6: Lambda 1 publishes the record ID to an SNS topic
Step 7: SNS triggers Lambda 2 (ConvertToAudio) asynchronously
Step 8: Lambda 2 fetches text and voice from DynamoDB
Step 9: Lambda 2 calls Amazon Polly → gets MP3 audio stream
Step 10: Lambda 2 uploads MP3 to S3 with public-read ACL
Step 11: Lambda 2 updates DynamoDB → status = UPDATED, url = S3 link
Step 12: User searches by Post ID via GET request to API Gateway
Step 13: API Gateway triggers Lambda 3 (PostReader_GetPost)
Step 14: Lambda 3 queries DynamoDB and returns the item with the MP3 URL
Step 15: Browser renders an HTML5 audio player with the S3 URL
```

**Key point to emphasize:**
> "The architecture is asynchronous. The user gets a Post ID immediately without waiting for Polly to finish. This is a classic decoupled design using SNS as a message broker between two Lambda functions."

---

## 3. WHY EACH SERVICE WAS CHOSEN

> *"Why did you use SNS between the two Lambda functions instead of calling them directly?"*

**Answer:**
"Direct Lambda-to-Lambda calls create tight coupling and can hit timeout limits for long text. SNS decouples the producer (NewPost) from the consumer (ConvertToAudio). This means:
- If ConvertToAudio fails, SNS can retry automatically
- The user gets an instant response with their Post ID
- The conversion happens in the background asynchronously
- It's more scalable — SNS can fan out to multiple subscribers later"

---

> *"Why DynamoDB and not RDS?"*

**Answer:**
"DynamoDB is a perfect fit here because:
- The data model is simple: each post is a flat JSON item with an ID as the key
- It's serverless and scales automatically with no capacity planning
- It integrates natively with Lambda via the boto3 SDK
- For this use case we don't need SQL joins or complex relational queries
- RDS would require VPC configuration, subnet groups, and ongoing instance management"

---

> *"Why host the frontend on S3 instead of EC2 or a web server?"*

**Answer:**
"S3 Static Website Hosting is ideal for a pure HTML/JS/CSS frontend:
- Zero server management
- Extremely low cost — you pay only per request and storage
- High availability built-in — S3 is replicated across multiple AZs
- Easy to update — just upload a new file
- Can be fronted by CloudFront for CDN caching globally"

---

> *"Why API Gateway instead of an Application Load Balancer?"*

**Answer:**
"API Gateway is the right tool for Lambda-backed REST APIs:
- Native Lambda integration with no extra networking setup
- Built-in CORS support
- Can handle request/response mapping templates
- Supports multiple stages (dev, staging, prod)
- Pay-per-request pricing — no idle cost like an ALB"

---

## 4. DEEP DIVE: LAMBDA FUNCTIONS

> *"Explain what each Lambda function does."*

### Lambda 1 — PostReader_NewPost
```
Input:  { "voice": "Joanna", "text": "Hello world" }
Action: 1. Generate a UUID as the record ID
        2. Write item to DynamoDB: id, text, voice, status=PROCESSING
        3. Publish the UUID to SNS topic
Output: The UUID (Post ID) returned to API Gateway → browser
```

**Why it's designed this way:**
- Keeps the user-facing response fast (no Polly call here)
- The record is created immediately so the user can check status
- SNS publish is near-instant

---

### Lambda 2 — ConvertToAudio
```
Input:  SNS event → { "Records": [{ "Sns": { "Message": "<uuid>" } }] }
Action: 1. Read post from DynamoDB using the UUID
        2. Split text into blocks of ~2500 chars (Polly max ~3000)
        3. For each block: call polly.synthesize_speech()
        4. Write audio stream to /tmp/<uuid> (Lambda ephemeral storage)
        5. Upload /tmp/<uuid> to S3 as <uuid>.mp3 with ACL public-read
        6. Build public URL: https://s3-<region>.amazonaws.com/<bucket>/<uuid>.mp3
        7. Update DynamoDB: status=UPDATED, url=<s3-url>
Output: None (void return)
```

**Important detail to mention:**
> "I handle long text by splitting it at sentence boundaries (`.`) first, then at word boundaries (` `) — this prevents Polly from cutting off mid-word and ensures natural audio flow."

---

### Lambda 3 — PostReader_GetPost
```
Input:  { "postId": "<uuid>" }  OR  { "postId": "*" }
Action: If postId == "*" → DynamoDB table.scan() (all records)
        Else             → DynamoDB table.query() (single record)
Output: Array of DynamoDB items with id, text, voice, status, url
```

**Important detail:**
> "Using `*` as a wildcard for scanning all posts was a deliberate UX decision. In production I would add pagination to the scan, since DynamoDB scan reads the entire table and can be expensive at scale."

---

## 5. DYNAMODB DESIGN

> *"How did you design your DynamoDB table?"*

**Answer:**
"The table is named `posts` with a single partition key `id` (a UUID string). There are no sort keys because each audio post is a standalone item — we never need to sort posts under the same partition.

The five attributes are:
- `id` — UUID generated by Python's `uuid.uuid4()`
- `text` — the original input text
- `voice` — the Polly voice ID (e.g. Joanna, Mizuki)
- `status` — `PROCESSING` when created, `UPDATED` after MP3 is ready
- `url` — the public S3 URL added after conversion

The status field acts as a simple state machine. The client polls by Post ID and knows the audio is ready when `status == UPDATED`."

---

## 6. SECURITY & IAM

> *"How did you secure the application?"*

**Answer:**
"I followed the principle of least privilege with a custom IAM role called `CloudAge-Lambda-Role`. Each permission is scoped to exactly what the Lambda functions need:

- `polly:SynthesizeSpeech` — allows calling Polly (no read/list/delete)
- `dynamodb:Query, Scan, PutItem, UpdateItem` — no DeleteItem or DropTable
- `sns:Publish` — publish only, no subscribe or delete
- `s3:PutObject, PutObjectAcl, GetBucketLocation` — write only, no read of other objects
- CloudWatch Logs — scoped to specific log group prefixes only

For the S3 audio bucket, I enabled ACLs and made uploaded objects `public-read` so the browser can stream the audio directly without needing pre-signed URLs. In a production system I would use CloudFront with signed URLs instead."

---

## 7. AMAZON POLLY KNOWLEDGE

> *"What is Amazon Polly and how does it work?"*

**Answer:**
"Amazon Polly is an AWS AI service that converts text to lifelike speech using deep learning. It supports:
- **Standard voices** — using concatenative synthesis
- **Neural voices** — using neural TTS for more natural-sounding output
- 30+ languages and 60+ voices
- Output formats: MP3, OGG, PCM
- SSML (Speech Synthesis Markup Language) for fine-tuned control over pronunciation, pace, and pitch

In this project I use the `synthesize_speech` API with `OutputFormat='mp3'` and pass the `VoiceId` selected by the user. The response contains an `AudioStream` which I write to Lambda's `/tmp` directory, then upload to S3."

---

> *"What is the text length limit for Polly and how did you handle it?"*

**Answer:**
"A single Polly `synthesize_speech` call can handle approximately 3,000 characters. To support longer texts, I implemented a text-splitting algorithm:

```python
while (len(rest) > 2600):
    end = rest.find(".", 2500)   # split at sentence boundary
    if end == -1:
        end = rest.find(" ", 2500)  # fallback: split at word boundary
    textBlock = rest[begin:end]
    rest = rest[end:]
    textBlocks.append(textBlock)
```

Each block is converted separately and the audio streams are appended to the same file in `/tmp`, producing a single seamless MP3."

---

## 8. API GATEWAY DESIGN

> *"How did you configure the API Gateway?"*

**Answer:**
"I created a Regional REST API called `PostReaderAPI` with two methods on the root resource `/`:

**POST /** — Accepts raw JSON body, passes directly to `PostReader_NewPost` Lambda.

**GET /?postId=** — Has a `postId` query string parameter. Because Lambda expects JSON input, I added an Integration Request Mapping Template:

```json
{
    "postId" : "$input.params('postId')"
}
```

This transforms the query string into a JSON object before it reaches Lambda.

I also enabled CORS on the resource, including `Default 4XX` and `Default 5XX` gateway responses, so browser fetch/AJAX calls aren't blocked by cross-origin policy."

---

## 9. ASYNC FLOW & EVENTUAL CONSISTENCY

> *"What happens if I search for a post immediately after creating it?"*

**Answer:**
"You'll see status = `PROCESSING` because Polly conversion is asynchronous. The flow is:

1. User clicks 'Say it!' → gets Post ID instantly (< 1 second)
2. SNS triggers ConvertToAudio in the background
3. Polly synthesizes the audio (takes 2–10 seconds depending on text length)
4. DynamoDB is updated to `UPDATED` with the S3 URL

When the user searches again a few seconds later, the audio player appears. This is **eventual consistency** — a trade-off we accept in serverless architectures to keep the user experience responsive. If real-time status was critical, I would implement WebSocket notifications via API Gateway WebSocket API or AWS AppSync subscriptions."

---

## 10. COST & SCALABILITY

> *"How much does this cost to run?"*

**Answer:**
"The application costs nearly nothing at low traffic because every component is pay-per-use:

| Service | Cost basis |
|---|---|
| Lambda | $0.20 per 1M requests + compute time |
| API Gateway | $3.50 per 1M API calls |
| DynamoDB | $0.25 per million read/write units (on-demand) |
| SNS | $0.50 per 1M notifications |
| Amazon Polly | $4.00 per 1M characters (standard) |
| S3 | ~$0.023/GB storage + $0.0004/1K requests |

For a demo or portfolio project the monthly cost is effectively $0 within AWS Free Tier limits. At scale, the architecture handles thousands of concurrent users automatically since Lambda and DynamoDB both auto-scale."

---

## 11. WHAT WOULD YOU IMPROVE IN PRODUCTION?

> *"What improvements would you make before putting this in production?"*

**Answer — list 5–6 strong points:**

1. **Authentication** — Add Amazon Cognito for user authentication so only authorized users can submit text and access their own posts.

2. **CloudFront CDN** — Put CloudFront in front of the S3 audio bucket. Use signed URLs instead of public-read ACL for secure audio delivery and global low-latency streaming.

3. **API Gateway Throttling** — Add usage plans and API keys to prevent abuse and control costs.

4. **DynamoDB Pagination** — The `scan` operation for `postId=*` reads the whole table. In production I'd implement pagination using `LastEvaluatedKey` and limit results per page.

5. **Error Handling & Dead Letter Queues** — Add an SQS DLQ to the SNS subscription so failed ConvertToAudio invocations are retried and failures are captured for alerting.

6. **Neural Voices** — Switch from standard to neural Amazon Polly voices for more natural-sounding speech, especially for long-form content.

7. **Infrastructure as Code** — Define the entire stack using AWS CDK or Terraform for repeatable, version-controlled deployments.

---

## 12. COMMON INTERVIEW QUESTIONS & ANSWERS

---

**Q: What is serverless computing?**

A: "Serverless means you write and deploy code without provisioning or managing servers. The cloud provider handles infrastructure, scaling, patching, and availability. You pay only for the compute time your code actually uses. AWS Lambda is the serverless compute service — it runs your function in response to an event and automatically scales to zero when idle."

---

**Q: What is event-driven architecture?**

A: "Event-driven architecture is a design pattern where services communicate by producing and consuming events rather than calling each other directly. In this project, the creation of a new post (an event) triggers a chain: Lambda publishes to SNS, SNS delivers the event to ConvertToAudio Lambda. Services are loosely coupled — they don't know about each other, only about the events they produce and consume."

---

**Q: What is the difference between SNS and SQS?**

A: "SNS (Simple Notification Service) is a pub/sub messaging service — it pushes messages to all subscribers immediately. SQS (Simple Queue Service) is a message queue — messages sit in the queue until a consumer pulls them. In this project I used SNS because I want the Lambda to be triggered immediately (push model). SQS would be better if I needed to process messages at a controlled rate or handle backpressure."

---

**Q: What is CORS and why did you need to configure it?**

A: "CORS (Cross-Origin Resource Sharing) is a browser security mechanism that blocks web pages from making requests to a different domain than the one that served the page. My frontend is hosted at an S3 domain and my API is at an API Gateway domain — different origins. Without CORS headers, the browser would block the AJAX calls. I configured API Gateway to return `Access-Control-Allow-Origin: *` on all responses, including 4XX/5XX error responses."

---

**Q: What is an IAM role and why does Lambda need one?**

A: "An IAM role is an AWS identity with specific permissions attached. Lambda functions run in AWS's infrastructure, so when they need to access other AWS services (DynamoDB, S3, SNS, Polly), they must be authorized. Instead of using hardcoded credentials, Lambda assumes an IAM role at runtime and gets temporary credentials automatically. I created `CloudAge-Lambda-Role` with only the exact permissions each function needs — following the principle of least privilege."

---

**Q: What is DynamoDB and when would you use it over RDS?**

A: "DynamoDB is AWS's fully managed NoSQL key-value and document database. I'd choose DynamoDB when:
- Data access patterns are simple (get by key, scan)
- I need automatic scaling with no capacity planning
- The workload is serverless and bursty
- Schema is flexible or evolving

I'd choose RDS when I need complex SQL queries, joins across tables, ACID transactions, or a relational data model."

---

**Q: What is Amazon Polly?**

A: "Amazon Polly is an AWS AI service in the Machine Learning category. It converts text to lifelike speech using deep learning models. It supports 60+ voices in 30+ languages, SSML for speech customization, and returns audio in MP3, OGG, or PCM format. It's a fully managed API — no ML knowledge needed to use it."

---

**Q: How does static website hosting on S3 work?**

A: "You enable the Static Website Hosting property on an S3 bucket, specify an index document (index.html), and make the bucket publicly readable via a bucket policy. S3 then serves the files directly over HTTP at a bucket website endpoint URL. It's not a web server — there's no PHP or server-side processing — just static file delivery. For HTTPS and a custom domain, you'd add CloudFront in front."

---

## 13. TECHNICAL SKILLS DEMONSTRATED BY THIS PROJECT

| Skill | Evidence |
|---|---|
| AWS Lambda | 3 functions, different triggers, env vars, timeouts |
| Amazon DynamoDB | Table design, PutItem, Query, Scan, UpdateItem |
| Amazon SNS | Topic creation, Lambda subscription, publish/trigger |
| Amazon S3 | Object upload, ACL, static website hosting, bucket policy |
| API Gateway | REST API, POST + GET, CORS, mapping templates, deployment |
| Amazon Polly | synthesize_speech, audio stream handling, text chunking |
| IAM | Custom role, least-privilege policy, trust relationships |
| Python (boto3) | All Lambda functions written in Python 3.13 |
| Async Architecture | Decoupled design with SNS message broker |
| Serverless Design | Zero server management, pay-per-use, auto-scaling |
| Frontend (HTML/JS) | Vanilla JS, AJAX/jQuery, dynamic table rendering, audio player |

---

## 14. PROJECT METRICS TO MENTION

- **3 Lambda functions** coordinating via event-driven triggers
- **5 AWS services** integrated in a single workflow
- **15 Polly voices** across 10+ languages available in the UI
- **0 servers** — 100% serverless architecture
- **< 1 second** user response time (async processing)
- **Unlimited text length** — handled via automatic text chunking
- **Cost: ~$0** at demo scale within AWS Free Tier

---

## 15. CLOSING STATEMENT

> *"Is there anything else you'd like to add about this project?"*

**What to say:**

"This project demonstrates my ability to design and implement a complete cloud-native solution from scratch — from IAM permissions and Lambda code to API Gateway configuration and a functional frontend. I intentionally used an asynchronous, event-driven architecture to make it production-grade rather than a simple synchronous call chain.

What I'm most proud of is the decoupled design: the user always gets an instant response, and the heavy AI processing happens in the background. This pattern — create-then-process via a message broker — is used extensively in real production systems at scale.

I also handled edge cases like Polly's character limit with a text-splitting algorithm and managed CORS properly for cross-origin browser requests. The application is ready to be extended with authentication, CloudFront, and infrastructure-as-code."

---

*Prepared by Aamir | [linkedin.com/in/aamir-imran](https://www.linkedin.com/in/aamir-imran)*
