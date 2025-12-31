# GlowingCursor

GlowingCursor is a macOS SwiftUI experiment that replicates the glowing text insertion cursor shown when Dictation is active. The project is primarily an exercise in reverse engineering private AppKit APIs to understand how this effect is implemented internally.

The repository includes a minimal SwiftUI setup that demonstrates how to trigger the same glow animation programmatically.

![GIF of the Macintosh Dictation Glow Effect in-action.](https://i.imgur.com/Q9buyDy.gif)

## Motivation

While learning to reverse engineer private macOS APIs, I wanted to reproduce the cursor glow effect that appears when pressing the Dictation key. This effect is not exposed through public AppKit APIs, so achieving it requires runtime inspection and interaction with private selectors.

This repo contains:
- A SwiftUI wrapper around `NSTextView`
- Experiments with private AppKit methods
- A working approach to enable and disable the glowing insertion cursor

## Project Structure

- `BetterTextField.swift`  
  A SwiftUI component that wraps an `NSScrollView` and `NSTextView`. It exposes the underlying `NSTextView` so it can be configured or inspected.

- `ContentView.swift`  
  Hosts the text view and provides controls to enable or disable the glow effect by interacting with private APIs.

## Setup

Create a SwiftUI macOS app and embed an `NSTextView`. While `TextField` or `TextEditor` can be used, accessing their backing `NSTextView` requires additional SwiftUI work. This project avoids that by wrapping AppKit directly.

The `BetterTextView` passes the `NSTextView` back to SwiftUI via a callback so it can be stored and manipulated.

## Reverse Engineering Notes

### Inspecting AppKit

Using Hopper:
1. Open Hopper
2. File → Read from DYLD Cache
3. Open `AppKit.framework`
4. Search for symbols containing `glow`

This quickly reveals private methods such as:

- `-[NSTextInsertionIndicator setShowsGlow:]`

Apple documentation confirms that `NSTextInsertionIndicator` is responsible for drawing the insertion cursor used by `NSTextView` and `NSTextField`.

Further inspection shows that the insertion indicator can be accessed via the private selector:

- `-[NSTextView _insertionIndicator]`

### Why `setShowsGlow:` Alone Does Not Work

Calling `setShowsGlow:` directly does not activate the glow animation. LLDB inspection during Dictation shows that:

- `setShowsGlow:` is called internally
- Additional internal methods are invoked afterward
- The real behavior is driven by `-[NSTextInsertionIndicator dictationStateDidChange:]`

This method reacts to private notifications posted when Dictation starts and ends.

### Observed Notifications

When Dictation is toggled, the following notifications are passed:

- `_NSTextInputContextDictationDidStartNotification`
- `_NSTextInputContextDictationDidEndNotification`

Although these notifications are private, they can be recreated using `NSNotification` and passed to `dictationStateDidChange:`.

## Implementation

Below is the core logic used to enable and disable the glowing cursor:

```swift
struct ContentView: View {
    @State var text = ""
    @State var isGlowEnabled = false
    @State var _textView: NSTextView?
    
    var body: some View {
        ZStack(alignment: .bottomTrailing) {
            BetterTextView(text: $text) { nsTextView in
                DispatchQueue.main.async {
                    self._textView = nsTextView
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

            Button(isGlowEnabled ? "Disable Glow" : "Enable Glow") {
                if isGlowEnabled {
                    disableGlow()
                } else {
                    enableGlow()
                }
            }
            .padding()
        }
    }
    
    func enableGlow() {
        guard let textView = _textView else { return }
        isGlowEnabled = true

        let indicator = textView
            .perform(Selector(("_insertionIndicator")))
            .takeUnretainedValue()

        let notification = NSNotification(
            name: .init("_NSTextInputContextDictationDidStartNotification"),
            object: nil
        )

        _ = indicator.perform(
            Selector(("dictationStateDidChange:")),
            with: notification
        )
    }
    
    func disableGlow() {
        guard let textView = _textView else { return }
        isGlowEnabled = false

        let indicator = textView
            .perform(Selector(("_insertionIndicator")))
            .takeUnretainedValue()

        let notification = NSNotification(
            name: .init("_NSTextInputContextDictationDidEndNotification"),
            object: nil
        )

        _ = indicator.perform(
            Selector(("dictationStateDidChange:")),
            with: notification
        )
    }
}
```

## Notes and Warnings
* This project relies on private AppKit APIs and undocumented notifications.
* It is not App Store safe and may break across macOS releases.
* The code is intended for experimentation and learning purposes only.
