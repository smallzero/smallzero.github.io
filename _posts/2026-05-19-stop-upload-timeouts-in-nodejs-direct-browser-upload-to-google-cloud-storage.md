---
layout: post
title: Stop Upload Timeouts in Node.js Direct Browser Uploads to Google Cloud Storage (Express + HTML)
categories: Programming
excerpt: Large file uploads through a Node.js Express backend often end in request timeout. If this sounds familiar, the fix is usually architectural—not just increasing timeout settings.
---

Large file uploads through a Node.js/Express backend often end in **request timeout**.  
If this sounds familiar, the fix is usually architectural—not just increasing timeout settings.

In this article, I’ll show how to move uploads directly from browser to **Google Cloud Storage (GCS)** using a **Signed POST Policy**, while your Express API only handles secure policy generation.

---

## Why uploads timeout in Express apps

When users upload big files to your backend first, your app becomes the middleman for all file bytes:

- Browser → Express → Storage
- Higher memory/CPU/network pressure on app server
- Reverse proxy/serverless timeout limits (Nginx, ALB, Cloud Run, etc.)
- Slow or unstable client networks make timeout risk worse

Even if you tune timeouts, your app still carries unnecessary upload bandwidth.

---

## Better architecture: direct-to-GCS upload

Use this flow instead:

1. User selects file in your web app.
2. Browser asks Express API for a short-lived upload policy.
3. Express creates a **Signed POST Policy V4** for GCS.
4. Browser uploads file **directly to GCS** (multipart/form-data).
5. Browser optionally notifies backend to store metadata.

✅ Benefits:

- No heavy upload traffic through Express
- Less timeout risk
- Better scalability
- Better security control (private bucket, short-lived signatures)

---

## “Can I redirect users to GCS upload UI?”

Not really for end users.

GCS has Google Cloud Console UI, but it is for admins/developers with IAM access—not a public upload portal for your customers.

For a web app, the right approach is:

- Build your own upload page, then
- Upload directly to GCS using signed policy or signed URL

---

## Project setup

Install dependencies:

```bash
npm init -y
npm i express @google-cloud/storage
```

Create structure:

```text
.
├── server.js
└── public
    └── index.html
```

---

## Express backend: generate Signed POST Policy

```js
const express = require('express');
const path = require('path');
const crypto = require('crypto');
const { Storage } = require('@google-cloud/storage');

const app = express();
const port = process.env.PORT || 3000;

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

const storage = new Storage({
  projectId: process.env.GCP_PROJECT_ID,
});

const bucketName = process.env.GCS_BUCKET;
if (!bucketName) throw new Error('Missing env GCS_BUCKET');
const bucket = storage.bucket(bucketName);

app.post('/api/upload-policy', async (req, res) => {
  try {
    const { fileName, contentType } = req.body;
    if (!fileName || !contentType) {
      return res.status(400).json({ error: 'fileName and contentType are required' });
    }

    const safeName = fileName.replace(/[^\w.\-]/g, '_');
    const objectKey = `uploads/${Date.now()}-${crypto.randomUUID()}-${safeName}`;
    const expires = Date.now() + 10 * 60 * 1000; // 10 minutes
    const maxSize = 200 * 1024 * 1024; // 200 MB

    const [policy] = await bucket.file(objectKey).generateSignedPostPolicyV4({
      expires,
      conditions: [
        ['eq', '$Content-Type', contentType],
        ['content-length-range', 0, maxSize],
      ],
    });

    return res.json({
      url: policy.url,
      fields: policy.fields,
      objectKey,
      maxSize,
      expiresAt: new Date(expires).toISOString(),
    });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: 'Failed to create upload policy' });
  }
});

app.post('/api/upload-complete', async (req, res) => {
  const { objectKey, originalName, size } = req.body;
  // Save metadata to DB if needed
  return res.json({ ok: true, objectKey, originalName, size });
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
```

---

## Frontend HTML: upload directly to GCS

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>GCS Direct Upload</title>
</head>
<body>
  <h2>Upload file to GCS</h2>
  <input id="fileInput" type="file" />
  <button id="btnUpload">Upload</button>
  <pre id="log"></pre>

  <script>
    const fileInput = document.getElementById('fileInput');
    const btnUpload = document.getElementById('btnUpload');
    const logEl = document.getElementById('log');
    const log = (m) => (logEl.textContent += m + '\n');

    btnUpload.addEventListener('click', async () => {
      try {
        const file = fileInput.files[0];
        if (!file) return alert('Choose a file first');

        // 1) Request policy
        const policyResp = await fetch('/api/upload-policy', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            fileName: file.name,
            contentType: file.type || 'application/octet-stream',
          }),
        });

        if (!policyResp.ok) throw new Error('Failed to get upload policy');
        const policy = await policyResp.json();

        // 2) Upload directly to GCS
        const formData = new FormData();
        Object.entries(policy.fields).forEach(([k, v]) => formData.append(k, v));
        formData.append('Content-Type', file.type || 'application/octet-stream');
        formData.append('file', file);

        const gcsResp = await fetch(policy.url, {
          method: 'POST',
          body: formData,
        });

        if (!gcsResp.ok) {
          const text = await gcsResp.text();
          throw new Error(`GCS upload failed: ${gcsResp.status} ${text}`);
        }

        // 3) Optional callback
        await fetch('/api/upload-complete', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            objectKey: policy.objectKey,
            originalName: file.name,
            size: file.size,
          }),
        });

        log('Upload success');
      } catch (e) {
        log(`Error: ${e.message}`);
      }
    });
  </script>
</body>
</html>
```

---

## Environment variables

```bash
export GCP_PROJECT_ID="your-project-id"
export GCS_BUCKET="your-bucket-name"
export GOOGLE_APPLICATION_CREDENTIALS="/absolute/path/service-account.json"
```

Run:

```bash
node server.js
```

Open:

```text
http://localhost:3000
```

---

## Configure bucket CORS (required for browser upload)

Create `cors.json`:

```json
[
  {
    "origin": ["http://localhost:3000"],
    "method": ["POST", "GET", "HEAD"],
    "responseHeader": ["Content-Type", "x-goog-resumable"],
    "maxAgeSeconds": 3600
  }
]
```

Apply:

```bash
gsutil cors set cors.json gs://your-bucket-name
```

---

## Security checklist (important)

- Keep bucket **private**
- Use short policy expiration (5–10 minutes)
- Authenticate user before issuing upload policy
- Scope object key to user/tenant (e.g. `uploads/{userId}/...`)
- Enforce file size/type conditions
- Store metadata in DB
- Use signed URLs for downloads too

---

## When to use resumable upload

For very large files or unstable mobile networks, use **GCS resumable uploads**.  
This supports chunked transfer and retry, making uploads much more reliable.

---

## Final takeaway

If your Express app times out during large uploads, don’t just increase timeout values.  
Move to **direct browser → GCS uploads** with signed policies/URLs.

You’ll reduce server load, improve reliability, and gain better security control with a cloud-native upload flow.