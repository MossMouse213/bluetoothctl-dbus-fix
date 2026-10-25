# Bluetooth headset connected but missing from sound outputs

On Omarchy/Arch Linux, a headset can reconnect over Bluetooth without appearing
in the sound output selector. This guide records one confirmed recovery with
BlueZ, PipeWire, and WirePlumber on a 2019 Mac Pro (MacPro7,1).

This is separate from the `bluetoothctl` NULL D-Bus connection crash fixed by
the patches in this repository. No client rebuild was needed for this recovery.

## Check whether this matches your problem

Run from a normal host terminal:

```sh
bluetoothctl devices
bluetoothctl info YOUR_HEADSET_ADDRESS
pactl list cards short
pactl list sinks short
journalctl -u bluetooth --since '-10 min' --no-pager
```

Replace `YOUR_HEADSET_ADDRESS` with your own device address. In the confirmed
case:

- The Microsoft Modern Wireless Headset showed `Paired: yes`, `Connected: yes`,
  `Blocked: no`, and `Trusted: no`.
- PipeWire had no Bluetooth audio card or playback sink for the headset.
- BlueZ logged `profiles/audio/a2dp.c:auth_cb() Access denied:
  org.bluez.Error.Canceled` during reconnection attempts.

These observations suggest service authorization prevented audio setup. An
untrusted device can still work when authorization succeeds, so `Trusted: no`
alone does not establish the cause of every missing audio device.

## Recovery for your own already-paired headset

Trust only a device you recognize and intend to reconnect automatically.
Trust is persistent and allows future service connections without repeated
authorization prompts. The following commands briefly disconnect the headset:

```sh
bluetoothctl trust YOUR_HEADSET_ADDRESS
bluetoothctl disconnect YOUR_HEADSET_ADDRESS
bluetoothctl --timeout 15 connect YOUR_HEADSET_ADDRESS
```

Alternatively, use Bluetooth Manager to mark the headset trusted, then
disconnect and reconnect it.

Verify the result:

```sh
bluetoothctl info YOUR_HEADSET_ADDRESS
pactl list cards short
pactl list sinks short
wpctl status
```

In our test, `Trusted: yes` persisted, a `bluez_card` and `bluez_output`
returned, the active profile was `a2dp-sink`, and the headset became the
default playback device. The user confirmed the recovery worked. If it is
listed but not selected, choose it in your sound output settings.

To reverse the trust change:

```sh
bluetoothctl untrust YOUR_HEADSET_ADDRESS
```

## Verify playback and reconnection

Play audio, power the headset off and on, then check the output selector and
`wpctl status` again. Repeat if you need to establish reliability; one successful
reconnection does not prove every future reconnection or resume will succeed.

Suspend/resume remains unverified. This guide does not establish support for
the Mac's internal Bluetooth controller: the active controller in this test
used Realtek RTL8761BU firmware.

## Microphone and dictation verification (2026-10-24)

The Microsoft Modern Wireless Headset microphone was subsequently verified
working, including audio capture and local Voxtype transcription. The user
confirmed that the headset was working.

At the start of this test, the headset used `a2dp-sink` for music playback,
and the saved default input pointed to an unavailable wired headset microphone.
Switching the Bluetooth card to `headset-head-unit` (HSP/HFP, mSBC), selecting
its Bluetooth input as the default source, and unmuting that input enabled
microphone capture. Input volume was set to 100%.

To inspect and configure your own headset from a normal host terminal:

```sh
pactl list cards
pactl list sources short
# Replace these placeholders with the names reported above.
pactl set-card-profile YOUR_BLUEZ_CARD headset-head-unit
pactl set-default-source YOUR_BLUEZ_INPUT
pactl set-source-mute @DEFAULT_SOURCE@ 0
pactl set-source-volume @DEFAULT_SOURCE@ 100%
pactl get-default-source
pactl get-source-mute @DEFAULT_SOURCE@
```

Use a profile actually listed for your headset. A2DP provides higher-quality
playback; the tested headset required HSP/HFP for microphone use, with reduced
playback quality while that profile was active.

PipeWire showed an active capture stream from the headset. Local recordings
at 16 kHz, mono, signed 16-bit PCM contained speech and were successfully
transcribed by Voxtype using the installed `base.en` model. The requested
“taco cat” test phrase was not accurately recognized, so this confirms working
capture and transcription rather than exact phrase accuracy.

Voxtype's setup check passed, its user service was running and enabled at
login, and the existing Hyprland shortcuts were present: hold F9 to record
and release to transcribe, or use Super+Ctrl+X to toggle recording. These
checks did not include an end-to-end test of inserting text into an application.

If the headset is already trusted or the audio output still does not return,
capture the same diagnostics and investigate that failure rather than repeatedly
re-pairing devices or changing unrelated audio configuration.

## References and reporting

- [BlueZ device properties, including Trusted](https://github.com/bluez/bluez/blob/master/doc/org.bluez.Device.rst)
- [BlueZ bluetoothctl commands](https://github.com/bluez/bluez/blob/master/doc/bluetoothctl.rst)
- [WirePlumber Bluetooth configuration](https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/bluetooth.html)

When sharing logs, redact device addresses and personal information. This
repository does not include the tested headset's address or pairing keys.
This recovery is a troubleshooting observation, not a verified Omarchy code bug.

Documented for MossMouse213 with assistance from GPT-6 via Codex.
