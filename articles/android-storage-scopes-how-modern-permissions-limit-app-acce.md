---
layout: default
title: "Android Storage Scopes How Modern Permissions Limit App"
description: "Scoped storage (introduced in Android 10 and mandatory since Android 12) limits apps to their own sandbox directories and prevents broad read-write acce..."
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /android-storage-scopes-how-modern-permissions-limit-app-acce/
categories: [guides]
voice-checked: true
reviewed: true
score: 9
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Scoped storage (introduced in Android 10 and mandatory since Android 12) limits apps to their own sandbox directories and prevents broad read-write access to all files on external storage. Instead of requesting READ_EXTERNAL_STORAGE permission that granted access to all files, apps now use the MediaStore API for shared media or request user file picks through the Storage Access Framework. This fundamentally reduces privacy risks by preventing apps from accessing your documents, photos, or other sensitive files without explicit per-file user consent.


- Are there free alternatives: available? Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support.
- This approach provides the: best user experience, requires no permissions, and clearly communicates what data your app accesses.
- Reviewing app permissions regularly: and understanding that "storage access" no longer means "access everything" helps users maintain better privacy hygiene on their devices.
- What is the learning: curve like? Most tools discussed here can be used productively within a few hours.
- The system acts as an intermediary: presenting only files the app has legitimately created or that users have explicitly selected.
- Android 13 introduced photo: and video picker, eliminating the need for storage permissions in many scenarios.

The Evolution of Android Storage Permissions

Before Android 10, apps could request broad storage permissions that granted read and write access to nearly all files on external storage. The `READ_EXTERNAL_STORAGE` and `WRITE_EXTERNAL_STORAGE` permissions provided nearly unrestricted access to the device's shared storage, including other apps' data directories. This open model created significant privacy risks, apps could potentially read sensitive documents, photos, and downloaded files without explicit user awareness.

Google introduced scoped storage in Android 10 as a privacy-first approach, gradually making it mandatory in later versions. Under this model, each app receives an isolated sandbox containing its own files, while access to shared storage becomes mediated through specific permissions and user-granted file picks.

Understanding Scoped Storage Architecture

Scoped storage divides storage into distinct categories with different access rules:

App-specific storage provides a private directory for each app, accessible without any permissions. Files stored using `context.getExternalFilesDir()` or `context.filesDir` remain isolated to your application and get deleted when the user uninstalls the app.

Shared storage encompasses media files (images, videos, audio), documents, and downloads accessible to multiple apps. Access requires different permissions depending on the Android version and file type.

Media Store API allows querying and modifying media files (images, videos, audio) without requesting storage permissions on Android 13+. The system acts as an intermediary, presenting only files the app has legitimately created or that users have explicitly selected.

Storage Permissions by Android Version

The permission model varies significantly across Android versions:

Android 9 and Below
Apps could request `WRITE_EXTERNAL_STORAGE`, which implicitly granted `READ_EXTERNAL_STORAGE`. These permissions provided access to the entire shared storage hierarchy.

```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

Android 10 (API 29)
Scoped storage became available as an opt-in feature. Developers could set `android:requestLegacyExternalStorage="true"` in the manifest to revert to legacy storage behavior temporarily. Google Play started requiring justified use cases for broad storage access.

Android 11 (API 30)
Scoped storage became mandatory for apps targeting API 30+. The `MANAGE_EXTERNAL_STORAGE` permission replaced broad access but requires users to manually grant permission through a settings screen, Google restricts this permission to specific use cases like file managers.

Android 12+ (API 31+)
Additional granular permissions emerged. `READ_EXTERNAL_STORAGE` requires `android:maxSdkVersion="32"` for automatic granting on Android 13+. Android 13 introduced photo and video picker, eliminating the need for storage permissions in many scenarios.

Practical Code Examples

Accessing App-Specific Files

App-specific storage requires no permissions and provides secure, isolated file storage:

```kotlin
// Get app-specific external storage directory
val appDir = context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS)
val myFile = File(appDir, "my-document.pdf")

// Write without any permissions
FileOutputStream(myFile).use { output ->
    output.write(fileContent)
}
```

Using Media Store API (Android 10+)

Querying existing media files without requesting storage permissions:

```kotlin
// Android 13+ - no permission needed for reading
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    val projection = arrayOf(
        MediaStore.Images.Media.DISPLAY_NAME,
        MediaStore.Images.Media.DATE_ADDED
    )

    val cursor = contentResolver.query(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        projection,
        null,
        null,
        "${MediaStore.Images.Media.DATE_ADDED} DESC"
    )

    cursor?.use {
        while (it.moveToNext()) {
            val name = it.getString(0)
            val date = it.getLong(1)
            // Process image metadata
        }
    }
}
```

Requesting Photo Access (Android 13+)

The photo picker provides the most privacy-conscious approach:

```kotlin
// Launch system photo picker - no permission request needed
val pickMedia = registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
    uri?.let {
        // User selected photos, grant temporary access
        contentResolver.takePersistableUriPermission(
            it,
            Intent.FLAG_GRANT_READ_URI_PERMISSION
        )
        // Process the selected image
    }
}

// Select multiple images
pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly()))
```

Legacy Storage Access (File Managers Only)

Apps genuinely requiring broad file access must declare `MANAGE_EXTERNAL_STORAGE` and guide users to enable it in settings:

```xml
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
```

```kotlin
if (!Environment.isExternalStorageManager()) {
    val intent = Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION)
    intent.data = Uri.parse("package:$packageName")
    startActivity(intent)
}
```

Google typically rejects apps using this permission unless they function as file managers or have documented, essential file-handling requirements.

Best Practices for Developers

Default to app-specific storage when your app creates files. This approach requires no permissions, provides automatic cleanup on uninstall, and faces no restrictions.

Use Media Store API for accessing shared media. On modern Android versions, you can read media files without permissions while the system mediates write access through user prompts.

Implement photo picker for selecting user media. This approach provides the best user experience, requires no permissions, and clearly communicates what data your app accesses.

Test across Android versions using the `maxSdkVersion` attribute in permission declarations to ensure smooth behavior across the Android ecosystem.

Implications for Users

Modern storage permissions give users more control over their data. Users can see which apps have storage permissions through system settings, and Android prompts for permission just-in-time rather than at install. The photo picker means users share specific images intentionally rather than granting blanket photo access.

Reviewing app permissions regularly and understanding that "storage access" no longer means "access everything" helps users maintain better privacy hygiene on their devices.

Deep Dive: Scoped Storage Implementation

Understanding how scoped storage works internally helps developers make better decisions and users understand what's actually protected.

The AndroidManifest.xml Evolution

Manifest declarations control what storage an app can access:

```xml
<!-- Android 9 and below: Broad storage access -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

<!-- Android 10+: Scoped storage available -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />

<!-- Android 12+: Granular photo/video permissions -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

<!-- Legacy storage access (requires justification) -->
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
```

Each permission level grants different capabilities:

- No permission: App-specific storage only (~50MB)
- READ_MEDIA_IMAGES: Just images, requires Android 13+
- READ_EXTERNAL_STORAGE: All media files, deprecated Android 12+
- MANAGE_EXTERNAL_STORAGE: File manager access, restricted use

Access Patterns: What Apps Can Actually Do

Different access patterns have different security implications:

Reading your own files (no permissions required):
```kotlin
// App-specific directory - completely private
val appDir = context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS)
val myFile = File(appDir, "private-doc.pdf")

// Always accessible without permissions
FileInputStream(myFile).use { input ->
    // Read private data
}
```

Reading system media (MediaStore API, Android 13+):
```kotlin
// Query media without permissions using MediaStore
val mediaCollection = MediaStore.Images.Media.EXTERNAL_CONTENT_URI

val projection = arrayOf(
    MediaStore.Images.Media._ID,
    MediaStore.Images.Media.DISPLAY_NAME,
    MediaStore.Images.Media.DATE_ADDED
)

val cursor = contentResolver.query(
    mediaCollection,
    projection,
    null,
    null,
    null
)

// This query shows all images but only metadata
// App cannot read raw file bytes without user selection
```

User-selected files (Storage Access Framework):
```kotlin
// User explicitly chooses a file
val openDocumentIntent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
    addCategory(Intent.CATEGORY_OPENABLE)
    type = "application/*"  // Any file type
}

startActivityForResult(openDocumentIntent, OPEN_FILE_REQUEST_CODE)

// Result: Uri with temporary read access
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    if (requestCode == OPEN_FILE_REQUEST_CODE && resultCode == RESULT_OK) {
        val uri = data?.data ?: return

        // Can read this specific file
        contentResolver.openInputStream(uri).use { input ->
            val bytes = input?.readBytes()
        }

        // Make access permanent
        contentResolver.takePersistableUriPermission(
            uri,
            Intent.FLAG_GRANT_READ_URI_PERMISSION
        )
    }
}
```

Directories of files (Directory selection, Android 21+):
```kotlin
// User selects entire directory
val openDirIntent = Intent(Intent.ACTION_OPEN_DOCUMENT_TREE)
startActivityForResult(openDirIntent, OPEN_DIR_REQUEST_CODE)

// Result: Uri pointing to directory
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    if (requestCode == OPEN_DIR_REQUEST_CODE && resultCode == RESULT_OK) {
        val treeUri = data?.data ?: return

        // Can access all files in this directory
        val childrenUri = DocumentsContract.buildChildDocumentsUriUsingTree(
            treeUri,
            DocumentsContract.getTreeDocumentId(treeUri)
        )

        val cursor = contentResolver.query(
            childrenUri,
            arrayOf(DocumentsContract.Document.COLUMN_DOCUMENT_ID),
            null,
            null,
            null
        )

        // Enumerate files in directory
        cursor?.use {
            while (it.moveToNext()) {
                val docId = it.getString(0)
                // Access individual files
            }
        }
    }
}
```

Storage Partition Architecture

Modern Android separates storage into logical partitions:

```
/data/
 data/
    com.app1/
       files/          (app-specific, private)
       databases/      (SQLite databases)
    com.app2/
        ...

/sdcard/  (or /storage/emulated/0/)
 Documents/              (accessible via MediaStore)
 DCIM/
    Camera/            (photos accessible)
 Downloads/             (downloads area)
 .../
     com.app1/          (app-specific, scoped)
     com.app2/          (app-specific, scoped)
```

Each app's `getExternalFilesDir()` points to a private directory like `/sdcard/Android/data/com.app/files/`. This directory is inaccessible to other apps, scoped storage enforces this boundary.

Permission Checking and Handling

Developers must handle permission requests carefully. Modern best practice uses Activity Contracts (safer than deprecated requestPermissions):

```kotlin
class PhotoPermissionHandler {
    // Contract for single image selection
    private val selectImageContract =
        registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
            if (uri != null) {
                // User selected an image
                processSelectedImage(uri)
            }
        }

    fun requestPhotoAccess() {
        selectImageContract.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly())
        )
    }

    // Contract for multiple images
    private val selectMultipleImages =
        registerForActivityResult(ActivityResultContracts.PickMultipleVisualMedia()) { uris ->
            // User selected multiple images
            uris.forEach { uri ->
                processSelectedImage(uri)
            }
        }

    fun requestMultiplePhotos() {
        selectMultipleImages.launch(PickVisualMediaRequest())
    }
}
```

This approach doesn't require any permission declarations and is preferred over `READ_EXTERNAL_STORAGE`.

Understanding MediaStore Limitations

The MediaStore API provides read access to media but has limitations:

```kotlin
class MediaStoreQueries {
    fun queryAllImages(context: Context): Cursor? {
        val projection = arrayOf(
            MediaStore.Images.Media._ID,
            MediaStore.Images.Media.DISPLAY_NAME,
            MediaStore.Images.Media.DATE_ADDED,
            MediaStore.Images.Media.SIZE
        )

        return context.contentResolver.query(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            projection,
            null,
            null,
            null
        )
    }

    // MediaStore can ONLY see media it indexed
    // Private photos (in app-specific directories) are invisible
    // Raw file paths aren't exposed, only CONTENT:// URIs

    fun getImageUri(context: Context, imageId: Long): Uri {
        return ContentUris.withAppendedId(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            imageId
        )
    }

    fun accessImageData(context: Context, imageUri: Uri): ByteArray? {
        return context.contentResolver.openInputStream(imageUri)?.use { input ->
            input.readBytes()
        }
    }
}
```

MediaStore queries return metadata and URIs, not raw file paths. Apps cannot learn where files are physically stored.

The Permission Transition Challenge

Apps built for older Android versions must transition to scoped storage. Google Play enforces this:

- Android 12+: Required to use scoped storage
- Google Play: Apps targeting SDK 33+ must use scoped storage by August 2024
- Legacy apps: Targeting SDK 31 can still use legacy storage (max-sdk-version workaround)

Developers face a transitional period where they must support both old and new code paths:

```kotlin
fun readDocumentFiles(context: Context, filePath: String): ByteArray? {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        // Android 12+: Use scoped storage
        readWithScopedStorage(context, filePath)
    } else {
        // Android 11 and below: Can use direct file access
        try {
            File(filePath).readBytes()
        } catch (e: SecurityException) {
            // Fallback if permission denied
            null
        }
    }
}

fun readWithScopedStorage(context: Context, filename: String): ByteArray? {
    // Use SAF or MediaStore instead of direct file access
    // Query through DocumentsProvider or MediaStore
    return null  // Implementation depends on file type
}
```

Privacy Impact Assessment

For users, scoped storage provides measurable privacy benefits:

| Scenario | Before Scoped Storage | After Scoped Storage |
|----------|----------------------|----------------------|
| Malware installed | All photos accessible | Only selected photos |
| Spyware app | All documents readable | Nothing without user selection |
| Buggy app | Entire storage corrupted | Only own data at risk |
| Ad library | Scan all files for targeting | Cannot access data |
| Shoulder surfing | Full file browser visible | Only current app visible |

The cumulative effect significantly reduces privacy leakage.

Enterprise and IT Management

For companies deploying Android devices, scoped storage simplifies management:

```python
MDM policy for enforcing scoped storage compliance

mdm_policy = {
    "storage_access": {
        "required_targeting_sdk": 33,
        "enforce_scoped_storage": True,
        "permitted_legacy_storage": False
    },

    "app_permissions": {
        "MANAGE_EXTERNAL_STORAGE": "DENY",  # Deny blanket access
        "READ_MEDIA_IMAGES": "ASK",         # Ask for images
        "READ_MEDIA_VIDEO": "ASK",          # Ask for videos
        "READ_EXTERNAL_STORAGE": "DENY"     # Block legacy permission
    },

    "audit_storage_access": True
}
```

This policy ensures managed devices maintain privacy standards while allowing necessary functionality.

The Future: Further Granularity

Google continues reducing app permissions:

- Android 14+: More granular permissions expected
- Possible future: Per-file permissions instead of categories
- Likely: Tighter integration with privacy dashboard

As scoped storage matures, expect even more granular user control over what apps access.

Frequently Asked Questions

Who is this article written for?

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

How current is the information in this article?

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

Are there free alternatives available?

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

Can I trust these tools with sensitive data?

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

What is the learning curve like?

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

Related Articles

- [Audit Android App Permissions with ADB](/android-adb-app-permissions-audit)
- [Android App Permissions Audit Guide 2026](/android-app-permissions-audit-guide-2026/)
- [How to Audit Android App Permissions (2026)](/android-adb-app-permissions-audit/)
- [How To Audit Android App Permissions And Revoke Unnecessary](/how-to-audit-android-app-permissions-and-revoke-unnecessary-/)
- [How to Audit Android App Permissions: Step-by-Step Guide](/how-to-audit-android-app-permissions-guide/)

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
