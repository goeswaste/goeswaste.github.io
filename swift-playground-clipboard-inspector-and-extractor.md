---
title: Swift Playground Clipboard Inspector and Extractor
date: 2026-09-08 23:38:55
---

````markdown
Yes. The modern replacement requires specifying the class you expect to decode. Since a WebArchive's root object is typically a dictionary containing mixed Foundation types, `unarchivedObject(ofClasses:from:)` is more appropriate than `unarchivedObject(ofClass:from:)`.

Here’s the updated Swift Playground version:

:::writing{variant="document" id="58321" title="Updated Swift Playground Code"}
```swift id="7upots"
import SwiftUI
import UIKit

struct ContentView: View {
    @State private var output = "Ready."
    
    var body: some View {
        VStack(spacing: 16) {
            Text("Clipboard Inspector")
                .font(.title)
                .bold()
            
            Button("Extract Clipboard Image") {
                output = extractClipboardImage()
            }
            .buttonStyle(.borderedProminent)
            
            ScrollView {
                Text(output)
                    .font(.system(.body, design: .monospaced))
                    .frame(maxWidth: .infinity, alignment: .leading)
                    .padding()
            }
        }
        .padding()
    }
    
    private func extractClipboardImage() -> String {
        let pasteboard = UIPasteboard.general
        
        guard let data = pasteboard.items.first?[
            "com.apple.webarchive"
        ] as? Data else {
            return "No com.apple.webarchive data found."
        }
        
        print("WebArchive size: \(data.count) bytes")
        
        do {
            // Modern replacement for:
            // NSKeyedUnarchiver.unarchiveTopLevelObjectWithData(data)
            //
            // WebArchives can contain several different Foundation
            // object types, so allow the relevant classes.
            let allowedClasses: [AnyClass] = [
                NSDictionary.self,
                NSArray.self,
                NSString.self,
                NSNumber.self,
                NSData.self,
                NSURL.self,
                NSDate.self,
                NSValue.self
            ]
            
            let object = try NSKeyedUnarchiver.unarchivedObject(
                ofClasses: allowedClasses,
                from: data
            )
            
            return describe(object)
            
        } catch {
            return """
            Failed to decode WebArchive:
            
            \(error)
            
            Size: \(data.count) bytes
            """
        }
    }
    
    private func describe(_ object: Any?) -> String {
        guard let object = object else {
            return "Decoded object is nil."
        }
        
        var result = ""
        result += "Root type:\n"
        result += "\(type(of: object))\n\n"
        
        if let dictionary = object as? [String: Any] {
            result += "Dictionary keys:\n"
            
            for key in dictionary.keys.sorted() {
                result += "• \(key)\n"
            }
        }
        
        return result
    }
}
```

### The important change

You had:

```swift id="7ehs6o"
let object = try NSKeyedUnarchiver.unarchiveTopLevelObjectWithData(data)
```

The direct modern API is:

```swift id="rfiz02"
let object = try NSKeyedUnarchiver.unarchivedObject(
    ofClasses: allowedClasses,
    from: data
)
```

I used **`ofClasses:` rather than `ofClass:`** because you're decoding an archive whose contents aren't guaranteed to be a single known class. `ofClass:` is appropriate when you know the root type, for example:

```swift id="ukhj7z"
let image = try NSKeyedUnarchiver.unarchivedObject(
    ofClass: UIImage.self,
    from: data
)
```

One other consideration: if this Playground is intended to **actually extract the image embedded inside the WebArchive**, rather than just inspect its root dictionary, the code needs to recursively inspect the decoded archive and locate its image/data objects. The current `describe()` function only reports the root type and dictionary keys.
:::

Absolutely. The simplest Playground-friendly approach is to make the output `Text` selectable with SwiftUI’s `.textSelection(.enabled)` and add a **Copy Output** button that copies the entire log to the clipboard.

:::writing{variant="document" id="58321" title="Swift Playground Clipboard Inspector with Selectable and Copyable Output"}
```swift id="kbyy1n"
import SwiftUI
import UIKit

struct ContentView: View {
    @State private var output = "Ready."
    @State private var copied = false
    
    var body: some View {
        VStack(spacing: 16) {
            Text("Clipboard Inspector")
                .font(.title)
                .bold()
            
            HStack(spacing: 12) {
                Button("Extract Clipboard Image") {
                    output = extractClipboardImage()
                    copied = false
                }
                .buttonStyle(.borderedProminent)
                
                Button {
                    UIPasteboard.general.string = output
                    copied = true
                } label: {
                    Label(
                        copied ? "Copied!" : "Copy Output",
                        systemImage: copied ? "checkmark" : "doc.on.doc"
                    )
                }
                .buttonStyle(.bordered)
                .disabled(output.isEmpty)
            }
            
            ScrollView {
                Text(output)
                    .font(.system(.body, design: .monospaced))
                    .frame(maxWidth: .infinity, alignment: .leading)
                    .padding()
                    // Allows the user to select and copy individual text.
                    .textSelection(.enabled)
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)
            .background(
                RoundedRectangle(cornerRadius: 12)
                    .fill(Color(.secondarySystemBackground))
            )
        }
        .padding()
    }
    
    private func extractClipboardImage() -> String {
        let pasteboard = UIPasteboard.general
        
        guard let data = pasteboard.items.first?[
            "com.apple.webarchive"
        ] as? Data else {
            return "No com.apple.webarchive data found."
        }
        
        print("WebArchive size: \(data.count) bytes")
        
        do {
            let allowedClasses: [AnyClass] = [
                NSDictionary.self,
                NSArray.self,
                NSString.self,
                NSNumber.self,
                NSData.self,
                NSURL.self,
                NSDate.self,
                NSValue.self
            ]
            
            let object = try NSKeyedUnarchiver.unarchivedObject(
                ofClasses: allowedClasses,
                from: data
            )
            
            return describe(object)
            
        } catch {
            return """
            Failed to decode WebArchive:
            
            \(error)
            
            Size: \(data.count) bytes
            """
        }
    }
    
    private func describe(_ object: Any?) -> String {
        guard let object = object else {
            return "Decoded object is nil."
        }
        
        var result = ""
        result += "Root type:\n"
        result += "\(type(of: object))\n\n"
        
        if let dictionary = object as? [String: Any] {
            result += "Dictionary keys:\n"
            
            for key in dictionary.keys.sorted() {
                result += "• \(key)\n"
            }
        }
        
        return result
    }
}
```

### What changed

- **Select text:** The log area now uses `.textSelection(.enabled)`, so you can long-press/right-click and select portions of the output.
- **Copy everything:** Added a **Copy Output** button that puts the entire `output` string onto `UIPasteboard.general`.
- **Copy confirmation:** The button changes to **“Copied!”** with a checkmark after copying.
- **Scrollable log:** The output remains independently scrollable for large logs.

If you're running this in a **macOS Swift Playground**, I can also adapt it to use the macOS clipboard (`NSPasteboard`) and provide a more native selectable log view.
:::

That error actually tells us something important: **the problem isn't the deprecation replacement**.

`com.apple.webarchive` data is a **WebArchive**, and its serialized representation is a property-list-based archive, not necessarily an `NSKeyedArchiver` archive. Apple describes `WebArchive` as the format used for web content on the pasteboard, and the underlying archive contains things such as `WebMainResource` and `WebSubresources`.

So trying to feed those 28,166 bytes into `NSKeyedUnarchiver` produces:

> `incomprehensible archive version (-1)`

### Better approach

For your Playground, I'd first inspect the data as a **property list**, and then recursively inspect the WebArchive. This also lets us find embedded images.

Here is an updated version that:

- Reads `com.apple.webarchive`
- Detects the archive format
- Parses it with `PropertyListSerialization`
- Shows the WebArchive structure
- Reports embedded resources
- Identifies image resources
- Displays their MIME types, URLs, and sizes
- Keeps the selectable/copyable output interface

:::writing{variant="document" id="58321" title="Swift Playground WebArchive Inspector"}
```swift id="mogd25"
import SwiftUI
import UIKit

struct ContentView: View {
    @State private var output = "Ready."
    @State private var copied = false
    
    var body: some View {
        VStack(spacing: 16) {
            Text("Clipboard Inspector")
                .font(.title)
                .bold()
            
            HStack(spacing: 12) {
                Button("Inspect Clipboard") {
                    output = inspectClipboard()
                    copied = false
                }
                .buttonStyle(.borderedProminent)
                
                Button {
                    UIPasteboard.general.string = output
                    copied = true
                } label: {
                    Label(
                        copied ? "Copied!" : "Copy Output",
                        systemImage: copied ? "checkmark" : "doc.on.doc"
                    )
                }
                .buttonStyle(.bordered)
                .disabled(output.isEmpty)
            }
            
            ScrollView {
                Text(output)
                    .font(.system(.body, design: .monospaced))
                    .frame(maxWidth: .infinity, alignment: .leading)
                    .padding()
                    .textSelection(.enabled)
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)
            .background(
                RoundedRectangle(cornerRadius: 12)
                    .fill(Color(.secondarySystemBackground))
            )
        }
        .padding()
    }
    
    // MARK: - Clipboard
    
    private func inspectClipboard() -> String {
        let pasteboard = UIPasteboard.general
        
        guard let item = pasteboard.items.first else {
            return "Clipboard is empty."
        }
        
        var result = ""
        
        result += "Clipboard types:\n"
        result += "-----------------\n"
        
        for type in item.keys.sorted() {
            result += "• \(type)\n"
        }
        
        result += "\n"
        
        guard let data = item["com.apple.webarchive"] as? Data else {
            result += "No com.apple.webarchive data found."
            return result
        }
        
        result += inspectWebArchive(data)
        
        return result
    }
    
    // MARK: - WebArchive
    
    private func inspectWebArchive(_ data: Data) -> String {
        var result = ""
        
        result += "WebArchive\n"
        result += "==========\n"
        result += "Size: \(data.count) bytes\n\n"
        
        // Show first few bytes so we can identify the format.
        let headerBytes = data.prefix(16)
        
        let hexHeader = headerBytes
            .map { String(format: "%02X", $0) }
            .joined(separator: " ")
        
        let asciiHeader = String(
            bytes: headerBytes,
            encoding: .ascii
        ) ?? ""
        
        result += "Header:\n"
        result += "\(hexHeader)\n"
        result += "ASCII: \(asciiHeader)\n\n"
        
        // WebArchives are property-list based.
        do {
            let object = try PropertyListSerialization.propertyList(
                from: data,
                options: [],
                format: nil
            )
            
            result += "Property list decoding: SUCCESS\n\n"
            
            if let dictionary = object as? [String: Any] {
                result += inspectArchiveDictionary(dictionary)
            } else {
                result += "Root object type:\n"
                result += "\(type(of: object))\n"
            }
            
        } catch {
            result += """
            Property list decoding: FAILED
            
            Error:
            \(error)
            
            This data may not be a standard WebArchive property list.
            """
        }
        
        return result
    }
    
    // MARK: - Archive Dictionary
    
    private func inspectArchiveDictionary(
        _ dictionary: [String: Any]
    ) -> String {
        
        var result = ""
        
        result += "Root type:\n"
        result += "Dictionary\n\n"
        
        result += "Root keys:\n"
        result += "-----------\n"
        
        for key in dictionary.keys.sorted() {
            result += "• \(key)\n"
        }
        
        result += "\n"
        
        // Main resource
        if let mainResource = dictionary["WebMainResource"] as? [String: Any] {
            result += inspectResource(
                mainResource,
                title: "MAIN RESOURCE"
            )
        }
        
        // Subresources
        if let subresources = dictionary["WebSubresources"] as? [[String: Any]] {
            
            result += "SUBRESOURCES\n"
            result += "============\n"
            result += "Count: \(subresources.count)\n\n"
            
            for (index, resource) in subresources.enumerated() {
                result += inspectResource(
                    resource,
                    title: "RESOURCE \(index + 1)"
                )
            }
        }
        
        // Subframe archives
        if let subframes = dictionary["WebSubframeArchives"] as? [[String: Any]] {
            result += "SUBFRAME ARCHIVES\n"
            result += "=================\n"
            result += "Count: \(subframes.count)\n\n"
        }
        
        return result
    }
    
    // MARK: - Resource
    
    private func inspectResource(
        _ resource: [String: Any],
        title: String
    ) -> String {
        
        var result = ""
        
        result += "\(title)\n"
        result += String(repeating: "-", count: title.count)
        result += "\n"
        
        if let url = resource["WebResourceURL"] as? String {
            result += "URL: \(url)\n"
        } else if let url = resource["WebResourceURL"] as? URL {
            result += "URL: \(url.absoluteString)\n"
        }
        
        if let mimeType = resource["WebResourceMIMEType"] as? String {
            result += "MIME type: \(mimeType)\n"
        }
        
        if let encoding = resource["WebResourceTextEncodingName"] as? String {
            result += "Encoding: \(encoding)\n"
        }
        
        if let frameName = resource["WebResourceFrameName"] as? String,
           !frameName.isEmpty {
            result += "Frame: \(frameName)\n"
        }
        
        if let resourceData = resource["WebResourceData"] as? Data {
            
            result += "Data size: \(resourceData.count) bytes\n"
            
            let mimeType = (
                resource["WebResourceMIMEType"] as? String
            )?.lowercased() ?? ""
            
            if mimeType.hasPrefix("image/") {
                result += "IMAGE RESOURCE: YES\n"
                
                if let image = UIImage(data: resourceData) {
                    result += "UIImage decoding: SUCCESS\n"
                    result += "Image size: \(Int(image.size.width)) × \(Int(image.size.height))\n"
                    result += "Scale: \(image.scale)\n"
                } else {
                    result += "UIImage decoding: FAILED\n"
                }
            }
            
            // Show a text preview for textual resources.
            if mimeType.hasPrefix("text/") ||
                mimeType.contains("html") ||
                mimeType.contains("javascript") {
                
                if let text = String(
                    data: resourceData,
                    encoding: .utf8
                ) {
                    result += "\nText preview:\n"
                    result += "-------------\n"
                    
                    let preview = String(
                        text.prefix(1000)
                    )
                    
                    result += preview
                    
                    if text.count > 1000 {
                        result += "\n…"
                    }
                    
                    result += "\n"
                }
            }
        }
        
        result += "\n"
        
        return result
    }
}
```

### Why this should fix your specific error

Your original approach effectively did this:

```swift id="doki98"
NSKeyedUnarchiver.unarchivedObject(...)
```

But the bytes you're receiving don't appear to be an `NSKeyedArchiver` archive. A WebArchive is a separate WebKit format; Apple documents its data representation as the data used to save/restore a `WebArchive` or put it on the pasteboard.

The standard WebArchive structure looks roughly like:

```text id="hbuyg7"
WebArchive
│
├── WebMainResource
│   ├── WebResourceData
│   ├── WebResourceURL
│   ├── WebResourceMIMEType
│   └── ...
│
├── WebSubresources
│   ├── Resource 1
│   ├── Resource 2
│   ├── Resource 3
│   └── ...
│
└── WebSubframeArchives
```

WebKit's own source identifies these same keys, including `WebMainResource`, `WebSubresources`, `WebResourceData`, `WebResourceMIMEType`, and `WebResourceURL`.

### One important next step

Since your original button was called **“Extract Clipboard Image”**, I suspect your actual goal isn't merely to inspect the archive — you want to **extract the image that Safari/iOS put into `com.apple.webarchive`**.

The code above will tell us exactly what's inside. If you run it and paste the resulting log here, particularly the sections showing:

```text id="met3em"
MAIN RESOURCE
RESOURCE 1
RESOURCE 2
...
```

I can then modify it to **automatically extract the embedded image and display it in the Playground**, rather than just reporting its existence.

Excellent — that confirms the extraction path is working correctly. The image is **already embedded in the clipboard WebArchive**, and `UIImage` can decode it successfully.

The next logical step is to make the Playground from an *inspector* into an **image extractor**: find image resources inside `WebSubresources`, decode the first one, display it, and still provide the selectable/copyable log.

:::writing{variant="document" id="58321" title="Swift Playground WebArchive Image Extractor"}
```swift id="9phzq5"
import SwiftUI
import UIKit

struct ContentView: View {
    @State private var output = "Ready."
    @State private var extractedImage: UIImage?
    @State private var copied = false
    
    var body: some View {
        VStack(spacing: 16) {
            
            Text("Clipboard Image Extractor")
                .font(.title)
                .bold()
            
            HStack(spacing: 12) {
                
                Button("Extract Image") {
                    let result = extractClipboardImage()
                    output = result.log
                    extractedImage = result.image
                    copied = false
                }
                .buttonStyle(.borderedProminent)
                
                Button {
                    UIPasteboard.general.string = output
                    copied = true
                } label: {
                    Label(
                        copied ? "Copied!" : "Copy Log",
                        systemImage: copied ? "checkmark" : "doc.on.doc"
                    )
                }
                .buttonStyle(.bordered)
                .disabled(output.isEmpty)
            }
            
            // MARK: - Extracted Image
            
            if let image = extractedImage {
                VStack(spacing: 8) {
                    Text("Extracted Image")
                        .font(.headline)
                    
                    Image(uiImage: image)
                        .resizable()
                        .scaledToFit()
                        .frame(maxHeight: 300)
                        .clipShape(
                            RoundedRectangle(cornerRadius: 12)
                        )
                }
                .padding(.horizontal)
            }
            
            // MARK: - Log
            
            ScrollView {
                Text(output)
                    .font(.system(.body, design: .monospaced))
                    .frame(
                        maxWidth: .infinity,
                        alignment: .leading
                    )
                    .padding()
                    .textSelection(.enabled)
            }
            .frame(
                maxWidth: .infinity,
                maxHeight: .infinity
            )
            .background(
                RoundedRectangle(cornerRadius: 12)
                    .fill(
                        Color(.secondarySystemBackground)
                    )
            )
        }
        .padding()
    }
    
    // MARK: - Clipboard
    
    private func extractClipboardImage()
        -> (image: UIImage?, log: String)
    {
        let pasteboard = UIPasteboard.general
        
        guard let item = pasteboard.items.first else {
            return (
                nil,
                "Clipboard is empty."
            )
        }
        
        var log = ""
        
        log += "Clipboard types:\n"
        log += "-----------------\n"
        
        for type in item.keys.sorted() {
            log += "• \(type)\n"
        }
        
        log += "\n"
        
        guard let data = item[
            "com.apple.webarchive"
        ] as? Data else {
            log += "No com.apple.webarchive data found."
            
            return (
                nil,
                log
            )
        }
        
        log += "WebArchive\n"
        log += "==========\n"
        log += "Size: \(data.count) bytes\n\n"
        
        // MARK: Property List
        
        let object: Any
        
        do {
            object = try PropertyListSerialization.propertyList(
                from: data,
                options: [],
                format: nil
            )
        } catch {
            log += """
            Property list decoding FAILED
            
            \(error)
            """
            
            return (
                nil,
                log
            )
        }
        
        guard let archive = object as? [String: Any] else {
            log += "Root object is not a dictionary."
            return (
                nil,
                log
            )
        }
        
        log += "Property list decoding: SUCCESS\n\n"
        
        log += "Root keys:\n"
        log += "-----------\n"
        
        for key in archive.keys.sorted() {
            log += "• \(key)\n"
        }
        
        log += "\n"
        
        // MARK: Main Resource
        
        if let mainResource = archive[
            "WebMainResource"
        ] as? [String: Any] {
            
            log += describeResource(
                mainResource,
                title: "MAIN RESOURCE"
            )
        }
        
        // MARK: Subresources
        
        guard let subresources = archive[
            "WebSubresources"
        ] as? [[String: Any]]
        else {
            log += "No WebSubresources found.\n"
            return (
                nil,
                log
            )
        }
        
        log += "SUBRESOURCES\n"
        log += "============\n"
        log += "Count: \(subresources.count)\n\n"
        
        var firstImage: UIImage?
        var imageCount = 0
        
        for (index, resource) in subresources.enumerated() {
            
            log += describeResource(
                resource,
                title: "RESOURCE \(index + 1)"
            )
            
            guard let mimeType = resource[
                "WebResourceMIMEType"
            ] as? String else {
                continue
            }
            
            let lowerMIMEType = mimeType.lowercased()
            
            guard lowerMIMEType.hasPrefix("image/")
            else {
                continue
            }
            
            imageCount += 1
            
            guard let imageData = resource[
                "WebResourceData"
            ] as? Data else {
                log += "Image has no WebResourceData.\n\n"
                continue
            }
            
            if let image = UIImage(data: imageData) {
                
                if firstImage == nil {
                    firstImage = image
                }
                
                log += "Image extraction: SUCCESS\n"
                
                log += "Format: \(mimeType)\n"
                log += "Bytes: \(imageData.count)\n"
                log += "Dimensions: "
                log += "\(Int(image.size.width)) × "
                log += "\(Int(image.size.height))\n"
                log += "Scale: \(image.scale)\n\n"
                
            } else {
                
                log += "Image extraction: FAILED\n\n"
            }
        }
        
        log += "IMAGE SUMMARY\n"
        log += "=============\n"
        log += "Images found: \(imageCount)\n"
        
        if firstImage != nil {
            log += "First image: extracted successfully.\n"
        } else {
            log += "No decodable image found.\n"
        }
        
        return (
            firstImage,
            log
        )
    }
    
    // MARK: - Resource Description
    
    private func describeResource(
        _ resource: [String: Any],
        title: String
    ) -> String {
        
        var result = ""
        
        result += "\(title)\n"
        result += String(
            repeating: "-",
            count: title.count
        )
        result += "\n"
        
        if let url = resource[
            "WebResourceURL"
        ] as? String {
            result += "URL: \(url)\n"
        } else if let url = resource[
            "WebResourceURL"
        ] as? URL {
            result += "URL: \(url.absoluteString)\n"
        }
        
        if let mimeType = resource[
            "WebResourceMIMEType"
        ] as? String {
            result += "MIME type: \(mimeType)\n"
        }
        
        if let encoding = resource[
            "WebResourceTextEncodingName"
        ] as? String {
            result += "Encoding: \(encoding)\n"
        }
        
        if let resourceData = resource[
            "WebResourceData"
        ] as? Data {
            
            result += "Data size: "
            result += "\(resourceData.count) bytes\n"
        }
        
        result += "\n"
        
        return result
    }
}
```

### What this version does

When you tap **Extract Image**, it now:

1. Reads `com.apple.webarchive`.
2. Decodes it with `PropertyListSerialization`.
3. Looks through `WebSubresources`.
4. Checks each resource's `WebResourceMIMEType`.
5. Finds resources whose MIME type starts with `image/`.
6. Gets their `WebResourceData`.
7. Converts that data to `UIImage`.
8. Displays the first successfully decoded image directly in the Playground.
9. Reports all of this in the selectable log.

For your particular clipboard data, it should find:

```text id="yb0ujs"
MIME type: image/webp
Data size: 20478 bytes
Image extraction: SUCCESS
Format: image/webp
Dimensions: 620 × 349
```

### One potentially useful improvement

Your WebArchive contains the **actual image data**, so you don't need to download the Medium URL at all. That's preferable: the extractor can work entirely from the clipboard contents.

If the ultimate goal is to make this a **one-tap “copy image from Safari/Medium/etc.” utility**, the next improvement I'd make is to add a **“Copy Extracted Image”** button alongside the preview, so the extracted WebP gets placed back onto the iOS pasteboard as a normal image.

Excellent — that confirms the extraction path is working correctly. The image is **already embedded in the clipboard WebArchive**, and `UIImage` can decode it successfully.

The next logical step is to make the Playground actually **copy the extracted image back to the clipboard**, rather than merely display it.

For iOS, I recommend putting the decoded `UIImage` onto the pasteboard as PNG data. That makes it broadly compatible with apps that accept ordinary image paste operations.

Replace your current code with this version:

:::writing{variant="document" id="58321" title="Swift Playground WebArchive Image Extractor with Copy Image"}
```swift id="hlq0hd"
import SwiftUI
import UIKit

struct ContentView: View {
    @State private var output = "Ready."
    @State private var extractedImage: UIImage?
    @State private var copiedImage = false
    @State private var copiedLog = false
    
    var body: some View {
        VStack(spacing: 16) {
            
            Text("Clipboard Image Extractor")
                .font(.title)
                .bold()
            
            HStack(spacing: 12) {
                
                Button("Extract Image") {
                    let result = extractClipboardImage()
                    
                    output = result.log
                    extractedImage = result.image
                    
                    copiedImage = false
                    copiedLog = false
                }
                .buttonStyle(.borderedProminent)
                
                Button {
                    UIPasteboard.general.string = output
                    copiedLog = true
                    
                    DispatchQueue.main.asyncAfter(
                        deadline: .now() + 1.5
                    ) {
                        copiedLog = false
                    }
                } label: {
                    Label(
                        copiedLog ? "Copied!" : "Copy Log",
                        systemImage: copiedLog
                            ? "checkmark"
                            : "doc.on.doc"
                    )
                }
                .buttonStyle(.bordered)
                .disabled(output.isEmpty)
            }
            
            // MARK: - Extracted Image
            
            if let image = extractedImage {
                
                VStack(spacing: 10) {
                    
                    Text("Extracted Image")
                        .font(.headline)
                    
                    Image(uiImage: image)
                        .resizable()
                        .scaledToFit()
                        .frame(maxHeight: 300)
                        .clipShape(
                            RoundedRectangle(cornerRadius: 12)
                        )
                    
                    Button {
                        copyImageToPasteboard(image)
                    } label: {
                        Label(
                            copiedImage
                                ? "Image Copied!"
                                : "Copy Image",
                            systemImage: copiedImage
                                ? "checkmark"
                                : "photo.on.rectangle"
                        )
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding(.horizontal)
            }
            
            // MARK: - Log
            
            ScrollView {
                Text(output)
                    .font(.system(.body, design: .monospaced))
                    .frame(
                        maxWidth: .infinity,
                        alignment: .leading
                    )
                    .padding()
                    .textSelection(.enabled)
            }
            .frame(
                maxWidth: .infinity,
                maxHeight: .infinity
            )
            .background(
                RoundedRectangle(cornerRadius: 12)
                    .fill(
                        Color(.secondarySystemBackground)
                    )
            )
        }
        .padding()
    }
    
    // MARK: - Clipboard
    
    private func extractClipboardImage()
        -> (image: UIImage?, log: String)
    {
        let pasteboard = UIPasteboard.general
        
        guard let item = pasteboard.items.first else {
            return (
                nil,
                "Clipboard is empty."
            )
        }
        
        var log = ""
        
        log += "Clipboard types:\n"
        log += "-----------------\n"
        
        for type in item.keys.sorted() {
            log += "• \(type)\n"
        }
        
        log += "\n"
        
        guard let data = item[
            "com.apple.webarchive"
        ] as? Data else {
            
            log += "No com.apple.webarchive data found."
            
            return (
                nil,
                log
            )
        }
        
        log += "WebArchive\n"
        log += "==========\n"
        log += "Size: \(data.count) bytes\n\n"
        
        // MARK: Decode Property List
        
        let object: Any
        
        do {
            object = try PropertyListSerialization.propertyList(
                from: data,
                options: [],
                format: nil
            )
        } catch {
            
            log += """
            Property list decoding FAILED
            
            \(error)
            """
            
            return (
                nil,
                log
            )
        }
        
        guard let archive = object as? [String: Any] else {
            
            log += "Root object is not a dictionary."
            
            return (
                nil,
                log
            )
        }
        
        log += "Property list decoding: SUCCESS\n\n"
        
        log += "Root keys:\n"
        log += "-----------\n"
        
        for key in archive.keys.sorted() {
            log += "• \(key)\n"
        }
        
        log += "\n"
        
        // MARK: Main Resource
        
        if let mainResource = archive[
            "WebMainResource"
        ] as? [String: Any] {
            
            log += describeResource(
                mainResource,
                title: "MAIN RESOURCE"
            )
        }
        
        // MARK: Subresources
        
        guard let subresources = archive[
            "WebSubresources"
        ] as? [[String: Any]]
        else {
            
            log += "No WebSubresources found.\n"
            
            return (
                nil,
                log
            )
        }
        
        log += "SUBRESOURCES\n"
        log += "============\n"
        log += "Count: \(subresources.count)\n\n"
        
        var firstImage: UIImage?
        var imageCount = 0
        
        for (index, resource) in subresources.enumerated() {
            
            log += describeResource(
                resource,
                title: "RESOURCE \(index + 1)"
            )
            
            guard let mimeType = resource[
                "WebResourceMIMEType"
            ] as? String else {
                continue
            }
            
            let lowerMIMEType = mimeType.lowercased()
            
            guard lowerMIMEType.hasPrefix("image/")
            else {
                continue
            }
            
            imageCount += 1
            
            guard let imageData = resource[
                "WebResourceData"
            ] as? Data else {
                
                log += "Image has no WebResourceData.\n\n"
                
                continue
            }
            
            guard let image = UIImage(
                data: imageData
            ) else {
                
                log += "Image extraction: FAILED\n\n"
                
                continue
            }
            
            if firstImage == nil {
                firstImage = image
            }
            
            log += "Image extraction: SUCCESS\n"
            log += "Format: \(mimeType)\n"
            log += "Bytes: \(imageData.count)\n"
            log += "Dimensions: "
            log += "\(Int(image.size.width)) × "
            log += "\(Int(image.size.height))\n"
            log += "Scale: \(image.scale)\n\n"
        }
        
        // MARK: Summary
        
        log += "IMAGE SUMMARY\n"
        log += "=============\n"
        log += "Images found: \(imageCount)\n"
        
        if firstImage != nil {
            log += "First image: extracted successfully.\n"
        } else {
            log += "No decodable image found.\n"
        }
        
        return (
            firstImage,
            log
        )
    }
    
    // MARK: - Copy Image
    
    private func copyImageToPasteboard(
        _ image: UIImage
    ) {
        guard let pngData = image.pngData() else {
            return
        }
        
        UIPasteboard.general.setData(
            pngData,
            forPasteboardType: "public.png"
        )
        
        copiedImage = true
        
        DispatchQueue.main.asyncAfter(
            deadline: .now() + 1.5
        ) {
            copiedImage = false
        }
    }
    
    // MARK: - Resource Description
    
    private func describeResource(
        _ resource: [String: Any],
        title: String
    ) -> String {
        
        var result = ""
        
        result += "\(title)\n"
        result += String(
            repeating: "-",
            count: title.count
        )
        result += "\n"
        
        if let url = resource[
            "WebResourceURL"
        ] as? String {
            
            result += "URL: \(url)\n"
            
        } else if let url = resource[
            "WebResourceURL"
        ] as? URL {
            
            result += "URL: \(url.absoluteString)\n"
        }
        
        if let mimeType = resource[
            "WebResourceMIMEType"
        ] as? String {
            
            result += "MIME type: \(mimeType)\n"
        }
        
        if let encoding = resource[
            "WebResourceTextEncodingName"
        ] as? String {
            
            result += "Encoding: \(encoding)\n"
        }
        
        if let resourceData = resource[
            "WebResourceData"
        ] as? Data {
            
            result += "Data size: "
            result += "\(resourceData.count) bytes\n"
        }
        
        result += "\n"
        
        return result
    }
}
```

### Your current result proves the important part

You have:

```text id="5iaqwm"
MIME type: image/webp
Data size: 20478 bytes
Image extraction: SUCCESS
Dimensions: 620 × 349
```

So **don't use the URL to download the image**. The original WebP is already sitting inside the pasteboard's WebArchive.

The flow is now:

```text id="a3zt5y"
Safari / Medium
      │
      ▼
iOS Pasteboard
      │
      └── com.apple.webarchive
              │
              ▼
       PropertyListSerialization
              │
              ▼
        WebSubresources
              │
              ▼
          image/webp
              │
              ▼
        UIImage(data:)
              │
              ▼
       ┌──────────────┐
       │ Image Preview │
       └──────────────┘
              │
              ▼
          Copy Image
              │
              ▼
       public.png
```

### One potentially useful improvement

Your WebArchive contains the **actual image data**, so you don't need to download the Medium URL at all. That's preferable: the extractor can work entirely from the clipboard contents.

If your goal is to preserve the **original WebP bytes** rather than convert through `UIImage` to PNG, we can instead copy the extracted `WebResourceData` directly and preserve the original image without the PNG conversion.