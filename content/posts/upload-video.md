+++
date = '2025-11-07T22:09:19+07:00'
title = "Optimizing Video Uploads: Multipart Upload, Pre-Signed URLs, and S3 Events"
description = "How to build a scalable, reliable video upload system using AWS S3, pre-signed URLs, multipart uploads, and S3 event triggers — with structured logging best practices (and a sprinkle of developer humor)."
+++

## 🚀 Optimizing Video Uploads: Multipart Upload, Pre-Signed URLs, and S3 Events

Uploading big video files is like trying to send a watermelon through a straw 🍉 — it’s slow, clunky, and often ends in frustration.  
But fear not! With **AWS S3**, **pre-signed URLs**, and **multipart uploads**, we can make uploads as smooth as butter on a warm pancake 🥞.

In this post, I’ll show how to:

- Use **Pre-signed URLs** for secure direct uploads
- Enable **Multipart Upload** for speed and reliability
- Trigger **S3 Events** for post-processing
- And add **structured logs** to know exactly what’s happening — even when the servers are pretending everything’s fine 😅

---

## 🧱 1. Why Pre-Signed URLs?

Uploading directly to your backend is like sending every file through one poor server who’s already overworked.  
Instead, let S3 handle it directly — your server just gives the client a **ticket** (the pre-signed URL).

```ts
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: "ap-southeast-1" });

export const getUploadUrl = async (filename: string, type: string) => {
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: `uploads/${filename}`,
    ContentType: type,
  });

  const signedUrl = await getSignedUrl(s3, command, { expiresIn: 3600 });

  console.log(`[Upload] URL created for ${filename}`);
  return { url: signedUrl };
};
```

✅ No large files clogging your backend.  
✅ No crying servers.  
✅ No random “502 Bad Gateway” at 2AM.

---

## 🧩 2. Multipart Uploads — Faster & Resumable

For large video files (>100MB), single uploads often fail halfway — like watching Netflix buffer forever and _then_ crash.  
Multipart upload splits the file into small pieces, uploads them in parallel, and then reassembles them in S3.

```js
const upload = s3.upload({
  Bucket: "media-bucket",
  Key: "videos/test.mp4",
  Body: file,
});

upload.on("httpUploadProgress", (event) => {
  console.log(`[Upload Progress] ${event.loaded}/${event.total}`);
});
```

### Benefits:

- ⚡ Faster (parallel uploads)
- 🧩 Resumable (in case Wi-Fi decides to go on vacation)
- 🧠 Easier to track each part with logs

### 📝 Example Logs

| Stage       | Example Log                                            | Description   |
| :---------- | :----------------------------------------------------- | :------------ |
| Request     | `[Upload] Request started - file=test.mp4, size=420MB` | Upload begins |
| Part Upload | `[Upload] Part 3 of 8 completed (37%)`                 | Chunk done    |
| Complete    | `[Upload] Completed successfully - duration=52s`       | 🎉 Success    |
| Error       | `[Upload] Failed - network timeout part=4`             | 😭 Retry time |

Logging is like journaling for your backend — it helps future-you understand what went wrong when past-you was “sure it worked”.

---

## ⚡ 3. Triggering Processing via S3 Events

Once uploaded, S3 can automatically notify your system using **S3 Events**.  
This is how you trigger transcoding, thumbnail generation, or whatever dark magic your pipeline performs.

```yaml
# Example: s3-bucket.yml
Events:
  - s3:ObjectCreated:*
LambdaFunction:
  FunctionName: processVideo
  Handler: handler.process
```

When the event fires:

```js
export const handler = async (event) => {
  for (const record of event.Records) {
    const key = record.s3.object.key;
    console.log(`[S3 Event] New upload detected: ${key}`);
  }
};
```

🎬 “ACTION!” — that’s your cue to start processing.

---

## 🧠 4. Monitoring & Debugging (or, “Why did it fail _this_ time?”)

In production, I always send logs to **CloudWatch** or **Sentry**.  
Each log is structured, so when things go wrong, you can actually _read_ them instead of decoding a mystery stack trace.

```json
{
  "timestamp": "2025-11-07T14:12:00Z",
  "level": "info",
  "event": "UploadCompleted",
  "userId": "u123",
  "fileKey": "uploads/test.mp4",
  "duration": 49.68
}
```

👀 With structured logs, debugging becomes _detective work_ instead of _guesswork_.

---

## ✅ Summary

| Component            | Purpose                | Logging Tip                  |
| :------------------- | :--------------------- | :--------------------------- |
| **Pre-Signed URL**   | Secure client upload   | Log URL generation + expiry  |
| **Multipart Upload** | Reliability + speed    | Log part progress + retries  |
| **S3 Events**        | Auto trigger workflows | Log processing start/finish  |
| **Centralized Logs** | Debug & metrics        | Send to CloudWatch or Sentry |

---

## 🎉 Final Thoughts

If you’ve ever yelled at an upload bar that froze at 99%, this setup is for you.  
Your system will be **faster**, **cheaper**, and way more **observant** — and your logs will finally make sense.

Now go build something awesome!  
(And maybe send a thank-you note to your S3 bucket for doing all the heavy lifting 💪)

---
