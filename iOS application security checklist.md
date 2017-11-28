# iOS application security checklist
## Configuration
### CF-001 Check Position Independent Executable
### Overview
Position-Independent Executable (PIE) is an exploit mitigation security feature that allows an application to take full advantage of ASLR (Address Space Layout Randomization).

**The app is PIE if and only if the main executable and all its dependencies were built as PIE.**
### Code review
In *Xcode*, go to *Build Settings*, verify that the value of *Generate Position-Dependent Code* and *Generate Position-Dependent Executable* are set to "No".
### Manual assessment
Use `otool -hv` to check whether PIE is enabled or not. For example, the app is PIE enabled if:
```
$ otool -hv /path/to/MyApp.app/MyApp
MyApp:
Mach header
magic   cputype cpusubtype caps filetype  ncmds  sizeofcmds   flags
MH_MAGIC  ARM    V7        0x00  EXECUTE    23        2372     NOUNDEFS DYLDLINK TWOLEVEL PIE
```
### Expected result
PIE is enabled.

---
### CF-002 Check Stack smashing protection
### Overview
Additionally to ASLR and PIE, a further binary protection that iOS application can apply at compile time is stack-smashing protection. Enabling stack-smashing protection causes a known value or “canary” to be placed on the stack directly before the local variables to protect the saved base pointer, saved instruction pointer, and function arguments. The value of the canary is then verified when the function returns to see whether it has been overwritten. The LLVM compiler uses a heuristic to intelligently apply stack protection to a function, typically functions using character arrays.

**Stack-smashing prototoolection is enabled by default since *Xcode* 6.**
### Code review
N/A
### Manual assessment
Use `otool –I –v` to check whether Stack smashing protection is enabled or not. If stack smashing
protection is compiled into the application, two undefined symbols will be present: `___stack_chk_fail` and
`___stack_chk_guard`.
For example:
```
$ otool -I -v myapp | grep stack
0x00001e48 97 ___stack_chk_fail
0x00003008 98 ___stack_chk_guard
0x0000302c 97 ___stack_chk_fail
```
### Expected result
Stack smashing protection is enabled

---
### CF-003 Check Automatic Reference Counting
### Overview
Automatic Reference Counting(ARC) is a compiler feature that provides automatic memory management of Objective-C objects.

ARC was introduced in iOS SDK version 5.0 to move the responsibility of memory management and reference counting from the developer to the compiler. As a side effect ARC also offers some security benefits because it reduces the likelihood of developers’ introducing memory corruption (specifically, object use-after-free and double-free) vulnerabilities into applications.
### Code review
In *Xcode*, go to *Build Settings*, verify that the value of *Objective-C Automatic Reference Counting* is set to "Yes".
### Manual assessment
Use `otool –I –v <app name> | grep "_objc_release"` to check whether Automatic Reference Counting is enabled or not.
Symbols that indicate the presence of ARC:
```
_objc_retainAutoreleaseReturnValue, _objc_autoreleaseReturnValue, _objc_storeStrong, _objc_retain, _objc_release, _objc_retainAutoreleaseReturnValue
```
### Expected result
Automatic Reference Counting is enabled.

---
## Data Storage
### DS-001 Check Data Protection API
### Overview
Individual files and keychain items can be encrypted using the Data Protection API, which uses a key derived from the device passcode. Consequently, when the device is locked, items encrypted using the Data Protection API in this way will be inaccessible, and upon unlocking the device by entering the passcode, protected content becomes available.
### Code review
Check that the complete protection(`NSFileProtectionComplete` or `NSDataWritingFileProtectionComplete`) is enabled.
### Manual assessment
Try to access the file when the device is locked. The following shows an attempt to access the file while the device is locked:
```
$ ls -al Documents/ total 372
drwxr-xr-x 2 mobile mobile 102 Jul 20 15:24 ./
drwxr-xr-x 6 mobile mobile 204 Jul 20 15:23 ../
-rw-r--r-- 1 mobile mobile 379851 Jul 20 15:24 wahh-live.pdf
$ strings Documents/wahh-live.pdf
strings: can't open file: Documents/wahh-live.pdf
(Operation not permitted)
```
### Expected result
Complete protection should be used for files.

---
### DS-002 Check SQLite database
### Overview
SQLite database is full supported by iOS, but the data stored in it is not encrypted. So sensitive data must not be stored in it without encryption or use encrypted databases like SQLCipher.
### Code review
Check the data stored in SQLite. For example, `NSSQLiteStoreType` for Core Data and `sqlite3_exec` for SQLite 3.
### Manual assessment
Check no sensitive data is stored in plain text in SQLite database under `/var/mobile/Containers/Data/Application/{GUID}/`
### Expected result
No sensitive data is stored in SQLite database without protection

---
### DS-003 Check property list files
### Overview
Property lists are used as a form of data storage and are commonly used in the Apple ecosystem under the .plist file extension. The format is similar to XML and can be used to store serialized objects and key value pairs. Application preferences are often stored in the `/var/mobile/Containers/Data/Application/{GUID}/Library/Preferences` directory as property lists using the `NSDefaults` class.

**Data stored using NSUSerDefaults are getting saved in simple plist -in binary format without any encryption.**
### Code review
Check the usage of property list in source code, such as `ofType: @"plist"`,`NSUserDefaults` and so on.
### Manual assessment
Use tools like *plist Editor* to Check no sensitive data in property list files(\*.plist) under `/var/mobile/Containers/Data/Application/{GUID}/`.
### Expected result
No sensitive data is saved in property list files without protection.

---
### DS-004 Check keyboard cache
### Overview
To improve the user experience, iOS attempts to customize the autocorrect feature by caching input that is typed into the device's keyboard. Almost every non-numeric word is cached on the file system in plaintext in the keyboard cache file located in `/var/mobile/Library/Keyboard`.
### Code review
Check the source code to prevent sensitive data from being populated into the cache by either marking a field as a secure field using the `secureTextEntry` property or by explicitly disabling autocorrect by setting the `autocorrectionType` property to `UITextAutocorrectionTypeNo`.

Here is an example of how to do this:
```
securityAnswer.autocorrectionType = UITextAutocorrectionTypeNo;
securityAnswer.secureTextEntry = YES;
```
### Manual assessment
Check no sensitive data in keyboard cache under `/var/mobile/Library/Keyboard/`.
### Expected result
No sensitive data is stored in keyboard cache.

---
### DS-005 Check HTTP response cache
### Overview
To display a remote website, an iOS application often uses a `UIWebView` to render the HTML content. A `UIWebView` object uses `WebKit`, the same rendering engine as MobileSafari, and just like MobileSafari a `UIWebView` can cache server responses to the local file system depending on how the URL loading is implemented.
### Code review
Check the source code to identify the server response which contains sensitive data. HTTP cache can be cleared and removed by using:
```
[[NSURLCache sharedURLCache] removeAllCachedResponses];

-(NSCachedURLResponse *)connection:(NSURLConnection *)connection
willCacheResponse:(NSCachedURLResponse *)cachedResponse
{
    NSCachedURLResponse *newCachedResponse=cachedResponse;
    if ([[[[cachedResponse response] URL] scheme] isEqual:@"https"]) {
        newCachedResponse=nil;
    }
    return newCachedResponse;
}
```
### Manual assessment
Check the cache data in `/var/mobile/Containers/Data/Application/{GUID}/Library/Caches/Cache.db`
### Expected result
Sensitive data is not cached by Cache.db.

---
### DS-006 Check Cookies.binarycookies
### Overview
Binary cookies can be created by the URL loading system or webview as part of an HTTP request in a similar way to standard desktop browsers. The cookies get stored on the device’s file system in a cookie jar and are found in the `/var/mobile/Containers/Data/Application/{GUID}/Library/Cookies` directory in the `Cookies.binarycookies` file. As the name suggests, the cookies are stored in a binary format but can be parsed using the [BinaryCookieReader.py](http://securitylearn.net/wpcontent/uploads/tools/iOS/BinaryCookieReader.py).
### Code review
Check the cookie usage. For example, the usage of  `NSHTTPCookie` and `NSHTTPCookieStorage`.
### Manual assessment
Use *BinaryCookieReader.py* to check the data stored in Cookie in  `/var/mobile/Containers/Data/Application/{GUID}/Library/Cookies/Cookies.binarycookies`.
### Expected result
No sensitive data is stored in Cookie without protection.

---
## Data Transmission
### DT-001 Check sensitive data is transmitted via encrypted channel
### Overview
It is imperative that the communications between iOS applications and servers are performed in a proper encrypted channel.

**In iOS 9, Apple introduced "App Transport Security," or ATS. This defaults apps to requiring an HTTPS connection, and returning an error for non-HTTPS connections.**
### Code review
Check the settings of `NSAppTransportSecurity` in `Info.plist`.
Check the source code to identify all the communication with the server. For example, check the usage of `NSURLConnection`, `NSURLSession`, `AFHTTPRequestOperationManager`, and so on.
### Manual assessment
Use proxy tools like *Burp Suite* to monitor all the communication with the server.
### Expected result
HTTPS should be used to transfer sensitive data to the server.

---
### DT-002 Check the implementation of HTTPS
### Overview
Certificates should be checked properly. Developers may disable the certificate validation or override the behavior to accept any certificate.

**If the certificate is issued by a trusted CA, the certificate will be validated by iOS.**
### Code review
Check the implementation of certificate validation:
1. Trusted CA certificate. Check the HTTPS protocol is used.
2. Self signed certificate. Check the certificate is validated properly:
    - expired certificates
    - mismatching hostnames
    - expired root certificates
    - any root certificate

### Manual assessment
TBD
### Expected result
Certificate is validated properly.

---
## Data Validation
### DV-001 Check URL schemes
### Overview
iOS URL Schemes in general allow one App to be opened by other Apps, or essentially inter-app communication. Specific actions can be defined to not only open a URL, but populate what it is you’d like to search, for example coordinates, local donut shops, and much more.

### Code review
Check `openURL` function(`handleOpenURL` is deprecated).
### Manual assessment
N/A
### Expected result
Proper validation is performed for the `url` and `sourceApplication`.

---
### DV-002 Check UIWebView
### Overview
`UIWebView` is the iOS-rendering engine for displaying web content, it supports a number of different file formats, including:
- HTML
- PDF
- RTF
- Office Documents (doc, xls, ppt)
- iWork Documents (Pages, Numbers, and Keynote)

Consequently, a web view is also a web browser and can be used to fetch and display remote content. As would be expected of a web browser, web views also support JavaScript, allowing applications to perform dynamic, client-side scripting.
### Code review
Check the content of UIWebView which may loaded by `load`, `loadHTMLString`, `loadRequest` and so on. And if JavaScript is enabled, check the `stringByEvaluatingJavaScript` and `evaluateJavaScript`.
### Manual assessment
N/A
### Expected result
Data is validated and no malicious code could be executed in `UIWebView`.

---
### DV-003 Check XML Processing
### Overview
The iOS SDK provides two options for parsing XML: the `NSXMLParser` and `libxml2`. However, a number of popular third-party XML parser implementations are also widely used in iOS apps.

**Parsing of external XML entities is not enabled by default in the NSXMLParser class, but was enabled by default in the LibXML2 parser up to version 2.9.**
### Code review
Check setting of `setShouldResolveExternalEntities` option.
### Manual assessment
N/A
### Expected result
XXE attack should be prevented.

---
### DV-004 Check SQL injection protection
### Overview
Much like when SQL is used within web applications, if SQL statements are not formed securely, apps can find themselves vulnerable to SQL injection.

To perform data access on client-side SQLite databases, iOS provides the built-in SQLite data library. If using SQLite, the application will be linked to the `libsqlite3.dylib` library.
### Code review
Check the usage of SQLite.
### Manual assessment
N/A
### Expected result
Prepared statement should be used or data in SQL query is validated.

---
### DV-005 Check file system interaction
### Overview
Two main classes are used for file handling in the iOS SDK: `NSFileManager` and `NSFileHandle`.
### Code review
Check the usage of `NSFileManager` and `NSFileHandle`.
### Manual assessment
N/A
### Expected result
File path is validated and path for traversal attack is prevented.

---
## Logging
### LG-001 Check NSLog statements
### Overview
Logging in an iOS application is typically performed using the `NSLog` method that causes a message to be sent to the Apple System Log (ASL). These console logs can be manually inspected using the *Xcode* device's application.
### Code review
Check the usage of `NSLog`.
### Manual assessment
Use tools like *iTools* or *IOS Application Security Part* to check the logs.
### Expected result
No sensitive data is logged in Apple System Log.

---
### LG-002 Check crash logs
### Overview
Logging can prove to be a valuable resource for debugging during development. However, in some cases it can leak sensitive or proprietary information, which is then cached on the device until the next reboot.
### Code review
N/A
### Manual assessment
1.  Plug in the device and open *Xcode*.
2.  Open the *Organizer* window and select the *Devices* tab.
3.  Under the *DEVICES* section in the left column, expand the listing for the device.
4.  Select *Device Logs* to see crash logs.

### Expected result
No sensitive data is logged in crash logs.

---
## Sensitive Data
### SD-001 Check data cached in keyboard
### Overview
iOS devices utilize a feature called Auto Correction to populate a local keyboard cache on the device. The keyboard cache is designed to autocomplete the predictive common words. But it records everything that a user types in text fields. This should be disabled for any sensitive fields(e.g. credit card information).
### Code review
Check the source code for sensitive `UITextField` and `UISearchBar`, mark them as secure fields and disable auto-correction:
```
fieldName.secureTextEntry = YES;
fieldName.autocorrectionType = UITextAutocorrectionTypeNo;
```
### Manual assessment
Check data in keyboard cache in `/var/mobile/Library/Keyboard/dynamic-text.dat`.
### Expected result
Sensitive data is not cached in keyboard.

---
### SD-002 Check data in the Clipboard
### Overview
`UIPasteboard` helps to share data within or between applications. There are two default system pasteboards: `UIPasteboardNameGeneral` and `UIPasteboardNameFind`. There are no access controls or restrictions for the system pasteboard, so if there is any sensitive data should be displayed in `UITextField` but should not accessed outside the application, copy/paste should be disabled.

**But starting in iOS 10, the Find pasteboard (identified with the UIPasteboardNameFind constant) is unavailable.**
### Code review
Check the source code for any `UITextField` which includes sensitive data. Copy/paste should be disabled. For example:
```
-(BOOL)canPerformAction:(SEL)action withSender:(id)sender {
    if (action == @selector(cut:) || action == @selector(copy:))
    return NO;
else
    return YES;
}
```
### Manual assessment
Perform long press on any text field which includes sensitive data.
### Expected result
Copy/paste should be disabled for such text field.

---
### SD-003 Check data sent to third party
### Overview
3rd party may collect data from the mobile application.
### Code review
Check the source code to identify the data sent to 3rd party.
### Manual assessment
Use proxy tools like *Burpsuite* to verify the data collected by 3rd parties.
### Expected result
No sensitive data is collected by 3rd parties.

---
### SD-004 Check snapshots
### Overview
When an application is suspended in the  background, iOS will take a snapshot of the  app and store it in the application caches  directory.
### Code review
Application removes sensitive data from view when go to background.
Using "hidden" attribute:
```
- (void)applicationDidEnterBackground:(UIApplication *)application {
    viewController.creditcardNumber.hidden = YES;
}
```
### Manual assessment
Review any view includes sensitive data, then press Home key to suspend the application. Check the snapshots(every click) under `/var/mobile/Containers/Data/Application/{GUID}/Library/Caches/Snapshots/`.
### Expected result
No sensitive data will be cached by snapshots.

---
### SD-005 Check GeoLocation usage
### Overview
Apple provides a means of accessing the device's geolocation features using the Core Location framework. Device coordinates can be determined using GPS, cell tower triangulation, or Wi-Fi network proximity. When using geolocation data, developers should consider two main privacy concerns: how and where data is used and the requested accuracy of coordinates.
### Code review
When using `CLocationManager`, an app can request accuracy using the `CLLocationAccuracy` class that offers the following constants:
- kCLLocationAccuracyBestForNavigation
- kCLLocationAccuracyBest
- kCLLocationAccuracyNearestTenMeters
- kCLLocationAccuracyHundredMeters
- kCLLocationAccuracyKilometer
- kCLLocationAccuracyThreeKilometers

Check the source code to identify the level of location accuracy.

### Manual assessment
Check the client side storage and data sent to server.
### Expected result
Application uses suitable level of accuracy and no additional location information requested.
Application does not disclose location data.

---
### SD-006 Check Keychain
### Overview
The iOS keychain is an encrypted container used for storing sensitive data such as credentials, encryption keys, or certificates. The following list describes the available accessibility protection classes for keychain items:
- kSecAttrAccessibleAlways
- kSecAttrAccessibleWhenUnlocked
- kSecAttrAccessibleAfterFirstUnlock
- kSecAttrAccessibleAlwaysThisDeviceOnly
- kSecAttrAccessibleWhenUnlockedThisDeviceOnly
- kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
- kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly

### Code review
Check the Keychain usage in source code to identify the keychain accessibility.
### Manual assessment
N/A
### Expected result
The strictest accessibility should be used if no specific reason.
