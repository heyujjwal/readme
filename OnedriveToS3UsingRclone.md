# **📌 OneDrive → S3 Automated Migration System (Using Rclone + Django API)**

### **Complete Flow Explanation (Clear & Structured)**

This system automates the migration of **any folder from OneDrive to AWS S3**, removing all manual steps except a one-time authentication (only required if OneDrive client credentials are not provided).

---

# **🔷 1. Purpose**

The goal is to eliminate the slow, manual process of downloading files from OneDrive and re-uploading them to S3.
Using **Rclone + Django API**, the migration becomes automatic, repeatable, and error-free.

---

# **🔷 2. Components Involved**

### **a) Rclone**

* CLI tool used to connect to cloud storage services.
* Handles:

  * OneDrive authentication
  * S3 connection
  * File listing
  * Folder copy operations

### **b) Django API**

* Exposes a POST endpoint:

  ```
  POST /api/audio-manager/migrate-folder/
  ```
* Receives folder & remote names.
* Orchestrates the entire migration.

---

# **🔷 3. Input to the API**

Example:

```json
{
  "onedrive_remote": "onedriveCurebay",
  "s3_remote": "s3Migration",
  "folder": "3rd Nov 2025"
}
```

### Meaning:

* **onedrive_remote** → Name of rclone remote that points to OneDrive
* **s3_remote** → Name of rclone remote that points to AWS S3
* **folder** → The OneDrive folder to migrate

---

# **🔷 4. How the Process Works (Step-by-Step)**

## **STEP 1 — API receives the request**

User provides:

* OneDrive remote name
* S3 remote name
* Folder name

The API begins the migration flow.

---

## **STEP 2 — Check if OneDrive remote exists**

* If **remote exists**, continue.
* If **not**, create a new remote:

### Case 1️⃣ – OneDrive credentials NOT available

* Rclone creates remote using default client
* User must authenticate by visiting the browser link
* One-time manual action

### Case 2️⃣ – Credentials available

* API configures the remote automatically
* **Zero manual steps**

Once authenticated, the OneDrive remote is ready forever.

---

## **STEP 3 — Check if S3 remote exists**

* If **exists**, use it
* If not:

  * API creates a remote using AWS `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
  * Fully automatic

---

## **STEP 4 — Validate folder on OneDrive**

API checks:

```
rclone lsf "onedriveRemote:folderName"
```

If the folder does not exist → return error.

If the folder exists → continue.

---

## **STEP 5 — Prepare target S3 path**

Folder name is sanitized:

```
3rd Nov 2025 → 3rd_Nov_2025
```

Final S3 upload path becomes:

```
s3://<bucket>/testing/3rd_Nov_2025/
```

CloudFront URL becomes:

```
https://<cloudfront>/testing/3rd_Nov_2025/
```

---

## **STEP 6 — Dry Run (Safety Check)**

API runs:

```
rclone copy "onedrive:folder" "s3:path" --dry-run
```

This checks:

* Permissions
* Folder visibility
* Rclone connectivity
* Naming/quoting issues

If dry-run fails → API stops and returns error.

---

## **STEP 7 — Actual Copy to S3**

If dry-run is successful, then:

```
rclone copy "onedrive:folder" "s3:path"  
```

Files are uploaded:

* With original file names
* Without changing extensions (`.mp3.mp3` preserved)
* Into the correct S3 folder path

---

## **STEP 8 — Generate CloudFront URLs**

For each uploaded file:

```
https://cloudfront/testing/3rd_Nov_2025/<filename>
```

These are returned in the API response.

---

# **🔷 5. Output from the API**

The response includes:

### ✔ Successful upload

### ✔ All CloudFront URLs

### ✔ Full S3 path

### ✔ Dry-run and copy logs

### ✔ File count

Example:

```json
{
  "success": true,
  "uploaded_to": "s3://databrewery-staging/testing/3rd_Nov_2025/",
  "cloudfront_folder": "https://.../testing/3rd_Nov_2025/",
  "file_count": 13,
  "file_urls": [
    "https://.../202511031157_HIN_001.mp3.mp3",
    ...
  ]
}
```

---

# **🔷 6. One-Time Setup Requirements**

### OneDrive:

* If client credentials are missing → only once user must authenticate in browser
* After that, no human involvement is needed

### AWS:

* S3 Access Key & Secret Key must be present in environment variables
* No manual action required

---

# **🔷 7. Post-Setup — Fully Automated**

After initial setup:

* No more authentication
* No downloading
* No re-uploading
* Just call the API with a folder → the system handles everything

---

# **🔷 Final Summary (One-Liner)**

**A fully automated API-driven pipeline that uses Rclone to migrate any folder from OneDrive to S3, auto-generates CloudFront URLs, eliminates manual file handling, and requires zero human steps once remotes are authenticated.**

---

