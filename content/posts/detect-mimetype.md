+++
date = '2025-11-22T01:02:36+07:00'
title = 'How to Detect MIME Types in Node.js: `file-type` vs Linux `file` (and the 5MB Rule)'
+++

Detecting the correct MIME type is critical when processing uploads, generating thumbnails, validating content, or securing your storage pipeline. In Node.js, developers usually choose between two common approaches:

1. **`file-type` library** → detect MIME from **buffer**
2. **Linux `file` command** → detect MIME from **file path**

Both methods work well—but _not_ for the same scenarios.  
This guide shows you the differences, the performance impact, and a **best-practice hybrid method using the 5MB rule**.

---

## Why Two Methods?

Because:

- **`file-type` (buffer)** is _fast_, _cross-platform_, but requires loading file bytes into memory.
- **Linux `file` (shell command)** is _memory-safe_ for large files and extremely accurate—but slower due to I/O + process spawn.

So the ideal solution is **combining both**.

---

# 1. `file-type` — Detecting MIME from Buffer

`file-type` works by scanning only the **magic bytes** of the file buffer.

### ✔ Pros

- Very fast (microseconds)
- No external dependencies
- Works on all OS
- Doesn't require reading the whole file to disk

### ✘ Cons

- You must load bytes into memory → **Bad for large files**
- Doesn’t work if you only know the file path
- Limited support for some formats

```ts
import { fromBuffer } from "file-type";
const info = await fromBuffer(buffer);
```

---

# 2. Linux `file` — Detecting MIME from Path

Linux provides an extremely accurate MIME detector:

```bash
file --mime-type -b /path/to/file
```

### ✔ Pros

- Very accurate
- Memory-safe for large files
- Works even when file is already saved on disk

### ✘ Cons

- Requires spawning a child process → slower
- Linux-only (not ideal for Windows dev)
- Adds OS dependency

---

# The 5MB Best Practice Rule

### **Use buffer detection (`file-type`) when file < 5MB**

→ Fast, lightweight, ideal for most uploads.

### **Use Linux detection when file ≥ 5MB**

→ Avoids loading huge files into RAM.  
→ More stable for high-throughput applications.

### Why 5MB?

Most servers operate with memory-sensitive constraints. Files larger than 5MB:

- Increase V8 memory pressure
- Trigger garbage collection more often
- Slow down the event loop
- Risk out-of-memory when multiple uploads run concurrently

5MB is a safe threshold used in many production systems (but can be adjusted).

---

# 3. Hybrid Utility Function

```ts
import { fromBuffer } from "file-type";
import { spawnSync } from "child_process";
import fs from "fs";

const THRESHOLD = 5 * 1024 * 1024; // 5MB

export async function detectMimeSmart(filePath: string) {
  const stats = fs.statSync(filePath);

  // Case 1: < 5MB → use file-type (buffer)
  if (stats.size < THRESHOLD) {
    const buffer = fs.readFileSync(filePath);
    const type = await fromBuffer(buffer);
    if (type?.mime) return type.mime;
  }

  // Case 2: >= 5MB → use Linux `file`
  const result = spawnSync("file", ["--mime-type", "-b", filePath]);
  return result.stdout.toString().trim();
}
```

---

# 4. When Should You NOT Use `file-type`?

Avoid buffer detection when:

- Files can be **hundreds of MB or GB**
- Users upload **videos**, **RAW images**, or **large ZIPs**
- Running on memory-restricted systems (AWS Lambda, K8s pods, serverless)

---

# 5. When Should You Use Linux `file`?

Use Linux `file` for:

- Large uploads (≥ 5MB)
- Background thumbnail jobs
- Batch processing
- Microservices that run on Ubuntu/Debian

---

## Final Recommendation

| File Size | Method               | Reason                 |
| --------- | -------------------- | ---------------------- |
| `< 5MB`   | `file-type` (buffer) | Fast, efficient        |
| `≥ 5MB`   | Linux `file`         | No RAM overhead, safer |

This hybrid approach gives you:

- ⚡ Best performance
- 🔒 Best memory safety
- 🎯 Best accuracy
