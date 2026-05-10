# PDF-Malicious_Check

Here is a comprehensive `README.md` file that summarizes our entire workflow. You can copy this directly into a file on your GitHub repository.

---

# PDF Malware Analysis Lab

A step-by-step guide to identifying malicious objects, JavaScript, and obfuscated triggers in suspicious PDF files using an isolated Docker environment.

## 🛠️ Prerequisites

* **Docker Desktop** installed on Windows/Mac.
* **WSL 2 Backend** enabled (for Windows users).
* **RAM Management:** For large PDFs (300MB+), you must allocate sufficient memory to WSL 2.

### System Configuration

Create a file at `C:\Users\<YourUsername>\.wslconfig` to prevent system crashes during analysis:

```ini
[wsl2]
memory=4GB
swap=2GB
[experimental]
autoMemoryReclaim=gradual

```

*After creating this file, run `wsl --shutdown` in PowerShell.*

---

## 🚀 Environment Setup

Use the **REMnux** (Reverse-Engineering Malware Linux) Docker image. It contains all the pre-installed tools needed.

1. **Navigate** to your folder containing the suspicious PDFs.
2. **Run the container:**

```cmd
docker run --rm -it -m 4g -u remnux -v "%cd%":/home/remnux/files remnux/remnux-distro:noble bash

```

3. **Enter the workspace:**

```bash
cd /home/remnux/files

```

---

## 🔍 Analysis Workflow

### 1. High-Level Keyword Scanning (`pdfid`)

This tool scans the file for "dangerous" PDF keywords without executing them.

```bash
pdfid.py "suspicious_file.pdf"

```

**🚩 Red Flags:**

* `/JS` and `/JavaScript`: Indicates embedded scripts.
* `/ObjStm` counts the number of object streams. An object stream is a stream object that can contain other objects, and can therefor be used to obfuscate objects (by using different filters).
* `/OpenAction`: Code that runs automatically on opening.
* `/AA` (Additional Actions): Code triggered by scrolling or clicking.
* `/Encrypt`: Often used to hide malicious payloads from antivirus scanners.
  <details>
  <summary><b>To go inside the /ObjStm</b></summary>

  ### Run this in your Docker container:
  ```shell
  qpdf --qdf --object-streams=disable SSC_Kiran_English.pdf expanded_check.pdf
  ```
  ### Then scan the NEW file:
  ```shell
  pdfid.py expanded_check.pdf
  ```
  - If /JS was 0 before, but is now 1 or higher, it was hidden inside an /ObjStm.
  - ### Key Takeaway
  - For larger file it is normal (still peek inside to check the /JS) to compress the pdf
  - But for smaller file it is highly suspicious

</details>


### 2. Deep Object Inspection (`pdf-parser`)

If scripts are found, locate the specific objects containing them.

```bash
# Search for JavaScript objects
pdf-parser.py --search JS "suspicious_file.pdf"

# Decompress and read a specific object (e.g., obj 77)
pdf-parser.py -o 77 -f "suspicious_file.pdf"

```

*Note: If you get a `zlib.error`, the file may be using **Anti-Analysis** tactics (malformed headers) to crash your tools.*

### 3. Brute-Force String Extraction (`strings` + `grep`)

If the PDF structure is corrupted or too large for parsers, extract raw text patterns.

```bash
# Search for suspicious commands and obfuscated URLs
strings "suspicious_file.pdf" | grep -iE "powershell|cmd.exe|http|eval\(|unescape\("

# Look for obfuscation (e.g., weird capitalization like 'hTtP')
strings "suspicious_file.pdf" | grep -i "http" -B 5 -A 5

```

---

## 📉 Reclaiming System Resources

Because Docker and WSL 2 "reserve" RAM, you must shut them down manually after your analysis is complete.

1. **Exit the container:** Type `exit`.
2. **Shutdown WSL:** Open PowerShell and run:
```powershell
wsl --shutdown

```


3. **Quit Docker Desktop** from the system tray.

---

## ⚠️ Safety Warning

* **NEVER** open the suspicious PDF on your host Windows/Mac machine.
* **ONLY** interact with the file via the command-line tools inside the Docker container.
* **DELETE** the file and empty your Recycle Bin immediately if malicious indicators are confirmed.

---

## 📚 Reference Tools

* [Didier Stevens Suite](https://blog.didierstevens.com/programs/pdf-tools/)
* [REMnux Documentation](https://docs.remnux.org/)
* [MalwareAnalysis-in-PDF](https://github.com/filipi86/MalwareAnalysis-in-PDF)
