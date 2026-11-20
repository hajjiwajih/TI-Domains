# ThinkPad T14 Gen 3 (Intel) — Fedora 44 Dolby Audio with EasyEffects

**Last documented:** 20 November 2026  
**System:** Lenovo ThinkPad T14 Gen 3 Intel, Fedora 44 Workstation  
**Status:** Hardware-matched Dolby speaker correction generated successfully; autoload configured. EasyEffects login autostart was advised but **not yet verified** in the conversation.

---

## 1. Goal and outcome

Recreate the laptop's OEM Dolby speaker tuning on Fedora using the open-source [`speaker-tuning-to-easyeffects`](https://github.com/antoinecellerier/speaker-tuning-to-easyeffects) converter and EasyEffects (Flatpak).

**Successfully completed:**

- Identified the actual internal speaker and audio codec.
- Confirmed that EasyEffects is installed via **Flatpak**, not as an RPM.
- Extracted Lenovo Dolby tuning from `n3bao30w.exe` with `innoextract`.
- Matched the tuning XML to the laptop's **exact audio codec/subsystem IDs**.
- Generated `Dolby-Balanced`, `Dolby-Detailed`, and `Dolby-Warm` EasyEffects output presets and their impulse-response files.
- Configured the **internal speaker sink** to autoload `Dolby-Balanced`.
- Configured a `Nothing` fallback preset for other outputs.

**Still to confirm:** Login autostart/background operation was flagged by `--doctor`; the setting was recommended but not confirmed enabled. The actual listening comparison was not reported. Package-cleanup commands were discussed but not confirmed executed.

## 2. Confirmed hardware and audio stack

| Property | Value |
|---|---|
| Lenovo product ID | `21AJSDE800` |
| Model family | ThinkPad T14 Gen 3 |
| Processor | Intel Core i7-1270P (12th Gen) |
| OS | Fedora Linux 44 (Workstation Edition) |
| Kernel at diagnosis | `7.3.4-200.fc44.x86_64` |
| ALSA sound card | `sof-hda-dsp` |
| Audio codec | Realtek **ALC257** — `0x10EC0257` |
| Codec subsystem | **`0x17AA22E8`** |
| PCI audio controller subsystem | **`17AA:22E8`** |
| Speakers | One internal stereo pair, inferred from Linux codec pins |
| PipeWire / WirePlumber | `1.6.11` / `0.5.19` |
| EasyEffects | `8.2.8`, installed through **Flatpak** |
| EasyEffects Flatpak ID | `com.github.wwmm.easyeffects` |

**Internal speaker sink** (exact name detected at diagnosis):

```text
alsa_output.pci-0000_00_1f.3-platform-skl_hda_dsp_generic.HiFi__Speaker__sink
```

The system also exposed HDMI sinks and the EasyEffects virtual sink (`easyeffects_sink`). An `SUSPENDED` sink is generally normal when it is idle. `--doctor` reported no audio dropouts during its brief idle check; this is not a guarantee under all playback conditions.

### Useful inspection commands

```bash
echo "Model: $(cat /sys/class/dmi/id/product_name)"
lscpu | grep 'Model name' | head -1
flatpak info com.github.wwmm.easyeffects
pactl list short sinks
```

> `rpm -q easyeffects` returned **package not installed**, despite EasyEffects being available: the installation is the Flatpak edition.

## 3. Converter installation and prerequisites

Converter checkout used:

```text
~/Documents/speaker-tuning-to-easyeffects/
```

Project: https://github.com/antoinecellerier/speaker-tuning-to-easyeffects

Version observed during diagnosis:

```text
speaker-tuning-to-easyeffects v2026.10.1-31-g486168e
```

Setup commands recommended during the process:

```bash
sudo dnf install git python3-numpy python3-scipy \
  python3-rich python3-rich-argparse pipewire-utils

cd ~/Documents
git clone https://github.com/antoinecellerier/speaker-tuning-to-easyeffects.git
cd speaker-tuning-to-easyeffects
python3 dolby_to_easyeffects.py --doctor
```

The first conversion attempt failed because `innoextract` was missing:

```text
Error: Unpacking n3bao30w.exe needs innoextract, which is not installed.
```

Fix:

```bash
sudo dnf install innoextract
```

**Note:** The successful actual package was **`n3bao30w.exe`**, not the previously suggested `n3ga127w.exe`. Do not substitute another driver package without verifying that the converter identifies the exact codec/subsystem matching the laptop.

## 4. Successful conversion (actual command)

The Windows driver archive was in the converter's working directory when the following command was run:

```bash
cd ~/Documents/speaker-tuning-to-easyeffects
python3 dolby_to_easyeffects.py n3bao30w.exe --autoload
```

Important output:

```text
Unpacking n3bao30w.exe (Inno Setup installer, innoextract 1.9)…
Unpacked 219 tuning files into .../driver-cache/n3bao30w-27c1eca2dad6
Matched tuning XML: .../DEV_0257_SUBSYS_17AA22E8_PCI_SUBSYS_22E817AA.xml
Endpoint: internal_speaker
Profile: dynamic
Correction check passed: all 3 filters match the shape of the curve
  your tuning file asks for, within 0.03 dB
```

The critical matching filename is:

```text
DEV_0257_SUBSYS_17AA22E8_PCI_SUBSYS_22E817AA.xml
```

This is an **exact codec/subsystem match** to this laptop's ALC257 / `17AA:22E8` configuration. It confirms that the right tuning definition was selected; it does **not** mean the speaker's physical acoustic output was independently measured.

### Extracted tuning details

| Feature | Value / behavior |
|---|---|
| Selected Dolby sound mode | `dynamic` (first listed; not proved to be the Windows default) |
| Voicing profiles | Balanced, Detailed, Warm |
| Voicing strength (`ieq-amount`) | 10% |
| Speaker correction | Approx. -6 dB to +7 dB in the source curve |
| Speaker-protection high-pass filter | 150 Hz |
| Other speaker PEQ | 830 Hz: -10.5 dB; 1830 Hz: -3 dB; 1450 Hz: -4 dB |
| Dialog enhancer | In source configuration but disabled by converter by default |
| Autogain / volume leveler | In source configuration but disabled by converter by default |
| Multiband compressor (`mbc`) | 2 bands, enabled in generated preset |
| Regulator | Frequency-band-based protective limiting |
| Loudness (`volmax-boost`) | +6 dB in generated preset |
| Multichannel surround boost | Intentionally skipped for ordinary stereo playback |

The converter reports that the three profile voicings have different tonal shapes but equal nominal voicing strength. For general use, **start with `Dolby-Balanced`**; the other two profiles are optional preferences.

## 5. Files generated by the converter

EasyEffects Flatpak data directory:

```text
~/.var/app/com.github.wwmm.easyeffects/data/easyeffects/
```

Generated output presets:

```text
output/Dolby-Balanced.json
output/Dolby-Detailed.json
output/Dolby-Warm.json
output/Nothing.json
```

Generated impulse responses:

```text
irs/Dolby-Balanced-811d6901.irs
irs/Dolby-Detailed-366bed9a.irs
irs/Dolby-Warm-83d7b4f9.irs
```

Speaker-specific autoload rule:

```text
autoload/output/alsa_output.pci-0000_00_1f.3-platform-skl_hda_dsp_generic.HiFi__Speaker__sink:Speaker.json
```

The converter also updated its fallback-preset configuration in:

```text
~/.var/app/com.github.wwmm.easyeffects/config/easyeffects/db/easyeffectsrc
```

The fallback is `Nothing`, to avoid forcing the laptop speaker's acoustic correction onto headphones, Bluetooth, or other outputs. The converter explicitly instructed that EasyEffects must be restarted for this fallback setting to take effect.

## 6. Finish the configuration

### Restart EasyEffects

```bash
flatpak kill com.github.wwmm.easyeffects
flatpak run com.github.wwmm.easyeffects
```

In the EasyEffects interface:

1. In **Output → Presets**, check that **`Dolby-Balanced`** is active when the built-in speakers are selected.
2. Ensure the global bypass/power button is **off** so that output processing is active.
3. Open **Preferences → Background Service**.
4. Keep **Service Mode** enabled (it was already enabled in the diagnostic).
5. Enable **Autostart on login** (this was **off** in the diagnostic).
6. Close the window normally without intentionally terminating the background process.

The user's last reported diagnostic warning was:

```text
[WARN] Background service
EasyEffects won't reliably keep processing in the background ...
In EasyEffects > Preferences > Background Service,
turn on 'Autostart on login'.
```

**Autostart has not yet been confirmed fixed.** Re-run the diagnostic after changing the setting.

### Verify

```bash
cd ~/Documents/speaker-tuning-to-easyeffects
python3 dolby_to_easyeffects.py --doctor
```

Expected: the generated-preset and selected-preset warnings should disappear, and background service should pass after enabling login autostart. If the selected-preset check still warns, manually select `Dolby-Balanced` while using the internal speakers and recheck.

For an audible A/B test, play known audio at a **moderate** speaker volume and toggle the EasyEffects global bypass. Do **not** add the earlier generic EQ preset on top of this hardware-matched Dolby correction.

## 7. Optional adjustments — only if the sound needs them

The project exposes selective adjustments. Run them using the **same source driver** and reload the regenerated preset afterward:

```bash
# If loud content sounds distorted or crushed:
python3 dolby_to_easyeffects.py n3bao30w.exe --autoload --disable volmax

# If the multiband processing makes music sound unnaturally flat:
python3 dolby_to_easyeffects.py n3bao30w.exe --autoload --disable mbc

# If volume audibly pumps or wobbles:
python3 dolby_to_easyeffects.py n3bao30w.exe --autoload --disable regulator

# If the loudest moments seem overly clamped:
python3 dolby_to_easyeffects.py n3bao30w.exe --autoload --disable coupled-bands
```

Experimental / optional additions (not enabled in the successful setup):

```bash
--enable virtual-bass    # Experimental, may distort
--enable autogain        # Can cause swelling/ducking
--enable dialog          # Boosts speech band, including in music
--enable level-restore   # Experimental, can crackle or distort
```

**Default recommendation:** Retain the successfully generated `Dolby-Balanced` profile unchanged until you've listened to it. Avoid stacking another EQ, bass boost, or loudness processor on top by default.

## 8. Backup, restore, and rollback

### Backup the Dolby configuration

The following command saves the EasyEffects presets, impulse files, autoload rules, and configuration database to a dated compressed archive in `~/Documents`:

```bash
EE="$HOME/.var/app/com.github.wwmm.easyeffects"
tar -czf "$HOME/Documents/t14g3-easyeffects-$(date +%F).tar.gz" \
  -C "$EE" data/easyeffects/output \
  data/easyeffects/irs \
  data/easyeffects/autoload \
  config/easyeffects/db/easyeffectsrc
```

This includes *all* EasyEffects files in those directories, not just the Dolby files. Keep the archive private if your other presets contain personal configuration.

### Restore from your backup

Close EasyEffects first, then substitute your archive filename:

```bash
flatpak kill com.github.wwmm.easyeffects
EE="$HOME/.var/app/com.github.wwmm.easyeffects"
tar -xzf "$HOME/Documents/t14g3-easyeffects-YYYY-MM-DD.tar.gz" -C "$EE"
flatpak run com.github.wwmm.easyeffects
```

### Remove only the converter-managed files

The converter's documented uninstall command is:

```bash
cd ~/Documents/speaker-tuning-to-easyeffects
python3 dolby_to_easyeffects.py --uninstall
```

This is intended to remove files managed by the converter while leaving unrelated custom presets intact. Consider creating a backup first.

## 9. Fedora package cleanup — proposed, not confirmed executed

During the same session, optional system cleanup was discussed. **No deletion was confirmed in the conversation.**

```bash
# Preview automatic RPM removals; inspect the transaction first
sudo dnf autoremove --assumeno

# Only after reviewing and approving the proposed list
sudo dnf autoremove

# Remove unused Flatpak runtimes (review the transaction)
flatpak uninstall --unused

# Clear downloaded RPM package caches and repo metadata
sudo dnf clean all

# Optional: remove extractor only if no longer needed
sudo dnf remove innoextract
```

Retain PipeWire, WirePlumber, ALSA/SOF support and the EasyEffects Flatpak. **Do not remove** the EasyEffects application data folder because it contains your generated Dolby presets.

You may remove `innoextract` after generating profiles, but you will need it again if you unpack a new Inno Setup-based Windows audio driver.

To inspect installed/unneeded packages before a deeper cleanup:

```bash
dnf repoquery --userinstalled
flatpak list --app
dnf repoquery --unneeded
```

## 10. Quick reference

```bash
# Recheck health
cd ~/Documents/speaker-tuning-to-easyeffects
python3 dolby_to_easyeffects.py --doctor

# Rebuild three Dolby presets and the device autoload rule
python3 dolby_to_easyeffects.py n3bao30w.exe --autoload

# View available Dolby profiles
python3 dolby_to_easyeffects.py n3bao30w.exe --list

# Restart EasyEffects Flatpak
flatpak kill com.github.wwmm.easyeffects
flatpak run com.github.wwmm.easyeffects

# Check which outputs PipeWire sees
pactl list short sinks

# Remove converter files, if intentionally reverting
python3 dolby_to_easyeffects.py --uninstall
```

### Final status as of this document

- **Hardware ID / tuning match:** confirmed.
- **Preset generation and correction validation:** confirmed.
- **Internal-speaker autoload rule / fallback file generation:** confirmed.
- **Login autostart and audible improvement:** not yet confirmed.
- **System cleanup:** recommendations only; not recorded as executed.

---

*This is a record of the commands and outputs shared in the support conversation, not an independent listening or hardware measurement report.*
