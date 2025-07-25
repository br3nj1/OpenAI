
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



## **Cleaning MarkerPDF JSON for AI/RAG Use in Azure**


### Overview

When converting PDFs to JSON using MarkerPDF (especially in Azure environments for document intelligence or AI ingestion), the raw output is often too complex for direct use in Retrieval-Augmented Generation (RAG) pipelines or LLMs.
This article outlines how to flatten, filter, and clean MarkerPDF JSON for optimal AI use—removing unnecessary images, polygons, and empty structures, while extracting meaningful section headings and text.

---

### Why Clean MarkerPDF JSON?

* **Reduces size and complexity** for faster AI processing
* **Improves searchability** by flattening nested sections
* **Removes noise** (e.g., OCR artifacts, images, empty blocks)
* **Makes it trivial to ingest with OpenAI, Azure AI, LlamaIndex, Haystack, etc.**

---

### **Recommended Workflow**

#### 1. **Convert PDF with MarkerPDF**

* Run MarkerPDF (locally or in Azure VM/container)
* You’ll get:

  * `document.json` (full structure)
  * `meta.json` (section/page/heading map)

#### 2. **Clean the Output**

* **Objective:**
  Convert MarkerPDF’s hierarchical, image-heavy JSON into a flat array of `{section, page, content}` for each non-empty text block.
* **Automation:**
  Use the AI prompt below to process your files via ChatGPT, or adapt the code snippet for Python in Azure Functions/Notebooks.

---

#### 3. **Sample Output (Ideal for AI/RAG)**

```json
[
  {
    "section": "Preface",
    "page": 10,
    "content": "In the rapidly evolving world of digital security, 'Mastering Splunk for Cybersecurity' serves as a comprehensive guide..."
  },
  {
    "section": "1. Introduction to Splunk and Cybersecurity",
    "page": 16,
    "content": "Chapter 1 sets the stage for our exploration, outlining the importance of Splunk as a tool in the cybersecurity landscape..."
  }
]
```

* **No images, polygons, or empty/duplicate headers.**
* **Ready for Azure OpenAI, LangChain, LlamaIndex, etc.**

---

## **Reusable ChatGPT Prompt**

Paste this into ChatGPT (or customize as needed):

---

**Prompt:**

```
**PDF to JSON Processing Prompt (For MarkerPDF Exports):**

You are a professional technical assistant.
I have MarkerPDF `.json` and `.meta.json` files exported from a PDF.

I need you to process the `.json` file as follows:

* **Remove** all images, polygons, OCR fields, and bounding box data.
* **Ignore** OCR/image content entirely (assume only embedded text is valid).
* **For every non-empty text block**, output a flat JSON array with:

  * **Section or heading:** Use all available headings concatenated as a single "section" string, e.g., `"Section1 > Subsection > Subsubsection"`, for maximum context.
  * **Page number**
  * **Text content**
* **If section/heading is missing, use an empty string.**
* **Only include non-empty, meaningful main body text blocks**; do **not** include footers, headers, or copyright/boilerplate text unless it is essential to the main content.
* Output format:
  `[{"section": "...", "page": 1, "content": "..."}]`
* Make the result downloadable as a single cleaned JSON file.

**You should always default to these rules unless I specify otherwise.**

```
## **Tips for Azure Pipelines**

* **Automate via Python Notebooks or Functions:**
  Use the same cleaning logic shown above.
* **Store cleaned JSON in Azure Blob or Files for LLM ingestion.**
* **Consider running the cleaning step as a post-process in Azure Data Factory or Synapse.**

---

## **References**

* [MarkerPDF GitHub](https://github.com/MarkerPDF/MarkerPDF)
* [Azure OpenAI Service](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
* [LlamaIndex](https://gpt-index.readthedocs.io/)
* [LangChain](https://python.langchain.com/)

---

## License

This project leverages [marker](https://github.com/VikParuchuri/marker) by Vik Paruchuri (see marker repo for license and usage info).

---

## Issues / Questions

Open an issue on this GitHub repo or the [marker repo](https://github.com/VikParuchuri/marker/issues) for additional support.

---
