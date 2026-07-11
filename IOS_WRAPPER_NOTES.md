# iOS Wrapper (PWABuilder / TestFlight) Notes

Fixes and knowledge for the PWABuilder-generated iOS package (`Catholics.app.zip` → `src/`).

## 1. No sound on device (even after the web-side fix)

Two independent causes — check both:

**a) The ringer/silent switch mutes WebAudio.** By default the WKWebView wrapper plays web
audio in the *ambient* session category, which the hardware mute switch silences. Most testers
have the switch on silent → "the app has no sound." Fix in the PWABuilder Xcode project —
`src/Catholics.app/AppDelegate.swift`, inside
`application(_:didFinishLaunchingWithOptions:)` (before `return true`):

```swift
import AVFoundation   // top of file

// Play game audio even when the ringer switch is on silent
try? AVAudioSession.sharedInstance().setCategory(.playback, options: [.mixWithOthers])
try? AVAudioSession.sharedInstance().setActive(true)
```

**b) Web-side unlock (already fixed in `index.html`):** iOS creates the AudioContext in a
`suspended` state; the app now calls `audioCtx.resume()` on every tap, so any interaction
unlocks sound. If you regenerate the PWABuilder package, no wrapper change is needed for
this part — it ships with the site.

Quick test: flip the ringer switch ON (not silent) and tap once anywhere — if sound works
then, apply fix (a) so it also works on silent.

## 2. White flash on scroll bounce (fixed web-side)

The iOS rubber-band overscroll shows the WKWebView's white underlay. `index.html` now pins a
solid dark background on `<html>`, disables overscroll chaining, and locks page-level
scrolling (all scrolling happens inside the app's content area). If any white still peeks
through in the wrapper, add to `src/Catholics.app/ViewController.swift` after the web view
is created:

```swift
webView.isOpaque = false
webView.backgroundColor = .black
webView.scrollView.backgroundColor = .black
webView.scrollView.bounces = false
```

## 3. Regenerating the package

When you re-run PWABuilder against the deployed site, it produces a fresh `src/` — re-apply
fix 1a (and 2 if used). These are the only manual wrapper edits.
