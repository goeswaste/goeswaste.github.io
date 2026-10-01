---
title: iPod transfer
---

Develop a Cocoa application for macOS 1.0.6 to transfer files from a iPod (MB531ZP) running iOS 4.2.1

---

Build it as a **32-bit Objective-C Cocoa application targeting Mac OS X 10.6.6 Snow Leopard**, not “macOS 1.0.6.” The iPod model identifier **MB531ZP/A** is an iPod touch model from the iOS 4 era, and an iOS 4.2.1 device should be accessed through Apple’s USB device services rather than as a normal USB disk.

The practical architecture is:

1. Detect the iPod over USB.
2. Establish or reuse a trusted pairing record.
3. Start the AFC service.
4. Browse and transfer files in the app-specific AFC container.
5. Optionally read the media database for music, videos, ratings, playlists, and artwork.
6. Never modify the iPod unless the user explicitly requests it.

Apple’s normal file-sharing workflow is app-based rather than a general filesystem export. Apple documents file transfers through iTunes for older versions of OS X, while later systems expose the same concept through Finder. <citation src="1"></citation>

## Important limitation

Without a jailbroken iPod, an application cannot normally browse the complete iOS filesystem. AFC generally exposes:

- Files belonging to applications that support file sharing
- Certain media and diagnostic services, depending on the device and pairing state
- Possibly the media library through a separate database/service

It does not provide unrestricted access to `/System`, other applications’ private data, or the entire user filesystem.

## Recommended implementation

Use:

- **Objective-C**
- **Cocoa/AppKit**
- **Xcode 3.2.x**
- **Mac OS X 10.6 SDK**
- **32-bit Intel target**
- **libimobiledevice**, built for Snow Leopard
- **libplist**
- **libusbmuxd**
- **libiphone** or the corresponding historical libimobiledevice device API

A modern libimobiledevice build may not support an old 10.6 deployment target, so the dependencies should be compiled from historically compatible revisions or patched locally. The iPod touch 4.x firmware family includes iOS 4 releases, including the generation relevant to iOS 4.2.1. <citation src="4"></citation>

## Application layout

Use a three-pane interface:

```text
+---------------------------------------------------------+
| iPod Transfer                         [Connect]          |
+------------------+--------------------------------------+
| Device           | Files                                |
|                  |                                      |
| iPod touch       | Name        Size       Modified       |
| iOS 4.2.1        | Documents   --         --             |
| Serial: ...      | Photos      --         --             |
|                  |                                      |
+------------------+--------------------------------------+
| Status: Ready                         [Download] [Quit] |
+---------------------------------------------------------+
```

Suggested classes:

```text
IPTDeviceController
IPTAFCBrowser
IPTTransferOperation
IPTDeviceInfo
IPTFileItem
IPTMainWindowController
```

## Core device controller

The following is an illustrative Objective-C interface. Function names may vary slightly between libimobiledevice revisions.

```objc
// IPTDeviceController.h

#import <Foundation/Foundation.h>

@class AMDevice;

@interface IPTDeviceController : NSObject {
@private
    AMDevice *_device;
    BOOL _connected;
}

@property(nonatomic, readonly) BOOL connected;
@property(nonatomic, retain, readonly) NSString *deviceName;
@property(nonatomic, retain, readonly) NSString *productVersion;
@property(nonatomic, retain, readonly) NSString *serialNumber;

- (BOOL)connect:(NSError **)error;
- (void)disconnect;

@end
```

Implementation outline:

```objc
// IPTDeviceController.m

#import "IPTDeviceController.h"
#import <libimobiledevice/libimobiledevice.h>
#import <libimobiledevice/lockdown.h>

@implementation IPTDeviceController

@synthesize connected = _connected;

- (BOOL)connect:(NSError **)error
{
    idevice_t device = NULL;
    idevice_error_t result;

    result = idevice_new(&device, NULL);

    if (result != IDEVICE_E_SUCCESS || device == NULL) {
        if (error != NULL) {
            *error = [NSError errorWithDomain:@"IPTDeviceError"
                                          code:100
                                      userInfo:
                      [NSDictionary dictionaryWithObject:
                       @"No compatible iPod was detected."
                       forKey:NSLocalizedDescriptionKey]];
        }
        return NO;
    }

    lockdownd_client_t lockdown = NULL;

    result = lockdownd_client_new_with_handshake(
        device,
        &lockdown,
        "IPodTransfer"
    );

    if (result != IDEVICE_E_SUCCESS || lockdown == NULL) {
        idevice_free(device);

        if (error != NULL) {
            *error = [NSError errorWithDomain:@"IPTDeviceError"
                                          code:101
                                      userInfo:
                      [NSDictionary dictionaryWithObject:
                       @"The iPod is not paired with this Mac."
                       forKey:NSLocalizedDescriptionKey]];
        }
        return NO;
    }

    char *value = NULL;

    lockdownd_get_value(lockdown, NULL, "ProductVersion", &value);
    if (value != NULL) {
        _productVersion = [[NSString alloc] initWithUTF8String:value];
        free(value);
    }

    value = NULL;
    lockdownd_get_value(lockdown, NULL, "SerialNumber", &value);
    if (value != NULL) {
        _serialNumber = [[NSString alloc] initWithUTF8String:value];
        free(value);
    }

    lockdownd_client_free(lockdown);
    idevice_free(device);

    _connected = YES;
    return YES;
}

- (void)disconnect
{
    _connected = NO;
    [_device release];
    _device = nil;
}

- (void)dealloc
{
    [self disconnect];
    [_deviceName release];
    [_productVersion release];
    [_serialNumber release];
    [super dealloc];
}

@end
```

On the first connection, iOS may display a trust prompt. The user must accept it. A pairing record is normally created by iTunes or the device-service library.

## AFC file browser

AFC uses a service socket obtained through lockdown. A basic browser can expose directory contents using `afc_read_directory`.

```objc
// IPTAFCBrowser.h

#import <Foundation/Foundation.h>

@interface IPTAFCBrowser : NSObject

- (BOOL)openForDevice:(id)device error:(NSError **)error;
- (NSArray *)contentsOfDirectory:(NSString *)path
                            error:(NSError **)error;
- (BOOL)downloadFile:(NSString *)remotePath
          toLocalURL:(NSURL *)localURL
              error:(NSError **)error;

@end
```

Representative directory enumeration:

```objc
- (NSArray *)contentsOfDirectory:(NSString *)path
                           error:(NSError **)error
{
    afc_client_t afc = NULL;
    char **entries = NULL;
    afc_error_t result;

    result = afc_read_directory(afc, [path UTF8String], &entries);

    if (result != AFC_E_SUCCESS) {
        if (error != NULL) {
            *error = [NSError errorWithDomain:@"IPTAFCError"
                                          code:200
                                      userInfo:
                      [NSDictionary dictionaryWithObject:
                       @"Unable to read the iPod directory."
                       forKey:NSLocalizedDescriptionKey]];
        }
        return nil;
    }

    NSMutableArray *items = [NSMutableArray array];

    for (char **entry = entries; *entry != NULL; entry++) {
        NSString *name =
            [NSString stringWithUTF8String:*entry];

        if (![name isEqualToString:@"."] &&
            ![name isEqualToString:@".."]) {
            [items addObject:name];
        }
    }

    afc_dictionary_free(entries);
    return items;
}
```

For reliable transfers, do not load an entire file into memory. Stream it in chunks:

```objc
#define IPTTransferBufferSize (64 * 1024)

- (BOOL)downloadFile:(NSString *)remotePath
          toLocalURL:(NSURL *)localURL
              error:(NSError **)error
{
    afc_file_ref_t remoteFile = NULL;
    afc_error_t result;

    result = afc_file_open(
        _afcClient,
        [remotePath UTF8String],
        AFC_FOPEN_RDONLY,
        &remoteFile
    );

    if (result != AFC_E_SUCCESS || remoteFile == NULL) {
        return NO;
    }

    NSFileHandle *output =
        [NSFileHandle fileHandleForWritingToURL:localURL
                                         error:error];

    if (output == nil) {
        afc_file_close(remoteFile);
        return NO;
    }

    uint8_t buffer[IPTTransferBufferSize];
    uint32_t bytesRead = 0;

    while (YES) {
        result = afc_file_read(
            remoteFile,
            buffer,
            IPTTransferBufferSize,
            &bytesRead
        );

        if (result != AFC_E_SUCCESS) {
            [output closeFile];
            afc_file_close(remoteFile);
            return NO;
        }

        if (bytesRead == 0) {
            break;
        }

        [output writeData:
            [NSData dataWithBytes:buffer length:bytesRead]];
    }

    [output closeFile];
    afc_file_close(remoteFile);
    return YES;
}
```

The actual AFC client must be initialized after starting the AFC service through lockdown. Keep that code in `IPTAFCBrowser` so the user-interface controller does not depend directly on USB protocol details.

## Cocoa transfer operation

Transfers should run outside the main thread. For Snow Leopard, `NSOperation` is appropriate:

```objc
@interface IPTDownloadOperation : NSOperation {
    NSString *_remotePath;
    NSURL *_localURL;
    id _browser;
}

- (id)initWithRemotePath:(NSString *)remotePath
                localURL:(NSURL *)localURL
                 browser:(id)browser;

@end
```

Use an operation queue:

```objc
_downloadQueue =
    [[NSOperationQueue alloc] init];

[_downloadQueue setMaxConcurrentOperationCount:1];
```

A single transfer at a time is safer for old iOS USB services and reduces the chance of AFC socket failures.

## Media transfer

If the goal is to copy music, videos, photos, or playlists rather than application documents, implement a separate media layer:

```text
IPTMediaDatabaseReader
IPTArtworkReader
IPTPlaylistReader
IPTMediaExporter
```

The media database is not equivalent to ordinary AFC browsing. A media exporter should:

1. Read the device database.
2. Extract track metadata.
3. Locate the corresponding media file.
4. Copy the file to a user-selected folder.
5. Reconstruct a folder structure such as:

```text
Artist/
    Album/
        01 - Title.m4a
```

For each exported item, preserve:

- Title
- Artist
- Album
- Genre
- Track number
- Disc number
- Rating
- Play count
- Artwork, if available
- Playlist membership

Do not overwrite existing files by default. Use a collision policy:

```text
Skip
Replace
Keep both
```

## Build settings

Recommended legacy settings:

```text
Base SDK:          Mac OS X 10.6
Deployment Target: Mac OS X 10.6
Architectures:     i386
Objective-C GC:    Supported or Required
C++ Standard:      gnu++98
Compiler:          GCC 4.2
Link Frameworks:   Cocoa, Foundation, Security
```

The third-party libraries should be built as static libraries or bundled private frameworks:

```text
libimobiledevice.a
libplist.a
libusbmuxd.a
libcrypto.a
libssl.a
```

Avoid relying on APIs introduced after Snow Leopard, including modern sandboxing, ARC, Grand Central Dispatch-heavy designs, Finder Sync, or Swift.

## Failure handling

Display specific messages for these cases:

```text
No device detected
Device is locked
Trust relationship was not established
iTunes is currently using the device
AFC service unavailable
File is protected or inaccessible
USB connection interrupted
Destination file already exists
Insufficient disk space
```

When the cable disconnects:

1. Stop the active transfer.
2. Close the AFC and lockdown clients.
3. Mark the operation as interrupted.
4. Offer “Retry.”
5. Never resume by blindly appending; verify the partial file size first.

## Recommended MVP scope

For a first working version, implement only:

- Device detection
- Pairing/trust handling
- Device information display
- AFC directory listing
- Single-file download
- Multiple-file download
- Progress reporting
- Cancel operation
- Destination-folder selection
- Safe disconnect and retry

Then add media database export as a separate feature. This avoids confusing app file sharing with unrestricted extraction of the iPod’s entire contents.
