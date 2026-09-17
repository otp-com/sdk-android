# otp.com Android SDK

Verifies a phone number or an email address with a one-time code. The channel is chosen by your
account routing, so you pass the recipient and nothing else.

Requires **Android API 26** (8.0) and Java 17 to build.

API 26 rather than 24, and the reason is the security floor: the SDK proves that a send came from
your app with a hardware-backed Keystore key, and hardware attestation is only guaranteed from 8.0. A
device below it may have no hardware attestation at all and would fall back to a software key whose
signing key sits in the AOSP source, which proves nothing. Advertising a floor that is silently not a
floor on part of the fleet is worse than not supporting that part of the fleet.

## Before you start

You need two keys, and they are not interchangeable.

1. Sign in at [panel.otp.com](https://panel.otp.com?utm_source=github-sdk-android). If you do not
   have an account yet, [create one](https://panel.otp.com/signup?utm_source=github-sdk-android); it
   takes a minute and comes with sandbox credit.
2. Open your app, then **API Keys**, and create two:
   - a **publishable key** (`otp_pk_live_…`) for this SDK, which goes in your app;
   - a **server key** (`otp_live_…`) for your own backend, which never leaves it.
3. While you are integrating, use the `otp_pk_test_…` and `otp_test_…` pair instead. Sandbox sends no
   real messages and costs nothing.

The publishable key is meant to be readable: it ships inside your APK, it is scoped to that one app,
and it can only start and answer verifications. It cannot read a recipient and it cannot exchange a
verification, which is why the server key exists and why it stays on your server.

## Install

```kotlin
dependencies {
    implementation("com.otp:sdk-android:0.2.0")
}
```

Nothing to add to your manifest. The SDK declares the internet permission and its own screen host,
and manifest merging does the rest.

## Use it

Configure once, at launch:

```kotlin
OtpClient.configure(context, "otp_pk_live_…")
```

Then run a verification. This presents a screen, sends the code, takes the user's input, and returns
when it is done:

```kotlin
val verification = OtpClient.verify("+14155552671")
```

If your flow has no phone field yet, let the SDK collect it:

```kotlin
val verification = OtpClient.verify(collecting = RecipientKind.PHONE)   // or EMAIL
```

Both are `suspend` functions, and both throw an `OtpException` of kind `CANCELLED` if the user closes
the screen.

In a Compose app you can place the same screen in your own composition instead, which keeps it inside
your navigation and gives it your own Material theme:

```kotlin
OtpVerification(recipient = "+14155552671") { result ->
    result.onSuccess { verification -> /* verification.token */ }
}
```

The message and the screen follow the same locale, so they never disagree. It defaults to the
device's; pass one to override:

```kotlin
val verification = OtpClient.verify("+14155552671", locale = "tr-TR")
```

The screen speaks English, Turkish, Russian, Arabic, German and French, lays itself out right to left
where the language reads that way, follows the system light and dark appearance, and takes its accent
colour from your panel.

## The one thing to get right

**`verification.token` is the result. Nothing else is.**

Send it to your own backend, which exchanges it with your **server** key:

```
POST https://api.otp.com/api/v1/verifications/exchange
Authorization: Bearer otp_live_…

{ "verification_token": "…" }
```

It answers with what was actually verified:

```json
{
  "otp_id": "6f0d2c5e-1c3a-4f1b-9a2e-6a1f2b3c4d5e",
  "recipient": "+14155552671",
  "recipient_type": "phone",
  "channel": "sms",
  "verified_at": "2026-09-08T19:33:21Z"
}
```

Until you make that call, your backend knows nothing. A success read off a device you do not control
is not evidence, and anyone running a modified build can claim any outcome they like. The token is
short-lived and single-purpose, so treat it as the only thing you trust.

Our [backend SDKs](https://github.com/otp-com?utm_source=github-sdk-android) do this call for you in
Node, PHP, Go and Python.

## Resuming

On the WhatsApp channel the code is not sent until the user messages us, which means they leave your
app and Android may kill it while they are away. Call this when your app becomes visible and they
come back to the screen they left:

```kotlin
val verification = OtpClient.resumeInterrupted()
```

It returns null when there was nothing in flight, which includes a verification that expired while
they were away and one they closed without answering. In a Compose app, `OtpResumedVerification` is
the same thing as a composable.

## Your own screen

The drop-in screen has no view-slot API, because a screen assembled from someone else's slots is
worse than one you wrote. If you want a different screen, build it on the same core the drop-in uses:

```kotlin
val session = OtpSession()
val pending = session.start("+14155552671")

pending.codeLength         // how many boxes to draw, and it is not always 6
pending.expiresAt          // count down from this
pending.resendAvailableAt  // null means it can never be resent, not "resend now"
pending.handoffUrl         // WhatsApp only: open this, the code follows

when (val outcome = session.submit(entered)) {
    is CodeSubmission.Verified -> outcome.verification.token
    // A wrong code is an outcome, not an error.
    is CodeSubmission.Rejected -> outcome.attemptsRemaining
}
```

Everything a screen needs is on `PendingOtp`, including both deadlines, so no part of this polls.

Two values are worth reading rather than assuming. `codeLength` is your account's setting and it is
not always six. `resendAvailableAt` being null means this verification can never be resent, which is
the opposite of "resend is available now": show no button at all rather than one that always fails.

After a restart you can pick a verification back up without having stored anything yourself:

```kotlin
val inFlight = OtpClient.interrupted     // reads the device, makes no request
if (inFlight != null) {
    val pending = OtpSession().resume(inFlight.otpId)
}
```

## Device integrity

The SDK registers a hardware-backed Keystore key on first use and signs every send with it, and the
API verifies the attestation chain up to Google's root. That is what stops a publishable key lifted
out of your APK from being used outside your app.

You configure nothing for this. Two consequences worth knowing:

- **An emulator cannot produce a proof.** Its attestation cannot be verified, so where a proof is
  required the send is refused with kind `DEVICE_PROOF_REJECTED`. Test that path on hardware.
- **Whether a proof is required follows the key.** A sandbox key (`otp_pk_test_…`) never requires
  one, so an emulator is fine while you integrate. A live key does, once the platform asks for it,
  which is why the last thing to test before going live is a real device with your live key.

## Errors

Every call throws `OtpException`. Read `kind` to decide what to do, and keep `message` for your logs:
it is written for you, not for your user, and it is not translated.

```kotlin
try {
    val verification = OtpClient.verify(recipient)
} catch (error: OtpException) {
    when (error.kind) {
        OtpException.Kind.CANCELLED -> Unit                       // the user closed the screen
        OtpException.Kind.RATE_LIMITED -> wait(error.retryAfterSeconds)
        OtpException.Kind.VALIDATION_FAILED -> showYourOwnFieldError()
        else -> log(error)
    }
}
```

`error.type` carries the API's own name for the failure, which is what tells two failures of the same
kind apart: a `CONFLICT` is either a recipient no channel can reach or a verification that can never
be resent again.

## Support

Docs and status: [otp.com](https://otp.com?utm_source=github-sdk-android). Anything else:
info@otp.com.
