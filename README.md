## 🍎 macbook-m2-forensic-acquisition

# Forensic Acquisition of Apple Silicon MacBooks (M2/M3)

This repository provides documentation, step-by-step guides, and command line examples for performing forensically sound acquisitions of internal drives on Apple MacBook devices utilizing the **M2** or **M3** Apple Silicon chips.

-----

## 🎯 Project Goal

The primary challenge in acquiring data from Apple Silicon devices is the secure, signed volume structure and the reliance on tools like Apple Software Restore (**asr**) for physical-to-physical imaging, often used in forensic triage or live imaging scenarios.

This guide focuses on two main techniques:

1.  **Live Acquisition via asr**: Creating a bit-for-bit copy of the running system to an external target.

-----

## 🛠 Prerequisites

To perform a successful acquisition, you will need:

  * **Target Device**: An Apple Silicon MacBook (M1/M2/M3).
  * **External Storage**: An external drive with enough capacity to store the full image. This drive should be formatted as **APFS** or **HFS+** for optimal compatibility.
  * **Terminal Access**: Administrative (`sudo`) privileges on the target device. Crucially, the **Terminal** application must be granted **Full Disk Access** via `System Settings > Privacy & Security > Full Disk Access`.
  * **Caffeine**: The built-in `caffeinate` utility to prevent system sleep during the process.

-----

## 📝 Guide: Live Acquisition using `asr`

The most common method for obtaining a full-disk equivalent image in a forensically sound manner on a live macOS system is by using the **`asr`** (Apple Software Restore) utility.

### Step 1: Create a Sparse Disk Image

A **sparse disk image** (`.sparseimage`) is created to serve as the target for the acquisition. The user first created the destination directory and then the image with the specific sector count of the source disk.

```bash
quynhluong@Air-cua-Quynh ~ % mkdir -p /Volumes/data/Images
quynhluong@Air-cua-Quynh ~ % hdiutil create -sectors 480724992 \
    -fs APFS \
    -type SPARSE \
    -volname "disk3s1s1" \
    -o /Volumes/data/Images/disk3s1s1.sparseimage
```

  * **Log Output:**
    ```
    created: /Volumes/data/Images/disk3s1s1.sparseimage
    ```

### Step 2: Attach the Image and Identify Target

The newly created image was attached and the volume identifier was noted for the target of the `asr` command.

```bash
quynhluong@Air-cua-Quynh ~ % hdiutil attach -nobrowse /Volumes/data/Images/disk3s1s1.sparseimage
```

  * **Target Volume Identification:** The sparse image volume was successfully mounted as **/dev/disk9s1** under the name `disk3s1s1`.

### Step 3: Perform the `asr` Restore (Acquisition)

The `asr restore` utility was executed using `caffeinate` to prevent system sleep.

#### Attempt 1: Using the Volume Name as Source (Failed)

The initial attempt to use the volume name path for the source failed as it was "Could not recognize" the path as an image file.

```bash
quynhluong@Air-cua-Quynh ~ % sudo caffeinate -dimsu asr restore --source "/System/Volumes/Macintosh HD" --target /dev/disk9s1 --erase --noprompt
...
Could not recognize “/System/Volumes/Macintosh HD” as an image file
```

#### Attempt 2: Using the Root Filesystem Path as Source (Successful)

The correct method of using the root filesystem path (`/`) was then successfully executed.

```bash
quynhluong@Air-cua-Quynh ~ % sudo caffeinate -dimsu asr restore --source / --target /dev/disk9s1 --erase --noprompt
...
Restore completed successfully.
```

### Step 4: Detach and Convert the Image

After the acquisition completed, the sparse image volume was detached, and then converted to a compressed, read-only DMG (**UDZO**) for final storage and verification.

```bash
# Detach the volume
quynhluong@Air-cua-Quynh ~ % hdiutil detach /dev/disk9s1

# Convert to a compressed DMG
quynhluong@Air-cua-Quynh ~ % hdiutil convert /Volumes/data/Images/disk3s1s1.sparseimage \
    -format UDZO \
    -o /Volumes/data/Images/disk3s1s1.dmg
```

### Step 5: Acquisition Statistics and Verification Hashes

The conversion process provided detailed statistics and the necessary cryptographic hashes for the Chain of Custody document.

  * **Acquisition Statistics (from conversion log):**

      * **Total Acquisition Time**: $12\text{m } 13.748\text{s}$.
      * **Final File Size**: $169,655,418,427\text{ bytes}$.
      * **Acquisition Speed**: $261.7\text{ MB/sec}$ (MB/sec).
      * **Compression Efficiency**: $31.1\%$ savings achieved.

  * **Acquisition Integrity Hashes:**

    ```bash
    quynhluong@Air-cua-Quynh ~ % md5 /Volumes/data/Images/disk3s1s1.dmg
    MD5 = 71eb365002639bd81b26c5cb5c6946c6

    quynhluong@Air-cua-Quynh ~ % shasum -a 1 /Volumes/data/Images/disk3s1s1.dmg
    SHA-1 = c39ac5a93b5a25f33d457af4946fc536cee50f2b
    
    quynhluong@Air-cua-Quynh ~ % shasum -a 256 /Volumes/data/Images/disk3s1s1.dmg
    SHA256 = e51f38054c383107632f4d9302a1035bd009ed8e0df3f349e1008469e36e6d46
    ```

    The **MD5 hash** of **`71eb365002639bd81b26c5cb5c6946c6`** must be recorded as the initial integrity check.

-----

## 🔎 Disk and Volume Summary (From `diskutil list` and `df -h`)

### Internal Disk (Source - `disk0` / `disk3`)

| Volume Name | Identifier | Size | Usage/Notes |
| :--- | :--- | :--- | :--- |
| **Physical Disk** | `disk0` | $251.0\text{ GB}$ | Internal Apple Silicon drive. |
| **Macintosh HD** | `disk3s1` | $9.2\text{ GB}$ | Sealed System Volume. Used $8.5\text{ GiB}$ ($16\%$ capacity). |
| **Data** | `disk3s5` | $180.7\text{ GB}$ | User-mutable data volume. Used $168\text{ GiB}$ ($79\%$ capacity). |
| **Recovery** | `disk3s3` | $801.7\text{ MB}$ | Recovery volume. |

### External Storage (Target Destination - `disk4`)

| Volume Name | Identifier | Size | Usage/Notes |
| :--- | :--- | :--- | :--- |
| **data** | `disk4s2` | $962.6\text{ GB}$ | External destination volume. Used $504\text{ GiB}$ ($57\%$ capacity). |
| **imager** | `disk4s1` | $37.6\text{ GB}$ | External volume. Used $53\text{ MiB}$ ($1\%$ capacity). |

-----

## 📚 Resources and Documentation

  * Apple Developer Documentation on asr: \[Link to ASR documentation]
  * Guidance on APFS and Volume Structures: \[Link to relevant forensic article]
