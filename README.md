# Gemini Voice Keyboard for iOS

A complete iOS sample for push-to-talk transcription with Gemini 3.5 Transcribe. It combines a containing app, a custom keyboard, Gemini Live streaming, a batch fallback, durable recovery, and careful cross-process coordination.

> [!IMPORTANT]
> This is a personal developer sample, not a production SDK or an App Store-ready product. The Debug configuration includes optional private iOS integration that makes the cold keyboard handoff unusually smooth on a personal device. Apple requires App Store apps to use public APIs, so that path may be rejected. Release compiles the private handoff code out, but still needs a backend before Gemini requests can work securely.

## What this sample demonstrates

- Real-time microphone transcription with `gemini-3.5-transcribe-live` over the Gemini Live WebSocket API.
- Batch transcription with `gemini-3.5-transcribe` through the Interactions API.
- Conversion of any input route to 16 kHz, mono, little-endian PCM in roughly 100 ms chunks.
- A custom keyboard that controls microphone capture in its containing app through an App Group.
- One-time request authorization, freshness checks, command ordering, cancellation precedence, and safe transcript insertion.
- A protected local WAV written alongside Live streaming, then deleted after success or retained for an explicit retry after failure.
- Recovery across connection loss, audio-route changes, app backgrounding, process termination, and device switching between built-in and Bluetooth inputs.
- Optional Live translation and image text extraction as adjacent examples.

Apple does not give custom keyboard extensions microphone access, even when Full Access is enabled. The containing app therefore owns `AVAudioEngine`; the keyboard only sends authenticated commands and inserts a matching result. See [Architecture](docs/ARCHITECTURE.md) for the full data flow and the invariants that keep the two processes synchronized.

## Start with the Gemini code

The Gemini implementation is intentionally self-contained under [`App/Gemini`](App/Gemini):

| Area | Purpose |
| --- | --- |
| [`Batch`](App/Gemini/Batch) | Builds Interactions API requests, validates responses, and reports service errors. |
| [`Live`](App/Gemini/Live) | Opens the WebSocket, sends setup/audio/activity messages, merges interim/final transcripts, and finalizes safely. |
| [`Audio`](App/Audio) | Owns `AVAudioSession` and `AVAudioEngine`, handles route recovery, writes WAV files, and produces Live PCM chunks. |
| [`Recovery`](App/Recovery) | Tracks finalized recordings that may be retried and stores a bounded result history. |
| [`Relay`](App/Relay) | Coordinates keyboard commands, capture, Live/batch fallback, background time, and result publication. |
| [`Shared/Relay`](Shared/Relay) | Defines the cross-process protocol and its locked App Group store. |

Each large coordinator is split into extensions named for one responsibility. This preserves the tuned state machine while making individual paths easy to find. A detailed file guide lives in [`App/Gemini/README.md`](App/Gemini/README.md).

## Requirements

- Xcode 26.6 or later
- iOS 26 or later
- A physical iPhone for microphone and custom-keyboard behavior
- An Apple Developer team with three bundle identifiers and one shared App Group
- A Gemini API key for personal development
- [XcodeGen](https://github.com/yonaskolb/XcodeGen) when regenerating the committed project

The current Gemini model names and preview APIs can change. Check Google's [audio transcription](https://ai.google.dev/gemini-api/docs/transcribe) and [Live transcription](https://ai.google.dev/gemini-api/docs/live-api/live-transcribe) documentation before adopting the sample.

## Configure a personal build

1. Copy the local configuration:

   ```sh
   cp Config/Secrets.xcconfig.example Config/Secrets.xcconfig
   ```

2. In `Config/Secrets.xcconfig`, set your Apple team, unique bundle identifiers, App Group identifier, and Gemini API key.

3. Register the same bundle IDs and App Group in your Apple Developer account. All three targets must use the same App Group.

4. Regenerate the project:

   ```sh
   xcodegen generate
   ```

`Config/Secrets.xcconfig` is ignored by Git. The Debug API key is still compiled into the app and can be extracted from the installed binary. Use a restricted key, set billing limits, and treat this only as a personal-device convenience.

## Build and test

Run the complete simulator suite:

```sh
./Scripts/test.sh
```

Deploy a signed Debug build to an available paired iPhone:

```sh
xcrun devicectl list devices
./Scripts/deploy-device.sh <core-device-identifier-or-hardware-udid>
```

The deploy script resolves the corresponding Xcode destination, builds, verifies the signature, installs the containing app with both extensions, and launches it.

After the first install:

1. Open **Settings → General → Keyboard → Keyboards → Add New Keyboard**.
2. Add **Gemini Voice** and enable **Allow Full Access**. App Group communication requires it.
3. Open Gemini Voice once and allow microphone access.
4. Switch to the keyboard in a normal text field, tap **Dictate**, speak, and tap the send arrow.

Custom keyboards do not appear in secure password fields, some phone-pad fields, or apps that block third-party keyboards.

## Live, Finish, Cancel, and recovery

Live mode sends audio while you speak. **Cancel** stops the stream, deletes the temporary WAV, discards any partial text, and inserts nothing; it cannot recall audio already sent to Gemini. **Finish** drains queued PCM, waits for an authoritative final result, and falls back to the complete local WAV if the Live result is missing or unreliable.

Only a user-finished recording becomes retryable. A failed finalized WAV is protected, excluded from backup, and listed under **Saved recordings**. A successful result is written to **Recent results** before audio cleanup. Crash-abandoned in-progress audio is deleted rather than uploaded after relaunch.

The capture engine rebuilds its audio graph after route changes, interruptions, and media-service resets. It tries Bluetooth high-quality, Bluetooth mixed, hands-free, and built-in mixed configurations in order while keeping unrelated audio playing when iOS permits it.

## Security and App Store caveats

- Google says not to ship long-lived API keys in mobile apps. Production batch requests need an authenticated backend proxy. Direct Live connections should use backend-issued [ephemeral tokens](https://ai.google.dev/gemini-api/docs/live-api/ephemeral-tokens).
- The personal Debug handoff dynamically accesses private UIKit, SpringBoard, Security, and LaunchServices behavior. It is isolated behind `GEMINI_PERSONAL_DEVICE` and absent from Release.
- Apple states that App Store apps may use only public APIs. Release uses the supported manual return path, but distribution still requires a fresh privacy, background-audio, authentication, and App Review assessment.
- Full Access lets the keyboard use networking and the App Group. This keyboard never sends keystrokes or surrounding document text to Gemini; it stores only short-lived request metadata and SHA-256 context fingerprints.
- Review the included privacy manifests and make your own App Store privacy disclosures. Data handling can differ by Gemini service tier and account configuration.

Relevant primary documentation:

- [Gemini audio transcription](https://ai.google.dev/gemini-api/docs/transcribe)
- [Gemini Live transcription](https://ai.google.dev/gemini-api/docs/live-api/live-transcribe)
- [Gemini API key security](https://ai.google.dev/gemini-api/docs/api-key)
- [Apple custom keyboard open access](https://developer.apple.com/documentation/uikit/configuring-open-access-for-a-custom-keyboard)
- [Apple App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)

## Project policy

This repository is provided as an unsupported, read-only sample. There is no support commitment, roadmap, or guarantee that a current model, iOS release, signing setup, or private integration will keep working. Contributions are not accepted; fork or copy the code and adapt it for your own use.

## License

Licensed under the [Apache License 2.0](LICENSE). You may use, modify, and redistribute the sample subject to that license.
