# OneApp patch to mobile_scanner 7.5.0

Upstream: https://github.com/juliansteenbakker/mobile_scanner (pub.dev:
https://pub.dev/packages/mobile_scanner).

This branch is upstream tag `v7.5.0` plus the OneApp patch commits. OneApp pulls it as a git
dependency (`dependency_overrides` in the app's `pubspec.yaml`, pinned to the branch tip). The
only code changes on top of the tag are in
`darwin/mobile_scanner/Sources/mobile_scanner/MobileScannerPlugin.swift`, in `releaseCamera()`
and `start()` / `configureSession()`, marked `OneApp patch`.

## Why

On iOS 27, OneApp froze on the scan screens (Scan SKU, Stocktake scan-location,
Articlepedia, the ordering cart) until iOS killed it: Datadog RUM `AppHang` issues whose
main-thread stack ends in `mobile_scanner` → `AVFCapture` (`releaseCamera()` /
`captureSession` setter), followed by `WatchdogTermination`. Over 30 days, roughly 1.0–1.6%
of scan sessions on iOS 27.0 / 27.2 were hit, against about 0.1–0.3% on iOS 26.x, on every
app version from 1.7.0 to 1.11.0 (mobile_scanner 7.2.0 up to 7.4.2; `releaseCamera()` is the
same in all of them).

`releaseCamera()` stops the capture session and drops the last reference to it on the main
thread. Both wait synchronously on the system camera service, which on iOS 27 can take
seconds. Upstream 7.5.0 still does this, so upgrading alone does not fix it.

## What changed

`releaseCamera()` clears `captureSession` on the main thread right away (so the plugin
reads as stopped and can be started again) and runs `stopRunning()`, the input/output
removal and the final release of the session on the plugin's serial `sessionQueue`.

`start()` builds the session configuration (`AVCaptureDeviceInput`, `addInput`, the video
output, `commitConfiguration()`) in `configureSession()` on that same `sessionQueue`, followed
by `startRunning()`. Creating the input and committing the configuration also wait on the
camera service, most of all right after another session was torn down: the Stocktake
Scan Location → Scan SKU hand-off and the camera coming back after the quantity sheet. Because
the queue is serial, a start's setup runs after any pending teardown, and the teardown of a
stop() that arrives meanwhile runs after it. The UIKit reads (interface orientation, the
orientation observer) and the device observers stay on the main thread. A setup failure hops
back to the main thread and releases the camera through `failStart`, unless a stop() already
released that session.

What the earlier 7.2.0 patch also carried is now upstream in 7.5.0 and was dropped:

- `startRunning()` on a serial, user-initiated `sessionQueue` (upstream #1828). Because the
  queue is serial, a restart queues behind the teardown above instead of racing it.
- `start()` releasing the camera when it fails (`failStart`), so a failed start no longer
  leaves every later start refused as "already started".
- Stale frames after a stop: frames now hop to the main queue and are dropped once the
  texture is released (`notifyFrameAvailable`), so the old `captureSession == nil` check in
  `captureOutput` is no longer needed (it ran on the main queue in 7.2.0; in 7.5.0
  `captureOutput` runs on `sampleBufferQueue`).

## Updating it

Make a new branch `oneapp/<version>-ios-stop-off-main` from the new upstream tag, cherry-pick
the OneApp patch commit onto it, check that `sessionQueue` is still the queue `start()` uses,
push it, and move the app's `ref` to the new commit.

## Removing it

Delete the `dependency_overrides` entry in the app once an upstream release moves the
session teardown off the main thread, then re-check the AppHang issues in Datadog on iOS 27.
