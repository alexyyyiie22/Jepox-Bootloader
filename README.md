================================================================================
                    JEXOP MULTI-STAGE SECURE PC BOOTLOADER
                             - SOURCE RELEASE -
================================================================================

1. OVERVIEW
-----------
Jexop Bootloader is a bare-metal, highly optimized UEFI secure-boot environment 
designed for modern x86_64 PCs running Windows. 

Unlike traditional flat bootloaders, Jexop utilizes an advanced Multi-Stage 
Verification Chain split into three distinct zones (Low, Middle, and Final). 
This architecture isolates hardware security checks, offers an automated bypass 
optimization framework for developers, and includes a built-in recovery shell.


2. SYSTEM ARCHITECTURE & STAGE MAP
----------------------------------
The entire execution pipeline is split into 12 distinct sub-stages:

[ ZONE A: LOW-LEVEL ZONE ]
 • Stage 1: Initial Hardware Interrupt Hook. Scans the keyboard controller immediately
            at power-on. If custom physical hardware key combinations are pressed,
            it intercepts the boot loop and routes directly to Stage 4 (HELOP).
 • Stage 2: Verification Logic Gate. Interrogates the motherboard's NVRAM 
            registers for the "JexopVerify" variable state.
            -> If JexopVerify == 0 (OFF): Triggers the OPTIMIZATION SHORTCUT. Bypasses 
               the entire Middle-Level security zone completely to save boot time.
            -> If JexopVerify == 1 (ON): Follows the default secure architecture.
 • Stage 3: Outbound Integrity Handshake. Package cryptographic memory variables 
            and hands execution pointer over to the Middle-Level Gatekeeper.
 • Stage 4 (HELOP RECOVERY SHELL): Bare-metal rescue environment. Features options to:
            (1) Flash New Firmware To Boot Slot (SPI Flash Write)
            (2) Turn Off Boot Verification (NVRAM Flag Toggle)
            (3) Install Binaries To Boot Slot (Direct FAT32 EFI Partition Write)
            (4) Delete Option Configs / Factory Reset
            (5) System Power/Reboot Routine Management

[ ZONE B: MIDDLE-LEVEL ZONE (GATEKEEPER) ]
 • Stage 1: Token Parsing & Validation. Receives cryptographic tokens from Low-Level. 
            If the calculated hash fails to match the hardware token, it drops instantly.
 • Stage 2: The Kill Switch Engine. Executed if security hashes are corrupted or tampered.
            Locks the CPU registers, prevents OS loading, and allows instant hard 
            system shutdown or redirection to the authenticated HELOP shell.
 • Stage 3: Verification Chain Cleared. Flushes tracking buffers and safely hands 
            execution over to the final system layer.

[ ZONE C: FINAL BOOTLOADER ZONE (OS LOCATOR) ]
 • Stage 1: Device Pool Mapping. Queries the UEFI protocol layer to find all active
            GPT-partitioned storage pools (SSDs, NVMe drives, USBs).
 • Stage 2: OS Partition Slot ID. Scans root volumes specifically targeting active 
            operating system boot blocks and system loader signatures.
 • Stage 3: Memory Mapping. Loads the targeted operating system binary byte stream 
            directly into allocated RAM channels using UEFI BootServices.
 • Stage 4: Jexop Launch Execution. Configures CPU execution states and passes total 
            machine command over to "winload.efi". Windows boots cleanly.


3. REPOSITORY STORAGE LAYOUT
----------------------------
To maintain clean compilation pipelines, the source directory is organized as follows:

 ├── include/
 │    ├── final_boot.h      <-- Storage scan and file-loading structures
 │    ├── low_level.h       <-- Hardware key hooks & NVRAM reading configs
 │    └── middle_level.h    <-- Security tokens & Kill Switch definitions
 ├── src/
 │    ├── final_boot.c      <-- Execution of Final Stages 1, 2, 3, and 4
 │    ├── low_level.c       <-- Execution of Low-Level Stages 1, 2, and 3
 │    └── middle_level.c    <-- Execution of Middle-Level Stages 1, 2, and 3
 ├── main.c                 <-- Global Entry point (efi_main) for the firmware
 ├── Makefile               <-- Automated GCC Cross-Compiler construction script
 └── README.txt             <-- This documentation file


4. BUILD PREREQUISITES
----------------------
Because this code interacts directly with raw x86_64 CPU instructions and the 
motherboard firmware interface, it cannot be compiled using standard application toolchains.

You must compile this inside a Linux terminal environment (Ubuntu, Debian, or Windows WSL).

Run the following command to install the industry-standard UEFI cross-compiling assets:
$ sudo apt update && sudo apt install build-essential gnu-efi


5. COMPILATION AND PACKAGING
----------------------------
Step 1: Open your terminal inside this root project directory.
Step 2: Compile the entire codebase using the automated build script by typing:
        $ make

The script will automatically grab the files from 'src/' and 'include/', link your 
entry blocks together, and generate a standalone, valid UEFI binary file:
-> BOOTX64.EFI

To clean up temporary builder object logs later, run:
        $ make clean


6. DEPLOYMENT AND FIRMWARE UPDATE FILE STRUCTURE
------------------------------------------------
To run the bootloader live on a target test machine:
1. Format a storage drive or USB drive partition as FAT32.
2. Drop your compiled binary file into this exact structural path layout:
   📁 (Drive Root)
    └── 📁 EFI
         └── 📁 BOOT
              └── ⚙️ BOOTX64.EFI

CREATING UPDATE ASSETS FOR HELOP RECOVERY:
When utilizing Option (3) inside the HELOP recovery shell, Jexop looks for uncompressed 
multi-file Tape Archives (.tar). You can bundle firmware files or text options together 
on a PC using standard packaging utilities:
$ tar -cvf jexop_update.tar BOOTX64.EFI options.txt

Because the Jexop core is equipped with a lightweight 512-byte header data parser loop, 
it will split the update file apart directly out of memory and flash the segments safely.


7. DEVELOPER DISCLAIMER
-----------------------
Flashing custom binaries to hardware structures or editing raw EFI paths can cause 
system instability if paths are misaligned. It is highly recommended to test all builds 
inside virtualized sandboxes like QEMU or VirtualBox with EFI emulation checked on before 
running it on your daily desktop machine.

By Alexyyyiie22.
