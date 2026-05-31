# PDF Malware Analysis Lab

<div align="center">

[![Docker](https://img.shields.io/badge/Docker-REMnux-blue?logo=docker&logoColor=white)](https://hub.docker.com/r/remnux/remnux-distro)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Language](https://img.shields.io/badge/Language-Python%20%7C%20Bash-orange.svg)](#)
![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-success.svg)

**A comprehensive guide to identifying malicious objects, JavaScript, and obfuscated triggers in suspicious PDF files using an isolated Docker environment.**

[⚡ Quick Start](#-quick-start) • [🔍 Analysis Workflow](#-analysis-workflow) • [📚 Tools Reference](#-reference-tools) • [⚠️ Safety](#️-safety-warning)

</div>

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#-prerequisites)
- [System Configuration](#system-configuration)
- [Environment Setup](#-environment-setup)
- [Quick Start](#-quick-start)
- [Analysis Workflow](#-analysis-workflow)
- [PDF Sanitization](#-to-pacify-the-pdf-file)
- [Cleanup](#-reclaiming-system-resources)
- [Reference Tools](#-reference-tools)
- [Safety Warning](#️-safety-warning)

---

## Overview

This project provides **step-by-step instructions** to safely analyze suspicious PDF files for malicious content. Using Docker and REMnux, you can:

✅ Scan PDFs for dangerous keywords  
✅ Extract hidden JavaScript  
✅ Detect obfuscated malware triggers  
✅ Sanitize malicious PDFs  
✅ Work in a **completely isolated environment**  

---

## 🛠️ Prerequisites

Before starting, ensure you have:

| Requirement | Details |
|-------------|---------|
| **Docker Desktop** | [Download here](https://www.docker.com/products/docker-desktop) |
| **WSL 2 Backend** | Required for Windows users |
| **RAM Allocation** | Min 4GB for small PDFs, 8GB+ for large files (300MB+) |
| **Disk Space** | ~3GB for REMnux image |

---

## System Configuration

### Windows Users: WSL 2 Memory Setup

Create a configuration file to prevent system crashes during analysis:

1. **Open PowerShell as Administrator**
2. **Create the file** `C:\Users\<YourUsername>\.wslconfig`:

```ini
[wsl2]
memory=4GB
swap=2GB

[experimental]
autoMemoryReclaim=gradual
```

3. **Apply changes:**
```powershell
wsl --shutdown
```

> **Why?** Large PDFs can consume significant memory. This config prevents your system from freezing.

---

## 🚀 Environment Setup

This lab uses **REMnux** (Reverse-Engineering Malware Linux), a specialized Docker image with all analysis tools pre-installed.

### Step-by-Step Setup

#### 1. Navigate to Your Analysis Folder
```bash
cd C:\Users\YourUsername\Documents\pdf_analysis
```

#### 2. Start the Docker Container
**Run without Internet connection:**
```bash
docker run --rm -it --network none -m 4g -u remnux -v "%cd%":/home/remnux/files remnux/remnux-distro:noble bash
```

**Run with Internet Connection**:
```bash
docker run --rm -it -m 4g -u remnux -v "%cd%":/home/remnux/files remnux/remnux-distro:noble bash
```


**Command Breakdown:**
- `--rm` → Auto-clean container after exit
- `-it` → Interactive terminal
- `-m 4g` → Allocate 4GB RAM
- `-u remnux` → Run as non-root user
- `-v "%cd%":/home/remnux/files` → Mount current folder

#### 3. Enter the Workspace
```bash
cd /home/remnux/files
ls -la
```

You should now see all your suspicious PDFs in the container.

---

## ⚡ Quick Start

**For the impatient:** Run these commands in order:

```bash
# 1. Initial scan for suspicious keywords
pdfid.py "suspicious_file.pdf"

# 2. If suspicious, decompress the PDF
qpdf --qdf --object-streams=disable "suspicious_file.pdf" expanded.pdf

# 3. Scan the expanded version
pdfid.py expanded.pdf

# 4. Extract JavaScript (if found)
pdf-parser.py --search javascript expanded.pdf
```

---

## 🔍 Analysis Workflow

### Step 1️⃣: High-Level Keyword Scanning with `pdfid`

This tool scans PDF structure for **dangerous keywords without executing them**.

```bash
pdfid.py "suspicious_file.pdf"
```

#### 🚩 Red Flags to Watch For

| Keyword | Risk Level | What It Means |
|---------|-----------|--------------|
| `/JS` or `/JavaScript` | 🔴 **HIGH** | Embedded scripts that can execute |
| `/OpenAction` | 🔴 **HIGH** | Code runs automatically when PDF opens |
| `/AA` (Additional Actions) | 🔴 **HIGH** | Code triggered by scrolling, clicking, or page transition |
| `/ObjStm` | 🟡 **MEDIUM** | Object stream (can hide malicious objects) |
| `/Encrypt` | 🟡 **MEDIUM** | Encryption often used to hide payloads from antivirus |
| `/Launch` | 🟡 **MEDIUM** | Can execute external programs |
| `/EmbeddedFile` | 🟡 **MEDIUM** | Hidden files embedded in PDF |

**Example Output:**
```
ID: 1
Version: 1.4
Extension: pdf
Producer: 
Creator: 
Encrypted: False
Number of objects: 87
/JS: 1          ← SUSPICIOUS!
/JavaScript: 0
/OpenAction: 0
/AA: 0
/ObjStm: 5      ← Also suspicious
/Encrypt: 0
```

---

### Step 2️⃣: Decompress Object Streams

If `/ObjStm` count is high or `/JS` is hidden, decompress to reveal hidden objects:

<details>
<summary><b>🔓 Expanding Object Streams</b></summary>

#### Why decompress?
Object streams compress multiple PDF objects into one. Malware authors use this to **hide JavaScript from initial scans**.

#### Run this command:
```bash
qpdf --qdf --object-streams=disable suspicious_file.pdf expanded.pdf
```

#### Then scan the new file:
```bash
pdfid.py expanded.pdf
```

#### ✅ What to look for:
- If `/JS` was **0 before** but is now **1 or higher** → JavaScript was hidden!
- Compare the object counts between original and expanded versions

#### 📊 Interpretation:
- **Large files (100MB+):** Compression is normal. Still peek inside `/JS` objects.
- **Small files (<10MB):** High `/ObjStm` counts are highly suspicious.

</details>

---

### Step 3️⃣: Extract and Analyze JavaScript

<details>
<summary><b>📜 Extracting Hidden JavaScript</b></summary>

#### Locate JavaScript Objects:
```bash
pdf-parser.py --search javascript expanded.pdf
```

You'll see output like:
```
Obj 123 0
...
JavaScript content here
...
```

#### Extract the Complete Code:
```bash
pdf-parser.py --object 123 --filter --raw expanded.pdf > extracted_code.js
```

**Command Flags:**
- `--object 123` → Replace with the object ID you found
- `--filter` → Decode "FlateDecode" compression (zlib)
- `--raw` → Remove extra formatting

#### 🚨 Common Malicious Patterns:

| Pattern | Purpose | Risk |
|---------|---------|------|
| `this.exportDataObject` | Extract and launch hidden EXE files | 🔴 **CRITICAL** |
| `util.printf("%", ...)` | Trigger buffer overflow in PDF readers | 🔴 **CRITICAL** |
| `app.launchURL` | Open malicious website in browser | 🔴 **HIGH** |
| `getAnnots` | "Side-loading" attacks, hide data in comments | 🟡 **MEDIUM** |
| `collab.collectEmailData` | Steal user data | 🔴 **CRITICAL** |
| `unescape` | Obfuscation technique | 🟡 **MEDIUM** |

#### ⚠️ Obfuscation Warning:
If code looks like this:
```javascript
var a = "x72\x65\x76\x65\x72\x73\x65";
```

**Do NOT run it manually.** Obfuscated malicious JS detects analysis and changes behavior. Use a disassembler or stay in the isolated environment.

</details>

---

### Step 4️⃣: Deep Object Inspection with `pdf-parser`

If standard scanning misses something:

```bash
# Search for all JS objects
pdf-parser.py --search JS "suspicious_file.pdf"

# Extract a specific object (replace 77 with actual ID)
pdf-parser.py -o 77 -f "suspicious_file.pdf"

# Get full analysis
pdf-parser.py -a "suspicious_file.pdf"
```

#### Handling Common Errors:

| Error | Cause | Solution |
|-------|-------|----------|
| `zlib.error` | Corrupted headers (anti-analysis tactic) | Use `peepdf` instead |
| No output | File may be PDF 2.0 or corrupted | Try `strings` command |
| Permission denied | Running as root | Use `-u remnux` flag |

---

### Step 5️⃣: Alternative Extraction with `peepdf`

If `pdf-parser` fails:

```bash
# Open interactive mode
peepdf -i "file_name.pdf"
```

Once the prompt loads (`PP>`), extract JavaScript:

```bash
extract js > script.js
exit
```

Then read the extracted code:
```bash
cat script.js
```

---

### Step 6️⃣: Brute-Force String Extraction

For corrupted or extremely large PDFs:

```bash
# Search for common malicious commands
strings "suspicious_file.pdf" | grep -iE "powershell|cmd.exe|http|eval\(|unescape\("

# Extract with context (5 lines before/after)
strings "suspicious_file.pdf" | grep -i "http" -B 5 -A 5
```

**Suspicious patterns to search for:**
- `powershell` → Shell command execution
- `cmd.exe` → Windows command line
- `http://` or `https://` → Network connections
- `eval(` → Dynamic code execution
- `base64` → Obfuscated payloads

---

### Step 7️⃣: Final Verification

```bash
# Get structural analysis
pdf-parser.py -a "suspicious_file.pdf"
```

Look for this output:
```
Catalog: 1
Info: 1
Pages: 79
Actions: 0     ← Should be 0
OpenAction: 0  ← Should be 0
```

**✅ Clean Result:** All action counts are 0  
**❌ Infected Result:** Any non-zero values = malware present

---

## 🛡️ To Pacify the PDF File

If you've identified malicious content, sanitize the PDF:

### Option 1: Ghostscript Sanitization (Quick)

```bash
gs -o "output_SANITIZED.pdf" -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress "suspicious_file.pdf"
```

**What this does:**
- Removes embedded scripts
- Flattens document structure
- Reduces file size
- Preserves readability

---

### Option 2: Nuclear Option - Image-Based Conversion (Safest)

This **destroys all active content** by converting pages to images:

#### Step 1: Create working folder
```bash
mkdir safe_pdf_output
```

#### Step 2: Convert pages to images (300 DPI)
```bash
gs -dNOPAUSE -dBATCH -sDEVICE=png16m -r300 -sOutputFile=safe_pdf_output/page-%d.png "suspicious_file.pdf"
```

**Why 300 DPI?** Sweet spot for clarity without excessive file size.

#### Step 3: Strip metadata from images (Security!)
```bash
for f in safe_pdf_output/*.png; do mogrify -strip "$f"; done
```

#### Step 4: Convert images back to PDF

**Method A - Using img2pdf (Recommended):**

First install:
```bash
sudo apt-get update && sudo apt-get install img2pdf -y
```

Then convert (maintains page order):
```bash
img2pdf $(ls -v safe_pdf_output/*.png) -o "output_SAFE.pdf"
```

**Method B - Using GUI PDF editor:**
1. Copy `.png` files to your Windows desktop
2. Use any PDF editor (Preview, Adobe, online tools)
3. Import images and export as PDF

---

### Final Verification

Always verify sanitization worked:

```bash
pdfid.py "output_SAFE.pdf"
```

**Expected result:**
```
/JS: 0          ← Must be 0
/JavaScript: 0  ← Must be 0
/AA: 0          ← Must be 0
/OpenAction: 0  ← Must be 0
/ObjStm: 0      ← Must be 0
```

> ⚠️ **If ANY value is NOT zero, the sanitization failed. Delete the file immediately.**

---

## 📉 Reclaiming System Resources

After analysis, Docker and WSL 2 continue reserving RAM. Free it up:

#### Step 1: Exit the Container
```bash
exit
```

#### Step 2: Shutdown WSL 2
```powershell
wsl --shutdown
```

#### Step 3: Quit Docker Desktop
- Right-click Docker icon in system tray
- Select "Quit Docker Desktop"

Your RAM will be immediately released.

---

## 📚 Reference Tools

- **[Didier Stevens PDF Tools](https://blog.didierstevens.com/programs/pdf-tools/)** - In-depth tool documentation
- **[REMnux Documentation](https://docs.remnux.org/)** - Complete REMnux guide
- **[Malware Analysis in PDF](https://github.com/filipi86/MalwareAnalysis-in-PDF)** - Additional resources
- **[pdf-parser.py GitHub](https://github.com/smalots/pdfparser)** - Python PDF parser
- **[PDFID by Didier Stevens](https://github.com/DidierStevens/DidierStevensSuite)** - Main analysis tool

---

## ⚠️ Safety Warning

### Critical Rules:

🚫 **NEVER** open suspicious PDFs on your host machine (Windows/Mac)

🚫 **NEVER** click links or run extracted JavaScript outside the Docker container

🚫 **NEVER** move files between Docker and host without scanning

✅ **ALWAYS** interact with files via command-line tools inside Docker

✅ **ALWAYS** delete suspected malware immediately and empty Recycle Bin

✅ **ALWAYS** run in an isolated environment (VM or Docker)

### If Malware is Confirmed:

1. Exit Docker container
2. Delete the original file
3. Empty Recycle Bin
4. Run malware scanner on host (Windows Defender, Malwarebytes, etc.)
5. Consider reformatting if deeply concerned

---

## 💡 Tips & Tricks

### Speed Up Scans on Large PDFs

```bash
# For 300MB+ files, increase container memory
docker run --rm -it -m 8g -u remnux -v "%cd%":/home/remnux/files remnux/remnux-distro:noble bash
```

### Keep Analysis Logs

```bash
# Redirect all output to a log file
pdfid.py "file.pdf" | tee analysis_log.txt
```

### Batch Process Multiple PDFs

```bash
# Scan all PDFs in a folder
for f in *.pdf; do echo "=== $f ===" ; pdfid.py "$f" ; done
```

---

## 📝 License

This project is provided as-is for educational and research purposes.

---

<div align="center">

**Created for secure PDF malware analysis** | **Stay safe, analyze in isolation** 🔒


### Sanitize.py for pdf
```python
import fitz  # PyMuPDF
import sys

# Configuration
input_pdf = "SSC_Kiran_Maths_Feeded.pdf"
output_pdf = "SSC_Kiran_Maths_SANITIZED.pdf"
dpi = 150  # 150 is the Dangerzone standard. Change to 300 if you need extreme zoom resolution.

# 1. Open the original dangerous document
print(f"Opening original file: {input_pdf}")
src_doc = fitz.open(input_pdf)
total_pages = len(src_doc)

# 2. Create a brand-new, completely empty PDF document for output
clean_doc = fitz.open()

print(f"Sanitizing {total_pages} pages into flat pixel layers...")

# 3. Process every page sequentially
for page_num in range(total_pages):
    # Load the page from the original document
    page = src_doc.load_page(page_num)
    
    # Force-render the page layout into a raw RGB pixel map (removes all active code/fonts)
    pix = page.get_pixmap(matrix=fitz.Matrix(dpi / 72, dpi / 72), alpha=False)
    
    # Convert the raw pixel block data into a standardized, safe PNG byte stream in memory
    img_bytes = pix.tobytes("png")
    
    # Create a temporary one-page PDF wrapper from those clean image bytes
    img_doc = fitz.open("pdf", fitz.convert_to_pdf(img_bytes))
    
    # Insert that 100% clean, flattened page into our final clean document
    clean_doc.insert_pdf(img_doc)
    
    # Close temporary page stream to save RAM
    img_doc.close()
    
    # Print real-time progress update on the terminal line
    print(f"Processed Page {page_num + 1} of {total_pages}", end="\r")

print("\nWriting out the secure, flat PDF file...")

# 4. Save the finalized sanitized document cleanly
clean_doc.save(output_pdf, garbage=4, deflate=True)

# Close documents to release system hooks
src_doc.close()
clean_doc.close()

print(f"Success! Your air-gapped, secure document is ready at: {output_pdf}")
```

### sanitize.py for creating the png (images)
```python
import fitz  # PyMuPDF
import os

# Configuration
input_pdf = "SSC_Kiran_Maths_Feeded.pdf"
output_dir = "pacified_pdf"
dpi = 150  # Dangerzone standard for readable text (increase to 300 if needed)

# Ensure output directory exists
os.makedirs(output_dir, exist_ok=True)

# Open the document
doc = fitz.open(input_pdf)
total_pages = len(doc)

print(f"Total pages to process: {total_pages}")

# Loop through pages numerically (1, 2, 3...)
for page_num in range(total_pages):
    page = doc.load_page(page_num)
    
    # Render page to an RGB pixel map (matrix handles the DPI/resolution)
    pix = page.get_pixmap(matrix=fitz.Matrix(dpi / 72, dpi / 72), alpha=False)
    
    # Save directly as a clean PNG image (numbered 1, 2, 3...)
    actual_page_number = page_num + 1
    output_path = os.path.join(output_dir, f"page-{actual_page_number}.png")
    
    pix.save(output_path)
    
    # Print real-time progress on a single line
    print(f"Processed Page {actual_page_number} of {total_pages}", end="\r")

print("\nSanitization complete! All pages converted to raw pixel images.")
```


</div>
