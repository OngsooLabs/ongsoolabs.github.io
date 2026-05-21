# OSL Secure Save

Server-authoritative save system for Unity using **Unity Gaming Services (UGS)**.  
All write operations are validated by a Cloud Code script before being committed to Cloud Save.  
Reads are performed directly via the Cloud Save SDK on the client.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Requirements](#requirements)
4. [UGS Project Setup](#ugs-project-setup)
5. [Unity Package Setup](#unity-package-setup)
6. [Cloud Code Endpoint Deployment](#cloud-code-endpoint-deployment)
7. [Remote Config Setup (Optional)](#remote-config-setup-optional)
8. [Assembly Definition Setup](#assembly-definition-setup)
9. [Using OSLSaveManager](#using-oslsavemanager)
10. [Demo Scene](#demo-scene)
11. [Issues Encountered & Solutions](#issues-encountered--solutions)
12. [Troubleshooting Reference](#troubleshooting-reference)

---

## Overview

OSL Secure Save prevents client-side cheating by routing all save operations through a Cloud Code endpoint (`osl_master_save`). The endpoint enforces:

- **Type validation** — value must match the declared type for that key
- **Range validation** — value must fall within `MinValue`/`MaxValue`
- **Delta validation** — for `AddOnly` keys, the increment cannot exceed `MaxDelta`
- **Operation validation** — each key is locked to `AddOnly`, `Replace`, or `Both`

Rules are loaded from **Remote Config** at runtime. If the Remote Config service is unavailable or the key is missing, the script falls back to a hardcoded rule set embedded in the script itself.

---

## Architecture

```
Client (Unity)
  │
  ├─ Write (gold, level, playerName, ...)
  │     └─► OSLSaveManager.AddInt / SetInt / SetString / SetBool
  │               └─► CloudCodeService.CallEndpointAsync("osl_master_save")
  │                         └─► [Server] Load rules → Validate → CloudSave.setItem()
  │
  └─ Read
        └─► OSLSaveManager.GetAsync<T> / LoadAllAsync
                  └─► CloudSaveService.Instance.Data.Player.LoadAllAsync()
                            (Default access class — where Cloud Code stores data)
```

> **Important:** Cloud Code's `cloudSaveApi.setItem()` writes to the **Default** access class, not the Protected access class. Always read from the Default class on the client.

---

## Requirements

| Requirement | Version |
|---|---|
| Unity | 2022.3 LTS or later |
| com.unity.services.cloudcode | 2.x or later |
| com.unity.services.cloudsave | 3.0.0 or later |
| com.unity.services.authentication | 2.x or later |
| com.unity.services.core | 1.x or later |

---

## UGS Project Setup

1. Open [Unity Dashboard](https://dashboard.unity3d.com/).
2. Create a project (or select an existing one).
3. Enable the following services:
   - **Cloud Code**
   - **Cloud Save**
   - **Authentication**
   - **Remote Config** (optional — see below)
4. In the Unity Editor, open **Edit → Project Settings → Services**.
5. Link your Unity project to the UGS project you created above.

---

## Unity Package Setup

Install all required packages via **Window → Package Manager → Unity Registry**:

- `Authentication` (`com.unity.services.authentication`)
- `Cloud Code` (`com.unity.services.cloudcode`)
- `Cloud Save` (`com.unity.services.cloudsave`)
- `Remote Config` (`com.unity.services.remoteconfig`) — optional

Or add them directly to `Packages/manifest.json`:

```json
"com.unity.services.authentication": "2.7.4",
"com.unity.services.cloudcode": "2.9.0",
"com.unity.services.cloudsave": "3.4.0",
"com.unity.services.remoteconfig": "4.1.0"
```

---

## Cloud Code Endpoint Deployment

### Step 1 — Configure parameters on the dashboard

1. Go to **Dashboard → Cloud Code → Scripts**.
2. Click **Create Script**, name it `osl_master_save`.
3. In the **Parameters** section, add the following two parameters:

   | Name | Type | Required |
   |---|---|---|
   | `key` | String | Yes |
   | `operation` | String | Yes |

   > **Do NOT add `value` as a typed parameter.** If you do, Cloud Code coerces it to the declared type (always String) before your script runs, breaking numeric/boolean validation. Pass `value` as an untyped argument from the client and let the script validate it directly.

4. Paste the contents of `Assets/OSL/SecureSave/CloudCode/osl_master_save.js` into the script editor.
5. Click **Save & Deploy**.

### Step 2 — Verify deployment

After deploying, call the endpoint once. The response log will contain:

```
[OSL] Script version: 1.4.0
```

If you see an older version number, the new script has not been deployed yet.

---

## Remote Config Setup (Optional)

If Remote Config is enabled, the script reads save rules from a config key named `osl_secure_save_rules`.

1. Go to **Dashboard → Remote Config → Add Key**.
2. Set:
   - **Key name:** `osl_secure_save_rules`
   - **Type:** JSON
3. Set the value to your rule set:

```json
{
  "Rules": [
    { "Key": "gold",       "Type": 0, "MinValue": 0, "MaxValue": 9999999, "MaxDelta": 10000, "AllowedOp": 0 },
    { "Key": "level",      "Type": 0, "MinValue": 1, "MaxValue": 999,     "MaxDelta": 1,     "AllowedOp": 1 },
    { "Key": "playerName", "Type": 2, "MinValue": 0, "MaxValue": 0,       "MaxDelta": 0,     "AllowedOp": 1 }
  ]
}
```

**Type enum:** `0` = Int, `1` = Float, `2` = String, `3` = Bool  
**AllowedOp enum:** `0` = AddOnly, `1` = Replace, `2` = Both

If Remote Config is not set up or returns a 400 error, the script automatically falls back to the hardcoded rules embedded in the script. The system is fully functional without Remote Config.

---

## Assembly Definition Setup

The `OSL.SecureSave.Runtime.asmdef` must reference both the Cloud Code and Cloud Save assemblies. This is not automatic — Unity requires explicit GUID references.

Open `Assets/OSL/SecureSave/Runtime/OSL.SecureSave.Runtime.asmdef` and ensure the following:

```json
{
  "references": [
    "GUID:f220a277eefae4d3aa9e31c2b43bb307",
    "GUID:be0d2672886cc45b2bcea5a5053a5347"
  ],
  "versionDefines": [
    {
      "name": "com.unity.services.cloudcode",
      "expression": "2.0.0",
      "define": "OSL_UGS_CLOUDCODE"
    },
    {
      "name": "com.unity.services.cloudsave",
      "expression": "3.0.0",
      "define": "OSL_UGS_CLOUDSAVE"
    }
  ]
}
```

| GUID | Assembly |
|---|---|
| `f220a277eefae4d3aa9e31c2b43bb307` | `Unity.Services.CloudCode` |
| `be0d2672886cc45b2bcea5a5053a5347` | `Unity.Services.CloudSave` |

The `versionDefines` enable the `#if OSL_UGS_CLOUDCODE` / `#if OSL_UGS_CLOUDSAVE` compile guards in `OSLSaveManager.cs`, keeping the package importable into projects that have not yet added these UGS packages.

---

## Using OSLSaveManager

### Initialization (required before any call)

```csharp
await UnityServices.InitializeAsync();
await AuthenticationService.Instance.SignInAnonymouslyAsync();
```

### Write operations

```csharp
// Add to an integer key (AddOnly — server enforces delta limit)
OSLSaveResult result = await OSLSaveManager.Instance.AddInt("gold", 50);

// Replace an integer key
await OSLSaveManager.Instance.SetInt("level", 5);

// Replace a string key
await OSLSaveManager.Instance.SetString("playerName", "Hero");

// Replace a bool key
await OSLSaveManager.Instance.SetBool("tutorialDone", true);

// Check result
if (!result.Ok)
    Debug.LogError(result.Error);
```

### Read operations

```csharp
// Read a single value with a default fallback
int gold   = await OSLSaveManager.Instance.GetAsync<int>("gold", 0);
int level  = await OSLSaveManager.Instance.GetAsync<int>("level", 1);
string name = await OSLSaveManager.Instance.GetAsync<string>("playerName", "Unknown");

// Load all keys at once
Dictionary<string, object> all = await OSLSaveManager.Instance.LoadAllAsync();
```

---

## Demo Scene

`Assets/OSL/SecureSave/Demo/OSLBasicUsageSample.cs` provides an interactive OnGUI demo.

**Setup:**
1. Create an empty scene.
2. Create an empty GameObject and attach `OSLBasicUsageSample`.
3. Enter Play mode.

**UI:**

| Button | Action |
|---|---|
| **Sign In (Anonymous)** | Initializes UGS, signs in anonymously, loads saved data |
| **+ Gold (+50)** | Calls `AddInt("gold", 50)` — validated server-side |
| **Level Up** | Calls `SetInt("level", currentLevel + 1)` — validated server-side |

Player nickname is auto-assigned on first login as `Player_` + first 6 characters of the UGS player ID, then saved permanently.

---

## Issues Encountered & Solutions

This section documents every issue encountered during development of this package.

---

### Issue 1 — `OSL: 'key' is required` error on every call

**Root cause:** `module.exports.params` was missing from the script entirely.  
Cloud Code requires this export to declare which parameters to bind from the client request. Without it, `params.key` is `undefined` and the null check throws immediately.

**Fix:** Add the params export at the bottom of the script:

```js
module.exports.params = {
    key:       { type: "String", required: true },
    operation: { type: "String", required: true }
};
```

---

### Issue 2 — Type validation always failed for numeric values

**Root cause:** `value` was declared in `module.exports.params` with `type: "String"`. Cloud Code's parameter binding coerces the incoming value to the declared type **before** the script body runs. This means a client sending `value: 50` (integer) was received by the script as `value: "50"` (string). The type check then correctly identified it as a string instead of a number, causing every numeric operation to fail.

**Fix:** Remove `value` from `module.exports.params` entirely. Cloud Code passes undeclared parameters through as their native JSON type. Add a `coerceValue()` function as a defensive fallback:

```js
function coerceValue(rule, value) {
    if (rule.Type === DataType.Int || rule.Type === DataType.Float) {
        if (typeof value === "string") return Number(value);
    }
    if (rule.Type === DataType.Bool) {
        if (typeof value === "string") return value === "true";
    }
    return value;
}
```

---

### Issue 3 — `logger.warn is not a function`

**Root cause:** The Cloud Code runtime exposes a `logger` object, but its available methods differ from the standard Node.js `console`. The `logger.warn` method does not exist in all Cloud Code runtime versions.

**Fix:** Add defensive wrapper functions that fall back through available methods:

```js
function getLogMethod(level) {
    if (level === "warn")  return logger.warning ?? logger.warn ?? logger.log ?? console.log;
    if (level === "error") return logger.error   ?? logger.log  ?? console.error;
    return logger.log ?? console.log;
}
const logInfo  = (...a) => getLogMethod("info")(...a);
const logWarn  = (...a) => getLogMethod("warn")(...a);
const logError = (...a) => getLogMethod("error")(...a);
```

---

### Issue 4 — Remote Config returned HTTP 400

**Root cause (a):** The `assignSettings` call was including `configType` and `key` filter parameters. The Remote Config 1.1 SDK does not accept these query parameters and rejects the request with 400.

**Root cause (b):** `context.environmentId` is not available in Cloud Code's execution context, causing a secondary error when it was passed to the API.

**Fix:** Call `assignSettings` with only the required fields:

```js
const rcResponse = await settingsApi.assignSettings({
    projectId:  context.projectId,
    userId:     context.playerId,
    attributes: {}
});
```

After retrieving all config entries, search the response for the target key manually instead of filtering at the API level. The system gracefully falls back to `HARDCODED_RULES_JSON` on any failure, so a 400 from Remote Config is non-blocking.

---

### Issue 5 — `CS0246: The type or namespace 'Item' could not be found`

**Root cause:** `OSLSaveManager.cs` referenced `Unity.Services.CloudSave.Models.Item`, but the Cloud Save assembly (`com.unity.services.cloudsave`) was not listed in the `OSL.SecureSave.Runtime.asmdef` references. Unity's assembly isolation means types from packages not explicitly referenced are invisible at compile time.

**Fix (step 1):** Add the Cloud Save assembly GUID to `.asmdef`:

```json
"GUID:be0d2672886cc45b2bcea5a5053a5347"
```

**Fix (step 2):** Change the return type of `LoadAllAsync()` and `LoadAsync()` from `Dictionary<string, Item>` to `Dictionary<string, object>`. Exposing the `Item` type across assembly boundaries requires every consumer assembly to also reference Cloud Save, which is unnecessarily restrictive. Returning `object` keeps the API self-contained.

---

### Issue 6 — Reads returned empty — data existed but was not loaded

**Root cause:** Cloud Code's `cloudSaveApi.setItem()` writes data to the **Default** access class. The client-side read code was calling `LoadAllAsync()` with `ReadAccessClassOptions` set to `ProtectedReadAccessClassOptions`. Protected and Default are separate storage namespaces in Cloud Save. The data simply did not exist in the Protected namespace.

**Fix:** Remove all access class options from read calls. The default behavior of `LoadAllAsync()` (no options argument) reads from the Default access class, which is where Cloud Code stores data:

```csharp
// Wrong — reads from Protected namespace
var data = await CloudSaveService.Instance.Data.Player.LoadAllAsync(
    new LoadAllOptions(new ProtectedReadAccessClassOptions()));

// Correct — reads from Default namespace
var data = await CloudSaveService.Instance.Data.Player.LoadAllAsync();
```

---

## Troubleshooting Reference

| Symptom | Likely Cause | Solution |
|---|---|---|
| `'key' is required` on every call | `module.exports.params` missing | Add params export to Cloud Code script |
| Numeric operations always fail validation | `value` declared as `"String"` in params | Remove `value` from params schema |
| `logger.warn is not a function` | Cloud Code logger lacks `warn` | Use `logger.warning ?? logger.log` fallback |
| Remote Config returns 400 | Extra filter params in `assignSettings` | Pass only `projectId`, `userId`, `attributes` |
| `CS0246 Item not found` | Cloud Save assembly not in asmdef | Add GUID `be0d2672886cc45b2bcea5a5053a5347` |
| Reads return empty despite data existing | Reading from wrong access class | Remove `ProtectedReadAccessClassOptions` — use plain `LoadAllAsync()` |
| Script version not updated after deploy | Old script still cached | Verify version log in Cloud Code response; redeploy |
