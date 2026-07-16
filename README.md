# iOS 12.1 Beta Static Analysis & Kernel Patching Workspace

quick cheatsheet for reversing the iOS 12.1 Beta 1 (16B5059d) kernel cache on the iPhone XS Max (iPhone11,6). basically just notes on finding offsets for usbliter8 downgrades so i don't have to look this up every 5 minutes.

---

## Links
* the ipsw download link: [https://theapplewiki.com](https://theapplewiki.com/wiki/Beta_Firmware/iPhone/12.x#iPhone_XS_Max)
* the keys + kbags: [https://theiphonewiki.com](https://www.theiphonewiki.com/wiki/PeaceBSeed_16B5059d_(iPhone11,6))

---

## Quick Info
* phone: iPhone XS Max (A12 Bionic / iPhone11,6) 
* board config: d331p
* target ios: iOS 12.1 Beta 1 (16B5059d) 
* exploit: usbliter8 (needs that cheap RP2350 microcontroller board to actually boot, wich i still dont have)

---

## Apps Needed on PC
1. ipsw (github tool to strip apple layers and extract files easily)
2. OpenSSL (to run the decryption command line)
3. Ghidra (or shadow/IDA pro, set it up for ARM:v8:64:LE)

---

## What to Do (According to Claude and 5 minutes of research*)

### Step 1: rip out the kernel
open terminal inside your working folder and run this to grab the kernel out of the beta ipsw:
```bash
ipsw extract iPhone11,6_12.1_16B5059d_Restore.ipsw -k
```
(or just change the extension to .zip and pull out kernelcache.release.iphone11 manually) 

### Step 2: decrypt it
grab the specific beta key and IV from the wiki bookmark above and run this command:
```bash
openssl aes-256-cbc -d -K [WIKI_KEY_HERE] -iv [WIKI_IV_HERE] -in kernelcache.release.iphone11 -out decrypted_kernel.bin
```

### Step 3: load into ghidra
import decrypted_kernel.bin into ghidra. 
* IMPORTANT: manually switch the language option to ARM:v8:64:LE before you click import or the code will look like corrupted text.
* click analyze. go get a coffee, it takes like an hour to parse everything.

### Step 4: search for the targets
use the global search tool inside ghidra to look for these strings:
* The SEP checks: search "AppleSEPManager" or "sep_boot". go to the main start functions and look for the conditional branches right after the handshake fails. save the hex address (offset).
* AMFI checks: search "AppleMobileFileIntegrity" or "amfi_authorize_format". this is where code signing lives. find the offsets so we can patch them out later.

---

## The Python Patch Script
once you find the hex offsets in ghidra, stick them in this template to auto-patch the binary:

```python
import os

TARGET_FILE = "decrypted_kernel.bin"
OUTPUT_FILE = "patched_kernel.bin"

# change this hex value to whatever you found in ghidra
TARGET_OFFSET = 0x1234AB 

# 0x1F2003D5 is just the hex value for a NOP (No Operation) instruction in ARM64
PATCH_BYTES = b'\xd5\x03\x20\x1f' 

def apply_patch():
    if not os.path.exists(TARGET_FILE):
        print("[-] dude, where is the decrypted_kernel.bin file?")
        return

    with open(TARGET_FILE, "rb") as f:
        file_data = bytearray(f.read())

    print(f"[+] overwriting check logic at: {hex(TARGET_OFFSET)}")
    file_data[TARGET_OFFSET:TARGET_OFFSET+len(PATCH_BYTES)] = PATCH_BYTES

    with open(OUTPUT_FILE, "wb") as f:
        f.write(file_data)
    print("[+] done. patched_kernel.bin ready.")

if __name__ == "__main__":
    apply_patch()
```
*PS: It wont work 
---

## Things to Remember so I can lose motivation 
* no SEP = no security: patching out the sep manager makes the phone boot, but Face ID, passcodes, Apple Pay, and cellular signals will be broken on the downgraded version. it's basically just a wifi iPod.
* it's tethered: if the phone dies or reboots, you have to plug it back into the RP2350 microcontroller board to launch the exploit again or it won't boot.
---

# Deployment Sequence 

```
 :DFU Mode] ➔ [RP2350 / usbliter8 Payload Injection] ➔ [PWND SecureROM State]
                                                               │
   ┌───────────────────────────────────────────────────────────┘
   ▼
[Send Patched iBoot via USB] ➔ [iBoot Accepts Unsigned Code]
                                        │
   ┌────────────────────────────────────┘
   ▼
[Send Modified Device Tree (No SEP Node)]
   │
   ▼
[Send Patched Kernel (Stubbed SEP Managers)] ➔ [Successful Boot to iOS 12 Screen]
```
