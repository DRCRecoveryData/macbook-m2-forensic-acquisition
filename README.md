# 🍎 macbook-m2-forensic-acquisition

## 🔐 Forensic Acquisition of Apple Silicon MacBooks (M2/M3)

This repository provides documentation, step-by-step guides, and command-line examples for performing forensically sound acquisitions of internal drives on Apple MacBook devices utilizing the **M2** or **M3** Apple Silicon chips.

---

## 🎯 Project Goal

The primary challenge in acquiring data from Apple Silicon devices is the secure, signed volume structure and reliance on tools like Apple Software Restore (**asr**) for physical-to-physical imaging, often used in forensic triage or live acquisition scenarios.

This guide focuses on:

1. **Live Acquisition via `asr`** – Creating a bit-for-bit copy of the running system to an external target.

---

## 🛠 Prerequisites

To perform a successful acquisition, you will need:

- **Target Device**: Apple Silicon MacBook (M1/M2/M3)
- **External Storage**: An external drive (formatted as **APFS** or **HFS+**) with enough capacity to store the image.
- **Terminal Access**: Admin (`sudo`) privileges, with **Full Disk Access** granted to Terminal:
```

System Settings > Privacy & Security > Full Disk Access

````
- **Caffeinate**: Use the built-in `caffeinate` utility to prevent system sleep during acquisition.

---

## 📝 Guide: Live Acquisition Using `asr`

### Step 1: Create a Sparse Disk Image

A sparse image with a safety buffer is created using the following:

```bash
mkdir -p /Volumes/data/Images

hdiutil create -sectors 480724992 \
-fs APFS \
-type SPARSE \
-volname "disk3s1s1" \
-o /Volumes/data/Images/disk3s1s1.sparseimage
````

> 💡 **Note**: The final sector count (480,724,992) includes a **2,000,000 sector buffer (~1GB)** over the base requirement.

### Step 2: Attach the Image

```bash
hdiutil attach -nobrowse /Volumes/data/Images/disk3s1s1.sparseimage
```

Target volume mounted as `/dev/disk9s1`.

### Step 3: Perform the `asr` Restore

* ❌ **Failed Attempt** (wrong path):

```bash
sudo caffeinate -dimsu asr restore --source "/System/Volumes/Macintosh HD" \
  --target /dev/disk9s1 --erase --noprompt
# Error: Could not recognize as image file
```

* ✅ **Successful Attempt**:

```bash
sudo caffeinate -dimsu asr restore --source / \
  --target /dev/disk9s1 --erase --noprompt
```

### Step 4: Detach & Convert Image

```bash
hdiutil detach /dev/disk9s1

hdiutil convert /Volumes/data/Images/disk3s1s1.sparseimage \
  -format UDZO \
  -o /Volumes/data/Images/disk3s1s1.dmg
```

### Step 5: Acquisition Stats & Integrity Hashes

* **Total Time**: 12m 13.748s
* **Final Size**: 169,655,418,427 bytes
* **Speed**: 261.7 MB/sec
* **Compression**: 31.1% savings

```bash
md5 /Volumes/data/Images/disk3s1s1.dmg
# MD5 = 71eb365002639bd81b26c5cb5c6946c6

shasum -a 1 /Volumes/data/Images/disk3s1s1.dmg
# SHA-1 = c39ac5a93b5a25f33d457af4946fc536cee50f2b

shasum -a 256 /Volumes/data/Images/disk3s1s1.dmg
# SHA256 = e51f38054c383107632f4d9302a1035bd009ed8e0df3f349e1008469e36e6d46
```

---

## 🍏 Mac Forensic Imaging Source Selection Guide

This section details the preferred imaging targets across different Mac architectures.

### ⚠️ Critical Warnings

* Be familiar with APFS imaging.
* Choosing the wrong source/output may result in **incomplete or invalid images**.
* Follow your agency’s approved forensic procedures.

---

## 💾 Supported Source Options by Mac Type

### 1. Apple Silicon Macs (M1, M2, M3, M4)

> 🛑 **Physical imaging is NOT possible**

| Priority | Target Description         | Identifier | Notes                   |
| :------: | -------------------------- | ---------- | ----------------------- |
|   **1**  | Synthesized APFS Container | `disk3`    | Complete logical target |
|   **2**  | APFS Data Volume           | `disk3s5`  | "Macintosh HD - Data"   |

---

### 2. T2 Intel Macs

> 🛑 **Physical imaging is NOT possible**

| Priority | Target Description         | Identifier | Notes                   |
| :------: | -------------------------- | ---------- | ----------------------- |
|   **1**  | Synthesized APFS Container | `disk1`    | Complete logical target |
|   **2**  | APFS Data Volume           | `disk1s1`  | "Macintosh HD - Data"   |

---

### 3. Intel Macs (Non-T2, Non-Fusion)

> ✅ **Physical imaging IS possible**

| Priority | Target Description         | Identifier | Notes                 |
| :------: | -------------------------- | ---------- | --------------------- |
|   **1**  | Physical internal drive    | `disk0`    | Full-sector capture   |
|   **2**  | Synthesized APFS Container | `disk1`    | Logical fallback      |
|   **3**  | APFS Data Volume           | `disk1s1`  | "Macintosh HD - Data" |

---

### 4. Fusion Drive Systems (Intel)

| Priority | Target Description         | Identifier        | Notes                           |
| :------: | -------------------------- | ----------------- | ------------------------------- |
|   **1**  | Synthesized APFS Container | `disk2`           | Covers the entire Fusion Volume |
|   **2**  | APFS Data Volume           | `disk2s1`         | "Macintosh HD - Data"           |
|   **3**  | Physical drives            | `disk0` & `disk1` | Complex, lowest priority        |

---

## 📏 Sparse Disk Image Sizing

### 🧮 Sector Calculation

| Component     | Sector Count | Purpose                           |
| ------------- | ------------ | --------------------------------- |
| Base Sectors  | 478,724,992  | Minimum required size             |
| Extra Sectors | 2,000,000    | Safety buffer (~1.02GB)           |
| Final Sectors | 480,724,992  | Used for `hdiutil create` command |

### 🔄 Conversion to Gigabytes

> 1 sector = 512 bytes

| Sector Count | Approx Size | Description             |
| ------------ | ----------- | ----------------------- |
| 478,724,992  | ~244.08 GB  | Base requirement        |
| + 2,000,000  | ~1.02 GB    | Buffer                  |
| 480,724,992  | ~245.11 GB  | Final sparse image size |

---

## 🔎 Disk & Volume Summary

### Internal Disk (Source)

| Volume Name   | Identifier | Size    | Notes                  |
| ------------- | ---------- | ------- | ---------------------- |
| Physical Disk | `disk0`    | 251.0GB | Internal Apple Silicon |
| Macintosh HD  | `disk3s1`  | 9.2GB   | Sealed System Volume   |
| Data          | `disk3s5`  | 180.7GB | User data volume       |
| Recovery      | `disk3s3`  | 801.7MB | macOS Recovery         |

### External Disk (Destination)

| Volume Name | Identifier | Size    | Notes                   |
| ----------- | ---------- | ------- | ----------------------- |
| data        | `disk4s2`  | 962.6GB | External storage target |
| imager      | `disk4s1`  | 37.6GB  | Minimal use             |

---

## 🧰 Imaging Tools Comparison

| Tool                                         | Type        | Highlights                                                  |
| -------------------------------------------- | ----------- | ----------------------------------------------------------- |
| **Cellebrite Digital Collector**             | Commercial  | Full file system imaging, Apple Silicon/T2, live & dead box |
| **Recon ITR Live (SUMURI)**                  | Commercial  | Mac-specific, supports T2 & M-series, court-validated       |
| **LLIMAGER**                                 | Commercial  | Modern macOS focus, full logical images, fast and intuitive |
| **FUJI** (Forensic Unattended Juicy Imaging) | Open-Source | Logical live imaging for macOS, GUI, community-supported    |

---

## 💡 Tool Selection Notes

The choice of forensic imaging tool depends on:

* **Device architecture** (Apple Silicon, Intel, T2)
* **Forensic methodology** (live, dead box, volatile memory, etc.)
* **Legal constraints** and admissibility in your jurisdiction
* **Budget & licensing** — commercial tools offer support, validation, and ease of use; open-source tools are flexible, cost-effective, and academic-friendly

---
