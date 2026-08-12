# Fix Shokz OpenComm 2 UC triggering the Linux power button

This workaround fixes a Linux issue where a **Shokz OpenComm 2 UC** headset, connected through its USB-A Loop120 receiver, acts like a power button whenever the headset is turned on.

Typical symptoms include an immediate lock, suspend, shutdown prompt, or repeated power-button actions that make the keyboard unusable. Audio may otherwise appear normal.

## Affected setup

- Shokz OpenComm 2 UC
- Shokz Loop120 USB receiver (USB-A or USB-C via an adapter)
- Linux systems using `systemd-udevd` and `systemd-logind`, including Ubuntu 24.04

The receiver identifies itself similarly to this:

```text
Shokz Loop120 by Shokz Consumer Control
Vendor ID: 3511
Product ID: 2ef2
```

## Why this happens

The Loop120 receiver exposes a USB HID *Consumer Control* interface. That interface advertises `KEY_POWER`, even though it belongs to headset controls rather than the computer's physical power button.

On systems using the standard udev rule `70-power-switch.rules`, an input device with `ID_INPUT_KEY=1` receives the `power-switch` tag. `systemd-logind` then treats events from that Shokz interface as system power-button events.

## Fix

This rule must run **after** the normal input-identification rules and **before** `70-power-switch.rules`. It changes only the Shokz receiver's key-interface classification, so udev never gives it the `power-switch` tag.

1. Turn the headset off.

2. Create or edit the rule:

   ```bash
   sudoedit /etc/udev/rules.d/69-shokz-no-power-switch.rules
   ```

3. Put this single line in the file, then save and exit:

   ```udev
   ACTION!="remove", SUBSYSTEM=="input", KERNEL=="event*", ENV{ID_VENDOR_ID}=="3511", ENV{ID_MODEL_ID}=="2ef2", ENV{ID_INPUT_KEY}:="0"
   ```

4. Reload udev rules, then physically unplug and reconnect the Loop120 receiver:

   ```bash
   sudo udevadm control --reload
   ```

5. Turn the headset on and confirm that the keyboard and desktop continue to work normally.

## Verify the fix

The event number is not guaranteed to be `event5`; it can change after reconnecting or rebooting. Find the Shokz event(s) with:

```bash
grep -A10 -B2 -i 'Shokz Loop120' /proc/bus/input/devices
```

For the `Consumer Control` event listed there, inspect its properties (replace `eventN`):

```bash
udevadm info --query=property --name=/dev/input/eventN | grep -E '^(ID_INPUT_KEY|TAGS|CURRENT_TAGS)='
```

After the workaround, the Consumer Control event should show `ID_INPUT_KEY=0` and must not show `power-switch` in `TAGS` or `CURRENT_TAGS`.

## Notes

- This does not disable USB audio, the microphone, or the receiver itself.
- It intentionally stops Linux from treating the receiver's Consumer Control interface as a system power switch. Headset media-control behavior may vary by desktop environment.
- If you previously added a Shokz rule that tries only `TAG-="power-switch"` in a `99-*.rules` file, remove it. A rule at order `99` runs after the system power-switch rule and did not reliably clear the tag in this setup.

## Remove the workaround

```bash
sudo rm /etc/udev/rules.d/69-shokz-no-power-switch.rules
sudo udevadm control --reload
```

Then unplug and reconnect the receiver.
