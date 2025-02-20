# capacitor-blob-writer
A faster, more stable alternative to @capacitor/filesystem's `Filesystem.writeFile` for writing Blobs to the filesystem.

## Writing
```javascript
import {Directory} from "@capacitor/filesystem";
import {Capacitor} from "@capacitor/core";
import write_blob from "capacitor-blob-writer";

// Firstly, get a reference to a Blob. This could be a file downloaded from the
// internet, or some binary data generated by your app.

let my_video_blob = ...;

// Secondly, write the Blob to disk. The 'write_blob' function takes an options
// object and returns a Promise, which resolves once the file has been
// successfully written.

write_blob({

// The 'path' option should be a string describing where to write the file. It
// may be specified as an absolute URL (beginning with "file://") or a relative
// path, in which case it is assumed to be relative to the 'directory' option.

    path: "media/videos/funny.mp4",

// The 'directory' option is used to resolve 'path' to a location on the disk.
// It is ignored if the 'path' option begins with "file://".

    directory: Directory.Data,

// The 'blob' option must be a Blob, which will be written to the file. The file
// on disk is overwritten, not appended to.

    blob: my_video_blob,

// Fast mode vastly improves read and write speeds on the web platform. For
// files written with 'fast_mode' set to true, Filesystem.readFile will produce
// a Blob rather than a Base64-encoded string. The 'fast_mode' option is
// ignored on iOS and Android. For backwards compatibility, it defaults to
// false.

    fast_mode: true,

// If the 'recursive' option is 'true', intermediate directories will be created
// as required. It defaults to 'false' if not specified.

    recursive: true,

// If 'write_blob' falls back to its alternative strategy on failure, the
// 'on_fallback' function will be called with the underlying error. This can be
// useful to diagnose slow writes. It is optional.

// See the "Fallback mode" section below for a detailed explanation.

    on_fallback(error) {
        console.error(error);
    }
}).then(function () {
    console.log("Video written.");
});
```

## Reading
Reading a file is more complicated. Continuing the previous example, we stream a video file from disk using a `<video>` element.

```javascript
const video_element = document.createElement("video");
document.body.append(video_element);

// The video file is accessed via a URL. How this URL is obtained depends on
// the platform.

if (Capacitor.getPlatform() === "web") {

// On the web platform, begin by reading the file.

    Filesystem.readFile({
        path: "media/videos/funny.mp4",
        directory: Directory.Data
    }).then(function ({data}) {

// For files written in Fast mode, the data is retrieved as a Blob. This is not
// true of files written using Filesystem.writeFile, where the data is
// retrieved as a string. A URL is created from the Blob.

        const url = URL.createObjectURL(data);
        video_element.src = url;

// To avoid memory leaks, the URL should be revoked when it is no longer
// needed.

        video_element.onended = function () {
            video_element.remove();
            URL.revokeObjectURL(url);
        };
    });
} else {

// It is much easier to get a URL on iOS and Android.

    Filesystem.getUri({
        path: "media/videos/funny.mp4",
        directory: Directory.Data
    }).then(function ({uri}) {
        video_element.src = Capacitor.convertFileSrc(uri);
    });
}
```

## Installation

Different versions of the plugin support different versions of Capacitor:

| Capacitor  | Plugin |
|------------|--------|
| v2         | v0.2   |
| v3         | v1     |
| v4         | v1     |
| v5         | v1     |

Read the documentation for v0.2 [here](https://github.com/diachedelic/capacitor-blob-writer/tree/0.2.x). See the changelog below for breaking changes.

```sh
npm install capacitor-blob-writer
npx cap update
```

### iOS

Configure `Info.plist` to permit communication with the local BlobWriter server. This step is necessary for Capacitor v4+.
```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

Run `Product -> Clean Build Folder` within Xcode if you experience weird runtime errors (#32).

### Android

Configure `AndroidManifest.xml` to [allow cleartext](https://github.com/diachedelic/capacitor-blob-writer/issues/20) communication with the local BlobWriter server.
```xml
<application
    android:usesCleartextTraffic="true"
    ...
```

## How it works

### iOS/Android
When the plugin is loaded, an HTTP server is started on a random port, which streams authenticated PUT requests to disk, then moves them into place. The `write_blob` function makes the actual `fetch` call and handles the necessary authentication. Because browsers are highly optimised for network operations, this write does not block the UI.

I had dreamed of having the WebView intercept the PUT request and write the request's body to disk. Incredibly, neither iOS nor Android's webview are capable of correctly reading request bodies, due to [this](https://issuetracker.google.com/issues/36918490) and [this](https://bugs.webkit.org/show_bug.cgi?id=179077). Hence an actual webserver will be required for the forseeable future.

### Web (Fast mode)
In contrast to `Filesystem.writeFile`, which stores the file contents as a Base64-encoded string, `write_blob` with `fast_mode` enabled stores the file as binary data. This is possible because IndexedDB, which is already used by the `Filesystem` plugin, supports the storage of Blob objects.

### Fallback mode
On iOS and Android, there are times when `write_blob` inexplicably fails to communicate with the webserver, or the webserver fails to write the file. A fallback mode is provided, which invokes an alternative strategy if an error occurs. In fallback mode, the Blob is split into chunks and serially concatenated on disk using `Filesystem.appendFile`. While slower than `Filesystem.writeFile`, this strategy avoids Base64-encoding the entire Blob at once, making it stable for large Blobs.

There is no fallback mode on the web platform, because it is unnecessary.

## Known limitations and issues
- potential security risk (only as secure as [GCDWebServer](https://github.com/swisspol/GCDWebServer)/[nanohttpd](https://github.com/NanoHttpd/nanohttpd)), and also #12
- no `append` option yet (see #11)

## Benchmarks
The following benchmarks compare the performance and stability of `Filesystem.writeFile` with `write_blob`. See `demo/src/index.ts` for more details.

### Android (Samsung A5)

| Size          | Filesystem       | BlobWriter          |
|---------------|------------------|---------------------|
| 1 kilobyte    | 18ms             | 89ms                |
| 1 megabyte    | 1009ms           | 87ms                |
| 8 megabytes   | 10.6s            | 0.4s                |
| 32 megabytes  | Out of memory[1] | 1.1s                |
| 256 megabytes |                  | 17.5s               |
| 512 megabytes |                  | Quota exceeded[2]   |

- [1] Crash `java.lang.OutOfMemoryError`
- [2] File cannot be moved into the app's sandbox, I assume because the app's disk quota is exceeded

### iOS (iPhone 6)

| Size          | Filesystem       | BlobWriter          |
|---------------|------------------|---------------------|
| 1 kilobyte    | 6ms              | 16ms                |
| 1 megabyte    | 439ms            | 26ms                |
| 8 megabytes   | 3.7s             | 0.2s                |
| 32 megabytes  | Out of memory[1] | 0.7s                |
| 128 megabytes |                  | 3.1s                |
| 512 megabytes |                  | WebKit error[2]     |

- [1] Crashes the WKWebView, which immediately reloads the page
- [2] `Failed to load resource: WebKit encountered an internal error`

### Google Chrome (Desktop FX-4350)

| Size          | Filesystem       | BlobWriter (Fast mode) |
|---------------|------------------|------------------------|
| 1 kilobyte    | 4ms              | 9ms                    |
| 1 megabyte    | 180ms            | 16ms                   |
| 8 megabytes   | 1.5s             | 43ms                   |
| 32 megabytes  | 5.2s             | 141ms                  |
| 64 megabytes  | 10.5s            | 0.2s                   |
| 512 megabytes | Error[1]         | 1.1s                   |

- [1] `DOMException: The serialized keys and/or value are too large`

## Changelog

### v1.1.11
- Works around CapacitorHttp's patching of window.fetch.

### v1.1.10
- Adds support for Capacitor v5.

### v1.1.1
- Adds support for Capacitor v4.

### v1.1.0
- Introduces Fast mode, an opt-in feature which enables efficient Blob storage on the web platform.

### v1.0.0
- BREAKING: `write_blob` is now the default export of the capacitor-blob-writer package.
- BREAKING: `write_blob` returns a string, not an object.
- BREAKING: The `data` option has been renamed `blob`.
- BREAKING: The `fallback` option has been removed. Now, fallback mode can not be turned off. However you can still detect when fallback mode has been triggered by supplying an `on_fallback` function in the options.
- BREAKING: Support for Capacitor v2, and hence iOS v11, has been dropped.
- Adds support for Capacitor v3.
