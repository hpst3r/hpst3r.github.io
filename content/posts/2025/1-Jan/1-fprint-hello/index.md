---
title: "Windows Hello fingerprint 'too similar to one that's already registered' with no fingerprints registered"
date: 2025-01-05T10:10:10-00:00
draft: false
---

# Problem
Windows Hello fingerprint/face broke themselves after a hardware change (reenabling iGPU.) Not sure why. Fingerprint still working in UEFI (as supervisor authentication.)
Able to easily reregister Hello face ID. Not able to reregister Hello fingerprint ID with error "this fingerprint is too similar to one that's already registered" when I attempt to register any fingerprint. Fingerprint is NOT cleared from the TPM (still usable for power on/supervisor authentication.)

Device: ThinkPad P1 Gen 4 w/ Synaptics UWP WBDI fingerprint reader (synaumdf.cat, WUDFRd.sys, AuthenticateFAM_SecureFP.dll) using 6.0.45.1136 driver (1/30/24)
# Solution
Clear stored fingerprint data from the fingerprint reader itself. This is usually an option in the UEFI.
# Troubleshooting
Removed Hello fingerprint.

Attempted to add my fingerprint again. Failed. "this fingerprint is too similar to one that's already registered"

Disabled Hello - removed PIN.

No change.

Stopped Winbiosvc, deleted DBs in `C:\Windows\System32\WinBioDatabase`, restarted service. No change.

Decrypted my drives. Cleared TPM.

No change.

Duh. Fingerprint data is saved to the fingerprint reader itself.

Cleared stored fingerprints from FPR itself from the UEFI. Tried again and was able to register my fingerprint again.

Fun.