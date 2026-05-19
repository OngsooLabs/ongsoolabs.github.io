# OSL Easy Patcher

A lightweight patch distribution system for Unity games.
Upload your patch files from the Editor, and let players automatically download only the changed files at runtime — no Addressables, no third-party SDK required.

---

## Table of Contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Editor — Uploader Window](#editor--uploader-window)
   - [Upload Tab](#upload-tab)
   - [Download Tab](#download-tab)
   - [Versions Tab](#versions-tab)
6. [Upload Backends — Detailed Setup](#upload-backends--detailed-setup)
   - [Local Folder](#1-local-folder)
   - [Google Cloud Storage](#2-google-cloud-storage)
   - [AWS S3](#3-aws-s3)
   - [Unity CCD](#4-unity-ccd)
7. [Runtime Setup](#runtime-setup)
8. [Security — Encryption & HMAC](#security--encryption--hmac)
9. [Custom Loader (IOSLLoader)](#custom-loader-ioslloder)
10. [Publishing & Installation Options](#publishing--installation-options)
11. [Frequently Asked Questions](#frequently-asked-questions)

---

## Overview

```
[Editor] Build patch files
     └─▶ OSL Uploader Window
             └─▶ Generate version.json manifest
                     └─▶ Upload to server (Local / GCS / S3 / CCD)

[Runtime] Game starts
     └─▶ OSLPatchUIController
             └─▶ OSLPatchManager
                     ├─▶ Fetch version.json from server
                     ├─▶ Compare with local version
                     ├─▶ Download changed files
                     └─▶ IOSLLoader.InitializeAsync()
```

---

## Requirements

| Item | Version |
|---|---|
| Unity | 2021.3 LTS or later (tested on 6000.x) |
| .NET Standard | 2.1 |
| TextMeshPro *(optional)* | Any — auto-detected via `OSL_TMP_PRESENT` |

---

## Installation

1. Copy the `Assets/OSL/EasyPatcher/` folder into your Unity project.
2. Unity will compile the assembly definitions automatically.
3. *(Optional)* If you use TextMeshPro, the `OSL_TMP_PRESENT` scripting define is set automatically through the assembly definition's `versionDefines`. No manual action is needed.

---

## Quick Start

> Goal: test the full flow locally in 5 minutes.

1. **Build your patch files** into any folder on your PC (e.g. `C:/MyPatchBuild`).
2. Open **Tools > OSL > Easy Patcher Uploader** from the Unity menu bar.
3. In the **Upload** tab:
   - Set **Patch Folder** to `C:/MyPatchBuild`.
   - Click **↺** to scan files.
   - Choose **Local Folder** backend and set **Destination Folder** (e.g. `C:/PatchServer`).
   - Click **Generate Manifest & Upload**.
4. Start a local HTTP server pointing at `C:/PatchServer`:
   ```powershell
   # PowerShell — serves on http://localhost:8080
   cd C:/PatchServer
   python -m http.server 8080
   ```
5. In the **Download** tab, set **Remote Base URL** to `http://localhost:8080`.
6. In your Scene, add the `OSLPatchUIController` component. Assign the Slider / Text fields and set **Remote Base URL** to `http://localhost:8080`.
7. Press **Play** — the controller will check the version and download any changed files.

---

## Editor — Uploader Window

Open via **OSL → Easy Patcher Uploader** in the Unity menu bar.

### Upload Tab

| Field | Description |
|---|---|
| **Active Slot** | Selects which version slot to work on (managed in the Versions tab). |
| **Patch Folder** | The local folder containing your built patch files. |
| **↺ (Refresh)** | Re-scans the folder and rebuilds the file tree. |
| **Patch Files tree** | Tick or untick individual files/folders to include or exclude from the manifest. Each file also has 🔒 (encrypt) and 🛡 (HMAC tamper-proof) toggles. |
| **Upload Backend** | Choose one of: Local Folder, Google Cloud Storage, AWS S3, Unity CCD. |
| **Generate Manifest & Upload** | Generates `version.json`, stages the files, then uploads everything via the selected backend. |

### Download Tab

| Field | Description |
|---|---|
| **Remote Base URL** | The HTTP(S) URL the runtime will use to fetch `version.json` and patch files. Example: `https://storage.googleapis.com/my-bucket/patches` |

### Versions Tab

Manage multiple patch version slots. Each slot stores a version string, source folder path, and the file tree state. Use **Duplicate** to branch a version for testing, and **+ Add Version Slot** to create a new one.

---

## Upload Backends — Detailed Setup

### 1. Local Folder

**Use case:** Local testing with a simple HTTP server.

| Field | Description |
|---|---|
| **Destination Folder** | Local path where files will be copied. Point an HTTP server at this folder. |

**Steps:**
1. Choose any empty folder as the destination (e.g. `C:/PatchServer`).
2. Click **Browse** and select it.
3. After uploading, start a local HTTP server:
   ```powershell
   cd C:/PatchServer
   python -m http.server 8080
   ```
4. Set the runtime URL to `http://localhost:8080`.

---

### 2. Google Cloud Storage

**Use case:** Production CDN via Google Cloud. Free tier available.

#### Step 1 — Create a GCS bucket

1. Go to [Google Cloud Console](https://console.cloud.google.com/) and sign in.
2. Navigate to **Cloud Storage → Buckets → Create**.
3. Choose a globally unique **bucket name** (e.g. `my-game-patches`).
4. Set **Location** to the region closest to your players.
5. Under **Access control**, choose **Uniform**.
6. After creating, go to the bucket's **Permissions** tab.
7. Click **Grant Access**, add the principal `allUsers`, and assign the role **Storage Object Viewer**.
   > This makes the files publicly readable so the runtime can download them without authentication.

#### Step 2 — Create a Service Account and download the JSON key

1. Navigate to **IAM & Admin → Service Accounts → Create Service Account**.
2. Give it a name (e.g. `easy-patcher-uploader`).
3. Assign the role **Storage Object Admin** (or a custom role with `storage.objects.create` + `storage.objects.delete`).
4. Click **Done**, then open the newly created service account.
5. Go to the **Keys** tab → **Add Key → Create new key → JSON**.
6. A `.json` file will be downloaded. **Keep this file secret — never commit it to Git.**

#### Step 3 — Configure the Editor window

| Field | Where to find the value |
|---|---|
| **Bucket Name** | The name you chose in Step 1 (e.g. `my-game-patches`) |
| **Object Prefix** | *(Optional)* Subfolder inside the bucket (e.g. `patches/v2/`). Leave empty to upload to the root. |
| **Service Account JSON Key** | Open the downloaded `.json` file in a text editor and paste the entire contents here. |

#### Step 4 — Set the runtime URL

```
https://storage.googleapis.com/<BUCKET_NAME>/<OBJECT_PREFIX>
```
Example: `https://storage.googleapis.com/my-game-patches/patches/v2`

---

### 3. AWS S3

**Use case:** Production CDN via Amazon Web Services.

#### Step 1 — Create an S3 bucket

1. Sign in to the [AWS Console](https://console.aws.amazon.com/) and open **S3**.
2. Click **Create bucket**.
3. Enter a **Bucket name** (e.g. `my-game-patches`) and choose your **AWS Region** (e.g. `ap-northeast-2` for Seoul, `us-east-1` for US East).
4. **Uncheck** "Block all public access" if you want public downloads, then acknowledge the warning.
5. After creating the bucket, go to **Permissions → Bucket policy** and add:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::my-game-patches/*"
       }
     ]
   }
   ```
   Replace `my-game-patches` with your actual bucket name.

#### Step 2 — Create an IAM user and access keys

1. Open **IAM → Users → Create user**.
2. Give it a name (e.g. `easy-patcher-uploader`), click Next.
3. Choose **Attach policies directly** and search for `AmazonS3FullAccess`.
   > For tighter security, create a custom policy that only allows `s3:PutObject` and `s3:DeleteObject` on your specific bucket.
4. Click **Create user**, then open the user and go to **Security credentials**.
5. Click **Create access key** → choose **Application running outside AWS** → **Create**.
6. **Copy the Access Key ID and Secret Access Key now** — the secret is only shown once.

#### Step 3 — Configure the Editor window

| Field | Where to find the value |
|---|---|
| **Access Key ID** | Copied in Step 2 (e.g. `AKIAIOSFODNN7EXAMPLE`) |
| **Secret Access Key** | Copied in Step 2. Never commit this to source control. |
| **Bucket Name** | The name from Step 1 (e.g. `my-game-patches`) |
| **Region** | AWS region identifier (e.g. `ap-northeast-2`, `us-east-1`) |
| **Object Key Prefix** | *(Optional)* Subfolder (e.g. `patches/`). Leave empty to upload to the root. |

#### Step 4 — Set the runtime URL

```
https://<BUCKET_NAME>.s3.<REGION>.amazonaws.com/<OBJECT_KEY_PREFIX>
```
Example: `https://my-game-patches.s3.ap-northeast-2.amazonaws.com/patches`

---

### 4. Unity CCD

**Use case:** Easiest option if you are already using Unity Gaming Services.

#### Step 1 — Enable CCD in your project

1. Open [Unity Dashboard](https://dashboard.unity3d.com/) and select your project.
2. Navigate to **Live Ops → Cloud Content Delivery**.
3. If not enabled, click **Get Started** to activate the service.

#### Step 2 — Create an environment and bucket

1. In the CCD section, choose or create an **Environment** (the default is `production`).
2. Click **Create Bucket**, give it a name (e.g. `game-patches`).
3. After creation, open the bucket and copy the **Bucket ID** from the URL or the detail panel.

#### Step 3 — Create an API key

1. In the Unity Dashboard, go to **Settings → Service Account API Keys**.
2. Click **Create API Key**.
3. Give it a name and assign the **Cloud Content Delivery** role.
4. Copy the generated key. **It will not be shown again.**

#### Step 4 — Find your Project ID and Environment ID

| Value | Where to find it |
|---|---|
| **Project ID** | Unity Dashboard → **Settings → General → Project ID** (also shown in Editor: *Edit → Project Settings → Services*) |
| **Environment ID** | CCD Dashboard → the environment detail page, or use `production` for the default environment |
| **Bucket ID** | CCD Dashboard → your bucket → **Bucket Details** |

#### Step 5 — Configure the Editor window

| Field | Value |
|---|---|
| **Project ID** | From Step 4 |
| **Environment ID** | From Step 4 (default: `production`) |
| **Bucket ID** | From Step 4 |
| **API Key** | From Step 3 |
| **Auto Create Release** | When enabled, a new release is created automatically after upload, making files immediately available to players. |

#### Step 6 — Set the runtime URL

After uploading and releasing, the public URL follows this format:
```
https://cdn-{projectId}.unity3dusercontent.com/client_api/v1/environments/{environmentId}/buckets/{bucketId}/releases/latest/entries
```
> **Tip:** Check the CCD dashboard for the exact CDN URL after your first release.

---

## Runtime Setup

### 1. Add OSLPatchUIController to your Scene

1. Create a new empty GameObject in your loading Scene (e.g. name it `PatchManager`).
2. Add the `OSLPatchUIController` component.
3. Assign the Inspector fields:

| Field | Description |
|---|---|
| **Remote Base URL** | The CDN / server URL where `version.json` and patch files are hosted. |
| **Hmac Key Hex** | *(Optional)* 32-byte hex key for HMAC tamper-proof verification. Required only if any file has the 🛡 flag. |
| **Loader** | Assign an `IOSLLoader` instance via `[SerializeReference]`. The bundled `OSLSimpleBundleLoader` is a serializable class — pick it from the Inspector's *type selector* drop-down on the Loader field, then expand it to edit its fields. To use a custom loader, implement `IOSLLoader` and select it the same way. |
| **Progress Slider** | A `UnityEngine.UI.Slider` that shows download progress. |
| **Status Text** | A `TMP_Text` or `UI.Text` that shows status messages. |
| **Size Text** | A `TMP_Text` or `UI.Text` that shows `12.3 MB / 45.6 MB`. |

### 2. OSLSimpleBundleLoader

`OSLSimpleBundleLoader` is a `[Serializable]` class (not a `MonoBehaviour`). On the `OSLPatchUIController` Inspector, click the **Loader** field's type selector and choose `OSLSimpleBundleLoader` to instantiate it inline. Its fields will appear nested under the Loader field.

| Field | Description |
|---|---|
| **Local Root Path Override** | *(Optional)* Override the local cache folder. Defaults to `Application.persistentDataPath/OSL/EasyPatcher`. |
| **Aes Key Hex** | *(Optional)* 32-byte hex key for decrypting AES-256 encrypted files at runtime. Must match the key used during upload. |

### 3. Patch flow at runtime

```
Start()
  └─ CheckVersionAsync()        ← downloads version.json, compares with local
       ├─ New version found
       │    └─ StartDownloadAsync()  ← downloads only changed files
       └─ Already up to date
  └─ loader.InitializeAsync()   ← loads AssetBundles / catalogs from cache
```

---

## Security — Encryption & HMAC

Keys are **never** stored in `EditorPrefs` or committed to source control. They are read from environment variables at upload time.

| Purpose | Env Variable | Format |
|---|---|---|
| AES-256 file encryption | `OSL_PATCH_AES_KEY` | 64 hex characters (32 bytes) |
| HMAC-SHA256 tamper detection | `OSL_PATCH_HMAC_KEY` | 64 hex characters (32 bytes) |

**Setting environment variables (Windows PowerShell):**
```powershell
$env:OSL_PATCH_AES_KEY  = "0011223344556677889900aabbccddeeff0011223344556677889900aabbccdd"
$env:OSL_PATCH_HMAC_KEY = "aabbccddeeff00112233445566778899aabbccddeeff00112233445566778899"
```
> Set these in the same PowerShell session where you open Unity, or configure them as system environment variables in **Windows Settings → System → Advanced → Environment Variables**.

**Generating a random key (PowerShell):**
```powershell
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Max 256) })
```

**Runtime decryption:** Supply the same AES key in the `aesKeyHex` field of `OSLSimpleBundleLoader`. The IV is prepended to each encrypted file automatically, so no manual IV management is needed.

---

## Custom Loader (IOSLLoader)

Implement `IOSLLoader` to use Addressables, a custom bundle system, or any other asset loading strategy:

```csharp
using OSL.EasyPatcher.Runtime.Core;
using System;
using System.Threading.Tasks;
using UnityEngine;

[Serializable]
public class MyCustomLoader : IOSLLoader
{
    public async Task InitializeAsync()
    {
        // Initialize your catalog / bundle system here.
        await Task.CompletedTask;
    }

    public async Task<T> LoadAssetAsync<T>(string key) where T : Object
    {
        // Load and return the asset.
        return await Task.FromResult<T>(null);
    }

    public void ReleaseAsset(string key)
    {
        // Release the asset from memory.
    }
}
```

Add your loader component to a GameObject and assign it to the **Loader** field on `OSLPatchUIController`.

---

## Publishing & Installation Options

This package supports two distribution paths. Both work from the same source tree under `Assets/OSL/EasyPatcher/`.

### Option A — Asset Store (`.unitypackage`)

1. In Unity, open **Assets → Select Folder** and select `Assets/OSL/EasyPatcher`.
2. Right-click the folder and choose **Export Package…**.
3. In the export dialog, leave **Include dependencies** unchecked, keep all entries selected, and click **Export**.
4. Save as `OSL_EasyPatcher_<version>.unitypackage`.
5. The Sample scene and `PatchTestRunner` are exported alongside the runtime/editor assemblies, so buyers can open `Sample/DemoScene.unity` immediately after import.

### Option B — UPM (Git URL or local path)

The package root is `Assets/OSL/EasyPatcher/` and contains a valid `package.json`.

- **Git URL:** in **Window → Package Manager → + → Add package from git URL…**, enter:
  ```
  https://github.com/OngsooLabs/EasyPatcherRepo.git?path=EasyPatcher/Assets/OSL/EasyPatcher
  ```
- **Local path:** in **Package Manager → + → Add package from disk…**, select the `package.json` inside `Assets/OSL/EasyPatcher/`.

When installed via UPM, the `Sample/` folder ships as a regular runtime folder so the demo scene remains directly browsable. If you prefer the official UPM `Samples~` convention (samples imported on demand), rename `Sample` to `Samples~` and add a `samples` array to `package.json`.

---

## Folder Layout

```
Assets/OSL/EasyPatcher/
  Editor/
    Upload/             ← Upload backend implementations (GCS, S3, CCD, Local)
    OSLUploaderWindow   ← Editor window entry point
    OSLPatcherSettings  ← Serializable settings / tree node data
    OSLPatcherTreeView  ← File tree view renderer
  Runtime/
    Core/               ← OSLPatchManager, PatchManifest, IOSLLoader
    Loaders/            ← OSLSimpleBundleLoader
    UI/                 ← OSLPatchUIController (MonoBehaviour)
  Sample/               ← Example scene and test runner
```

---

## Frequently Asked Questions

**Q: Where are patch files cached on the player's device?**
A: `Application.persistentDataPath/OSL/EasyPatcher/` by default. You can override this in `OSLSimpleBundleLoader.localRootPathOverride`.

**Q: What happens if the download is interrupted?**
A: `OSLPatchManager` only writes the local `version.json` after all files have been verified. On next launch, the version check will detect a mismatch and retry the download.

**Q: Can I patch non-AssetBundle files (e.g. Lua scripts, JSON configs)?**
A: Yes. The patch system transfers any file type. Loading non-bundle files is the responsibility of your `IOSLLoader` implementation.

**Q: The editor shows "Environment variable is not set" when I click Upload.**
A: You need to set `OSL_PATCH_AES_KEY` or `OSL_PATCH_HMAC_KEY` as environment variables **before** launching Unity Editor. See [Security — Encryption & HMAC](#security--encryption--hmac).

**Q: Do I need the TextMeshPro package?**
A: No. If TMP is not installed, the UI components fall back to legacy `UnityEngine.UI.Text` automatically. No code changes are needed.

**Q: Can I use multiple patch versions simultaneously?**
A: Each version slot is independent. Use the **Versions** tab to manage them. Only the slot selected in the **Upload** tab is processed when you click "Generate Manifest & Upload".
