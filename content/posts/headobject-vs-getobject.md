+++
date = '2025-11-06T23:40:28+07:00'
title = "HeadObjectCommand vs GetObjectCommand in AWS SDK"
description = "Understand the difference between HeadObjectCommand and GetObjectCommand in AWS SDK for JavaScript (v3)."
tags = ["AWS", "S3", "Node.js", "Backend", "Dev Notes", "Media Service"]
+++

## 🧠 Introduction

When working with **Amazon S3**, two common commands developers use are `HeadObjectCommand` and `GetObjectCommand`.  
At first glance, they might seem similar — both retrieve object information — but they serve **different purposes**.

Let’s explore how they differ and when to use each.

---

## ⚙️ `HeadObjectCommand`

`HeadObjectCommand` **retrieves only the metadata** of an S3 object, **without downloading the file content**.

This is ideal when you just want to:

- Check if a file exists.
- Get metadata (like Content-Type, Content-Length, Last-Modified, ETag, etc.).
- Verify object accessibility or permissions.

### ✅ Example

```ts
import { S3Client, HeadObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: "ap-southeast-1" });

async function checkFileInfo() {
  try {
    const command = new HeadObjectCommand({
      Bucket: "my-bucket",
      Key: "example/data.json",
    });

    const response = await s3.send(command);
    console.log("✅ Object found:", response);
  } catch (err) {
    if (err.name === "NotFound") {
      console.log("❌ Object not found");
    } else {
      console.error("⚠️ Error:", err);
    }
  }
}

checkFileInfo();
```

### 💡 Output (sample)

```json
{
  "AcceptRanges": "bytes",
  "LastModified": "2025-10-31T02:45:10.000Z",
  "ContentLength": 24987,
  "ContentType": "application/json",
  "ETag": "\"d41d8cd98f00b204e9800998ecf8427e\""
}
```

👉 **No file data** is returned — only metadata.

---

## 📦 `GetObjectCommand`

`GetObjectCommand` **retrieves the full object** (its content and metadata).  
Use it when you actually need to **read or process file contents**.

### ✅ Example

```ts
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { Readable } from "stream";

const s3 = new S3Client({ region: "ap-southeast-1" });

async function downloadFile() {
  const command = new GetObjectCommand({
    Bucket: "my-bucket",
    Key: "example/data.json",
  });

  const { Body, ContentType } = await s3.send(command);
  console.log("📦 Content-Type:", ContentType);

  // Convert stream to string (for small files)
  const streamToString = (stream: Readable) =>
    new Promise((resolve, reject) => {
      const chunks: any[] = [];
      stream.on("data", (chunk) => chunks.push(chunk));
      stream.on("end", () => resolve(Buffer.concat(chunks).toString("utf-8")));
      stream.on("error", reject);
    });

  const content = await streamToString(Body as Readable);
  console.log("📄 File Content:", content);
}

downloadFile();
```

---

## ⚖️ Comparison Table

| Feature        | HeadObjectCommand             | GetObjectCommand                    |
| -------------- | ----------------------------- | ----------------------------------- |
| Purpose        | Check existence / metadata    | Retrieve full object                |
| Downloads data | ❌ No                         | ✅ Yes                              |
| Typical usage  | Validation, info, cache check | File reading, streaming, processing |
| Performance    | Faster (metadata only)        | Slower (includes full data)         |
| Cost           | Lower (no data transfer)      | Higher (data transfer applies)      |

---

## 🧩 When to Use Which?

- ✅ **Use `HeadObjectCommand`**  
  When you only need to verify if an object exists or get metadata quickly and cheaply.

- ✅ **Use `GetObjectCommand`**  
  When you actually need the object content (to read, download, or process).

---

## 📚 References

- [AWS SDK for JavaScript v3 – HeadObjectCommand](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/classes/headobjectcommand.html)
- [AWS SDK for JavaScript v3 – GetObjectCommand](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/classes/getobjectcommand.html)
- [AWS S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/API_HeadObject.html)

---

## 🧾 Summary

In short:

> `HeadObjectCommand` → metadata only  
> `GetObjectCommand` → metadata + file content

Use the **right command** for your case to optimize performance and cost.

---

_Written by [Frank Nguyen](https://github.com/hthanh12)_  
_(c) 2025 — Notes from real-world AWS projects 💻)_
