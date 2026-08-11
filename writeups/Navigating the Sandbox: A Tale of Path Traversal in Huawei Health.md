# Navigating the Sandbox: A Tale of Path Traversal in Huawei Health

> **Researchers:** [Mehrdoost](https://github.com/Mehrdoost) & [Mi0r4](https://github.com/miora-sora)

Recently, my research partner Mior and I decided to spend some late nights dissecting the IPC (Inter-Process Communication) mechanisms of the Huawei Health app. As a central hub for fitness metrics and GPS tracking, it handles a fascinating amount of user data. We enjoy looking at how applications handle data from external sources, and during this collaborative dive, we stumbled upon an interesting boundary validation flaw: a Path Traversal vulnerability ([CWE-22](https://cwe.mitre.org/data/definitions/22.html)) in the app's file import feature.

While directory traversal always sounds alarming, context is everything in cybersecurity. Following our responsible disclosure, the vendor evaluated the practical threat level as **Low Severity** due to the app’s internal structural constraints.

Even though it wasn't a "stop the presses" critical bug, the research journey was incredibly fun, and the vulnerability serves as a perfect case study on why implicit trust in content providers can lead to unexpected behaviors. Here is the story of how we found it, how it works, and how it was fixed.

---

## Executive Summary

| Attribute | Details |
| --- | --- |
| **Target Application** | Huawei Health (`com.huawei.health`) |
| **Vendor Tracking ID** | HWPSIRT-2026-41558 |
| **Vulnerability Type** | Path Traversal (CWE-22) / Arbitrary File Write |
| **Affected Component** | `com.huawei.health.browseraction.HwSchemeFilterActivity` |
| **Assessed Severity** | Low |
| **Remediation** | Fixed in version `16.1.6.300` and subsequent releases (via Huawei AppGallery) |

> **Impact:** An attacker could craft a malicious local app to escape the intended import directory, writing or overwriting files (like `.gpx` data) elsewhere within the app's isolated private sandbox.

---

## The Discovery Phase

The story starts with how Huawei Health handles custom workout routes. Users can import external files like GPX, TCX, or KML to track their runs.

Mior and I started tracking the entry point for this feature, which led us to `HwSchemeFilterActivity`. Whenever this activity receives an `ACTION_VIEW` intent carrying a `content://` URI, it kicks off a process to read and save the incoming file.

To save the file, the app obviously needs to know what to name it. It does this by querying the external app's `ContentProvider` for the `OpenableColumns.DISPLAY_NAME`.

As we reviewed the logic, a classic question popped into our heads: **What if the external app lies about its filename?**

We noticed that the application trusted the returned `DISPLAY_NAME` implicitly. It took the unsanitized string and concatenated it straight into the app's internal storage path:

```java
// Conceptual logic of the vulnerable mechanism
String targetPath = getFilesDir() + "/route/" + filename;
File destFile = new File(targetPath);

```

There was no sanitization to strip out relative directory characters (`../`). We immediately realized that if we supplied a filename like `../../files/malicious.gpx`, the final path would resolve to `getFilesDir()/files/malicious.gpx`. It would completely bypass the intended `/route/` folder and drop the file directly into the app's root files directory.

---

## The Attack Scenario & Conceptual PoC

To prove our theory to the vendor, Mior and I didn't want to just send a theoretical write-up; we wanted to build a working proof of concept. *(Note: To comply with vendor disclosure guidelines, we are omitting raw exploit code and sharing the conceptual methodology instead).*

We built a lightweight, seemingly harmless "mock" application to act as our attacker. Here was our playbook:

1. **The Setup:** Our testing app registered a custom, malicious `ContentProvider` designed to intercept file queries.
2. **The Trigger:** We fired an intent from our test app explicitly targeting Huawei Health's `HwSchemeFilterActivity`, passing along our custom `content://` URI.
3. **The Trap:** When Huawei Health queried our provider for the file's name, we didn't send back `"morning_run.gpx"`. Instead, we sent a poisoned string: `../injected_route.gpx`.
4. **The Payload:** Alongside the poisoned name, we served a fabricated GPX file loaded with fake heart rate metrics and location coordinates.

**The result?** Huawei Health blindly accepted our path traversal string. The file stream was written exactly where we aimed it—outside the route directory, dropping our fake medical and location data directly into an adjacent folder within the app's storage.

---

## Why "Low Severity"? Understanding the Threat Model

As researchers, it is easy to get hyped about a successful exploit, but evaluating the real-world impact is crucial. The vendor classified this as **Low Severity**, and after reviewing their assessment, Mior and I completely agreed. Here is why:

* **Strict Extension Constraints:** Even though we could traverse the directory, the application tightly restricts the types of files it will process. We couldn't drop arbitrary executable code, overwrite shared preferences with `.xml` files, or achieve Remote Code Execution (RCE).
* **The Sandbox Held Strong:** The traversal was entirely trapped within Huawei Health’s internal sandbox (`/data/user/0/com.huawei.health/`). It could not break out into system directories or affect other apps on the user's phone.
* **Attack Complexity:** Exploiting this required the victim to actively install our malicious third-party application on their device first.

While a malicious app could use this to slowly fill up the user's storage with junk GPX files (causing a local denial of service) or silently corrupt a user's legitimate workout history by overwriting their route files, the overall blast radius was highly contained.

---

## The Fix: How to Sanitize Properly

Fixing this issue is relatively straightforward. We recommended the vendor implement standard path sanitization to ensure external inputs can never dictate internal directory structures.

### 1. Isolate the Filename

Never trust the full string from `DISPLAY_NAME`. Always use file utilities to extract only the final segment of the path, stripping out all `../` characters automatically:

```java
String rawName = getFileNameFromCursor(uri);
String safeName = new File(rawName).getName(); 

```

### 2. Enforce Canonical Path Validation

As a defense-in-depth measure, applications should check the resolved path before executing a file write, ensuring it still lives inside the allowed directory:

```java
File routeDir = new File(getFilesDir(), "route");
File destFile = new File(routeDir, safeName);

if (!destFile.getCanonicalPath().startsWith(routeDir.getCanonicalPath())) {
    throw new SecurityException("Directory traversal attempt blocked!");
}

```

---

## Remediation & Disclosure Timeline

We responsibly disclosed this vulnerability to the Huawei PSIRT team, who assigned it tracking number **HWPSIRT-2026-41558**. They were professional, highly responsive, and worked collaboratively with us to verify the patch.

* **May 2026:** Initial Disclosure
* **June 2026:** Vendor Confirmation & Tracking Assigned (`HWPSIRT-2026-41558`)
* **July 30, 2026:** Patch Release Date
* **Remediated Version:** Huawei Health `v16.1.6.300` and subsequent releases

> **Security Recommendation:** To ensure your application is protected against this boundary validation flaw, we strongly advise all users to update Huawei Health to version `16.1.6.300` or later via the official Huawei AppGallery.

---

## Conclusion

Working on this finding was an absolute blast. It was a great reminder for Mior and me that even mature, widely-used applications can occasionally stumble on fundamental input validation. We want to extend our thanks to the Huawei PSIRT team for their receptive communication and swift action in rolling out a patch to fortify their sandbox.

Researching together is always about the thrill of the hunt, and finding this clever little escape hatch definitely made for a great collaborative project!
