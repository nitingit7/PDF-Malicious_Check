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
  qpdf --qdf --object-streams=disable suspicious_file.pdf expanded_check.pdf
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


  <details>
  <summary><b>To peek into /JS and /javascript</b></summary>

  ### Decompress the PDF
  - ```shell
    qpdf --qdf --object-streams=disable your_file.pdf unpacked.pdf
    ```
  ### Locate the JavaScript Objects
  - ```shell
    pdf-parser.py --search javascript unpacked.pdf
    ```
  - Look for a line that says obj 123 0 (where 123 is the Object ID).
  ### Extract the Code
  - ```shell
    pdf-parser.py --object 123 --filter --raw unpacked.pdf > extracted_code.js
    ```
  - `--object`: Specifies the ID you found.
  - `--filter`: Decodes the "FlateDecode" compression (zlib).
  - `--raw`: Prevents the tool from adding extra formatting.
  ### What you might see inside
  - `this.exportDataObject`: Used to extract and launch hidden EXE files.
  - `util.printf("%", ...)`: A common way to trigger "Buffer Overflows" in old PDF readers.
  - `app.launchURL`: Tries to open a malicious website in your browser.
  - `getAnnots`: Used in "Side-loading" attacks to hide data in comments.
  ### The "Obfuscation" Trap
  - If the code looks like a giant wall of random letters `(var a = "x72\x65\x76...";)`, the attacker is obfuscating the script.
  - **Do not try to run this code manually** to see what it does. Malicious JS often detects it's being analyzed and changes its behavior.
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

## To Pacify the PDF file

### Step 1: Ghostscript Sanitization
- **Run this command:**
- ```bash
  gs -o "SSC_Kiran_English_FIXED.pdf" -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress "SSC_Kiran_English.pdf"
  ```
- This command is excellent for reducing file size and flattening the document structure.

### Step 2: The "Nuclear" Option (Image-Based)
- If you are still suspicious, the absolute safest method is to turn every page into an image and then rebuild the PDF. This destroys all active content
- **Run this command:**
- ```basg
  gs -sDEVICE=pdfwrite -dFILTERIMAGE -o "SSC_Kiran_English_IMAGE.pdf" "SSC_Kiran_English_FIXED.pdf"
  ```

### To print each peges into png or jpg so extra secured pdf file
- First create the folder for storing the images:
- ```bash
  mkdir ssc_2025_pages
  ```
- ### Use Ghostscript to "Print" to PNG
- **Run this command. It tells Ghostscript to turn every page into a high-quality (300 DPI) image:**
- ```bash
  gs -dNOPAUSE -dBATCH -sDEVICE=png16m -r300 -sOutputFile=ssc_2025_pages/page-%d.png "SSC_Kiran_English_FIXED.pdf"
  ```
- `-r300`: This sets the resolution to 300 DPI. This is the "sweet spot"—it makes the text very clear for reading, but doesn't make the file size impossible to handle.
- `page-%d.png`: The `%d` tells the computer to number the pages automatically `(page-1.png, page-2.png, etc.)`.
- ### To convert back to pdf:
- **Method1**: Use the pdf editor software to compile all the images into pdf
- But fist remove the metadate from the PNG images run this code:
- ```bash
  mogrify -strip ssc_2025_pages/*.png
  ```
- Now then you can proceede with convert images to pdf from pdf Editor
- **Move to a "Clean" Folder:**
- Copy the `.png` **files** from your Docker folder to a new, empty folder on your Windows desktop.

- **Methode 2**: Use the img2pdf:
- Downlaod the img2pdf first
- ```bash
  sudo apt-get update && sudo apt-get install img2pdf -y
  ```
- Then run the command to convert the images into pdf (it will take time according to the size of thie file and it will sort the pages according to the numbering form the image)
- ```bash
  img2pdf $(ls -v ssc_2025_pages/*.png) -o "SSC_Kiran_2025_SAFE.pdf"
  ```
- `ls -v` (version sort) inside the command to make sure Page 2 comes after Page 1, not after Page 100
- `Security`: This tool is strictly for images. It doesn't even have the "brain" to understand JavaScript or shellcode, so it acts as a final filter that ensures your output is 100% clean.
- 
### Crucial Verification Step
- Run `pdfid` on the new file:
- ```bash
  pdfid.py "Processed.pdf"
  ```
- The result should now show `0` for `/JS`, `/JavaScript`, and `/AA`. If it still shows numbers greater than zero, the sanitization failed, and you should delete the file immediately.

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
