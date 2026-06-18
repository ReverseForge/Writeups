# Telegram file format 0day analysis

```
news:
https://restoreprivacy.com/telegram-for-android-hit-by-zero-day-evilvideo-exploit/
```

the only patch for file mime type check and  handle  is these function
```
before patch:

 public static boolean openForView(File f, String fileName, String mimeType, final Activity activity, Theme.ResourcesProvider resourcesProvider) {
```

```
after patch 

public static boolean openForView(File f, String fileName, String mimeType, final Activity activity, Theme.ResourcesProvider resourcesProvider, boolean restrict) {
```

in first line :
```
        if (f != null && f.exists()) {
            String realMimeType = null;
            Intent intent = new Intent(Intent.ACTION_VIEW);
```

The first line checks whether the file exists or not After that, the mimetype value is set to null . after that for video player and intent activity on android 

example
```
https://stackoverflow.com/questions/10430073/android-intent-action-view
```
after that check for file type after "." string
```
            int idx = fileName.lastIndexOf('.');
            if (idx != -1) {
                String ext = fileName.substring(idx + 1);
```

in new version the telegram check for some hashcodes for ext of substring of filename -> filetype (cooool  we can do it and add some hashcode maybe?)


```
after patch before patch these condition doesn't exist

                int h = ext.toLowerCase().hashCode();
                if (restrict && (h == 0x17a1c || h == 0x3107ab || h == 0x19a1b || h == 0xe55 || h == 0x18417)) {
                    return true;
                }
```


in 14.3 version after the ext and get the filetype index

```
                realMimeType = myMime.getMimeTypeFromExtension(ext.toLowerCase());
                if (realMimeType == null) {
                    realMimeType = mimeType;
```
canRequestPackageInstalls from google documents:
```
Checks whether the calling package is allowed to request package installs through package installer. Apps are encouraged to call this API before launching the package installer via intent Intent.ACTION_INSTALL_PACKAGE. Starting from Android O, the user can explicitly choose what external sources they trust to install apps on the device. If this API returns false, the install request will be blocked by the package installer and a dialog will be shown to the user with an option to launch settings to change their preference. An application must target Android O or higher and declare permission Manifest.permission.REQUEST_INSTALL_PACKAGES in order to use this API.
```

after that as you can see the video send the request for installation of the app in here just check for string license and after that install it 

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O && realMimeType != null && realMimeType.equals("application/vnd.android.package-archive") && !ApplicationLoader.applicationContext.getPackageManager().canRequestPackageInstalls()) {
                AlertsCreator.createApkRestrictedDialog(activity, resourcesProvider).show();
                return true;
```

so what the vulnerability?


in these condition its dont check any restriction and in the patch they add condition check for restrict

```
          if (realMimeType != null && realMimeType.equals("application/vnd.android.package-archive")) {
                if (restrict) return true; //here
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O && !ApplicationLoader.applicationContext.getPackageManager().canRequestPackageInstalls()) {
                    AlertsCreator.createApkRestrictedDialog(activity, resourcesProvider).show();
                    return true;
                }
```

so what is restrict the hash code check is the restrict of media check


```
                int h = ext.toLowerCase().hashCode();
                if (restrict && (h == 0x17a1c || h == 0x3107ab || h == 0x19a1b || h == 0xe55 || h == 0x18417)) {
                    return true;
                }
```

so in the vulnerable version anyone can request to install package but in new version they add restrict for some part of code if the restrict is true and the hash checks is valid they can request the restrict set static by program and the attacker can't access to it 


in the vulnerable version they call these like these without any check 

```
    public static boolean openForView(TLRPC.Document document, boolean forceCache, Activity activity) {
        String fileName = FileLoader.getAttachFileName(document);
        File f = FileLoader.getInstance(UserConfig.selectedAccount).getPathToAttach(document, true);
        return openForView(f, fileName, document.mime_type, activity, null);// 这里(here)
```

the file gonna be load without any problem and can send request for install packages ```these Feature use for players ```



first if the video can't play they call the openforview :)



```
                    builder.setMessage(LocaleController.getString("CantPlayVideo", R.string.CantPlayVideo));
                    builder.setPositiveButton(LocaleController.getString("Open", R.string.Open), (dialog, which) -> {
                        try {
                            AndroidUtilities.openForView(currentMessageObject, parentActivity, resourcesProvider);
```


so how the attacker can attach file like video?

in the vulnerable version to check is these the video or not its have some checks 
the checks only for it use 
```
       boolean isAnimated = false;
        boolean isVideo = false;
for (int a = 0; a < document.attributes.size(); a++) {
            TLRPC.DocumentAttribute attribute = document.attributes.get(a);
            if (attribute instanceof TLRPC.TL_documentAttributeVideo) {
                if (attribute.round_message) {
                    return false;
                }
                isVideo = true;
                width = attribute.w;
                height = attribute.h;
            } else if (attribute instanceof TLRPC.TL_documentAttributeAnimated) {
                isAnimated = true;
```
in the file only thing it need to introduce him self as an video is the control the attributes and cool part of it is handling the width and height. the only thing file need its the have the TLRPC document and be animated .



after patch they just add some hash checks


```
        if (filename != null) {
            int index = filename.lastIndexOf(".");
            if (index >= 0) {
                String ext = filename.substring(index + 1);
                switch (ext.toLowerCase().hashCode()) {
                    case 0x17a1c: case 0x3107ab: case 0x19a1b:
                    case 0xe55:   case 0x18417:  case 0x184fe:
                    case 0x18181:
                        return false;
```

