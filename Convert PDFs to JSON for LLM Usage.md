
# Convert PDFs to JSON for LLM Knowledge Base

*Using Azure VM, Storage, and [marker](https://github.com/VikParuchuri/marker)*

---

## Overview

This guide helps you:

- Deploy an Azure VM for processing PDFs
- Store and organize PDFs in Azure Blob Storage
- Convert PDFs (including scanned/image-based) to structured JSON for RAG/LLM use with marker
- Efficiently manage batch jobs, logs, and error correction

---

## 1. Azure VM Setup

**Recommended specs:**

- OS: Ubuntu 24.04
- Size: Standard F2ams v6 (2 vCPU, 16 GiB RAM)
- Create a local admin account (e.g., `markerpdf`)

---

## 2. Azure Storage & Upload PDFs

1. Create an Azure Storage Account
2. Create a Blob Container (e.g., `container01`)
3. Upload your PDFs (organize in virtual folders if needed, e.g. `azure-set-1/`)

---

## 3. VM Compute Environment Setup

Install system dependencies:

```bash
sudo apt update
sudo apt install -y python3 python3-pip git poppler-utils python3-venv
```

Clone and set up marker in a virtual environment:

```bash
cd /opt
git clone https://github.com/VikParuchuri/marker.git
cd marker
python3 -m venv marker-venv
source marker-venv/bin/activate
pip install -e .
```

**Activate the venv for each shell session before running marker:**

```bash
source /opt/marker/marker-venv/bin/activate
```

---

## 4. Copy PDFs from Azure Storage to VM

Login and download with Azure CLI:

```bash
az login
az storage blob download-batch   --account-name markerpdf   --account-key '<your-key>'   --destination ./pdfs/azure-set-1   --source container01   --pattern 'azure-set-1/*.pdf'   --output table
```

---

## 5. Convert PDFs with marker

### Single File Example

```bash
marker_single "./pdfs/azure-set-1/yourdoc.pdf" --output_format json --output_dir ./pdfs/chunks/azure-set-1
```

### Batch Convert Example

```bash
marker /opt/marker/pdfs/azure-set-1   --output_dir /opt/marker/pdfs/chunks/azure-set-1   --output_format json   --strip_existing_ocr   --workers 1
```
**Note:** Use `--strip_existing_ocr` to prevent excessive RAM use on scanned/image-based PDFs.

---

## 6. Upload Results to Azure Blob Storage

```bash
az storage blob upload-batch   --account-name markerpdf   --account-key '<your-key>'   --destination container01/json_azure-set-1   --source /opt/marker/pdfs/chunks/azure-set-1   --output table
```

---

## 7. Batch Script with Log and Wall Notification

Create a script to manage and monitor long jobs:

<details>
<summary>Click to expand script</summary>

```bash
#!/bin/bash

SRC_DIR="/opt/marker/pdfs/azure-set-1"
DEST_DIR="/opt/marker/pdfs/chunks/azure-set-1"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOGFILE="$DEST_DIR/marker_job_${TIMESTAMP}.log"

mkdir -p "$DEST_DIR"

if pgrep -fl "marker $SRC_DIR" | grep -v $$; then
    echo "A marker job for $SRC_DIR is already running. Exiting to avoid duplicate processing."
    exit 1
fi

echo "Starting marker batch job..."
echo "Logs will be written to: $LOGFILE"
echo "To monitor progress, run: tail -f $LOGFILE"

(
    echo "Start: $(date)"
    nohup marker "$SRC_DIR"       --output_dir "$DEST_DIR"       --output_format json       --strip_existing_ocr       --workers 1       >> "$LOGFILE" 2>&1
    echo "End: $(date)"
) &
JOB_PID=$!
echo "Marker job started with PID $JOB_PID"

(
    wait $JOB_PID
    MSG="
=========================================================
====         MARKER BATCH JOB COMPLETE!              ====
=========================================================
Source directory : $SRC_DIR
Output directory : $DEST_DIR
Log file         : $LOGFILE
Completed at     : $(date)
=========================================================
"
    echo -e "$MSG" | wall
    echo "$MSG"
) &
exit 0
```
</details>

**Usage:**

```bash
chmod +x run_marker_wall.sh
./run_marker_wall.sh
tail -f /opt/marker/pdfs/chunks/azure-set-1/marker_job_YYYYMMDD_HHMMSS.log
```
When complete, a wall message will alert all open terminals.

---

## 8. Troubleshooting & Error Correction

### High RAM/OCR Issues  
- **Symptom:** Marker tries to use 100GB+ RAM or is killed (`Killed` message).
- **Solution:** Always use `--strip_existing_ocr` with image-based or scanned PDFs.

### Too Many Open Files  
- **Symptom:** `RuntimeError: unable to open shared memory object ... Too many open files (24)`
- **Fix:**
  ```bash
  ulimit -n 16384
  ```
  For a permanent fix, add to `/etc/security/limits.conf`:
  ```
  * soft nofile 4096
  * hard nofile 8192
  ```
  Log out/in for permanent changes.

### Out of Memory (OOM)  
- **Fix:** Add swap to your VM
  ```bash
  sudo fallocate -l 16G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
  free -h
  ```
- For massive PDFs, process fewer at a time or split PDFs.

### Processing Large Folders  
Process 1–2 PDFs at a time if hitting RAM/OOM limits:
```bash
ls "/opt/marker/pdfs/azure-set-1/"*.pdf | while read f; do
  marker_single "$f" --output_format json --strip_existing_ocr --output_dir "/opt/marker/pdfs/chunks/azure-set-1"
done
```

---

## References

- [marker GitHub](https://github.com/VikParuchuri/marker)
- [Azure CLI: az storage blob](https://learn.microsoft.com/en-us/cli/azure/storage/blob)

---

## Common Marker Commands

| Task                           | Command Example                                                                                      |
|--------------------------------|------------------------------------------------------------------------------------------------------|
| Activate venv                  | `source /opt/marker/marker-venv/bin/activate`                                                        |
| Single PDF to JSON             | `marker_single "./pdfs/fileName.pdf" --output_format json --output_dir ./pdfs/chunks`                |
| Batch PDF to JSON              | `marker ./pdfs --output_dir ./pdfs/chunks --output_format json --strip_existing_ocr --workers 1`      |
| Download from Azure Blob       | `az storage blob download-batch --account-name ... --account-key ... --destination ... --source ...`  |
| Upload to Azure Blob           | `az storage blob upload-batch --account-name ... --account-key ... --destination ... --source ...`    |

---

## License

This project leverages [marker](https://github.com/VikParuchuri/marker) by Vik Paruchuri (see marker repo for license and usage info).

---

## Issues / Questions

Open an issue on this GitHub repo or the [marker repo](https://github.com/VikParuchuri/marker/issues) for additional support.

---
