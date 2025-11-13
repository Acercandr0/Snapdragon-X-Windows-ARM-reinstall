# How to Reinstall Windows 11 (ARM) on Snapdragon X Devices

This guide uses the Samsung GalaxyBook 4 Edge as an example, but it works the same for other devices with a Snapdragon X processor.

I screwed up my laptop by doing a clean installation of Windows as if it were an x86 machine, but thanks to me messing everything up, you get to enjoy this tutorial right now 👌

## Prerequisites

- **Drivers for your device:**
  - Download from [Samsung GalaxyBooks Download Center](https://www.samsung.com/global/galaxybooks-downloadcenter/?siteCode=us)
  - You need **TWO different driver packages:**
    - **WinPE Drivers** - for the installation environment
    - **Windows Drivers** - for the installed operating system
  - Extract both driver packages to separate folders on your PC
- [CreatePartitions script](https://github.com/Acercandr0/GalaxyBook-4-Edge-Fresh-Windows-reinstall/blob/main/CreatePartitions.txt) and [ApplyImage script](https://github.com/Acercandr0/GalaxyBook-4-Edge-Fresh-Windows-reinstall/blob/main/ApplyImage.bat)
- 1 USB HUB (or two flash drives with USB-C ports)
- 2 USB Flash drives (16GB or more)
- A working PC with Windows or a virtual machine on macOS/Linux

## Step 1: Download and Prepare Drivers

**⚠️ CRITICAL: You need TWO different driver packages!**

1. **Go to Samsung GalaxyBooks Download Center:**
   - Visit [https://www.samsung.com/global/galaxybooks-downloadcenter/?siteCode=us](https://www.samsung.com/global/galaxybooks-downloadcenter/?siteCode=us)
   - Select your device model (e.g., Galaxy Book4 Edge)

2. **Download WinPE Drivers:**
   - Look for **"WinPE Drivers"**
   - Download and extract to `C:\Drivers_WinPE`

3. **Download normal Drivers:**
   - Look for **"Windows 11 Drivers"**
   - Download and extract to `C:\Drivers`

## Step 2: Create WinPE Bootable USB (First USB)

1. **Install Required Tools:**
   - From your Windows device, install the Windows Assessment and Deployment Kit (Windows ADK) and the Windows PE add-on
   - Follow the [official guide](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install) for installation instructions

2. **Launch Deployment and Imaging Tools Environment:**
   - Open the "Deployment and Imaging Tools Environment" app with administrator privileges

3. **Create and prepare WinPE:**
   ```cmd
   cd "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\arm64"
   copype arm64 C:\WinPE_arm64
   ```

4. **Mount the WinPE boot image:**
   ```cmd
   Dism /Mount-Image /ImageFile:"C:\WinPE_arm64\media\sources\boot.wim" /index:1 /MountDir:"C:\WinPE_arm64\mount"
   ```

5. **Inject drivers into WinPE (CRITICAL STEP):**
   
   **⚠️ IMPORTANT:** WinPE needs drivers to detect USB drives and storage devices. Without this step, you won't be able to see your second USB drive during installation!
   
   - Make sure your drivers backup from Step 1 is available at `C:\Drivers_WinPE`
   - Inject the drivers into WinPE:
   
   ```cmd
   Dism /Image:"C:\WinPE_arm64\mount" /Add-Driver /Driver:"C:\Drivers_WinPE" /Recurse
   ```
   
   **What this does:**
   - Adds USB controller drivers so WinPE can detect your USB drives
   - Adds storage drivers so WinPE can see your NVMe/SSD
   - Ensures all hardware is accessible during the installation process
   
   **Note:** DISM will automatically filter and install only the drivers compatible with WinPE. The same driver pack works for both WinPE and the full Windows installation.

6. **Unmount and save WinPE changes:**
   ```cmd
   Dism /Unmount-Image /MountDir:"C:\WinPE_arm64\mount" /commit
   ```

7. **Create the WinPE ISO:**
   ```cmd
   MakeWinPEMedia /ISO C:\WinPE_arm64 C:\WinPE_arm64.iso
   ```

8. **Locate the WinPE ISO:**
   - The WinPE ISO should now be available at `C:\WinPE_arm64.iso`

9. **Burn the WinPE ISO to the first USB:**
   - Use [Rufus](https://rufus.ie/en/) to burn `C:\WinPE_arm64.iso` to your first USB drive
   - This USB will be used to boot into the installation environment

## Step 3: Prepare Windows Image with Drivers (Second USB)

### 3.1: Download Windows 11 ARM ISO

1. **Download WinARM ISO:**
   - Download the ISO from the [Official Microsoft site](https://www.microsoft.com/software-download/windows11arm64)

### 3.2: Extract and Prepare the Install.wim

1. **Mount the Windows 11 ARM ISO:**
   - Right-click the downloaded Windows ARM ISO file
   - Select "Mount" to mount the ISO as a virtual drive

2. **Locate and copy the Install.wim file:**
   - Open the mounted ISO drive in File Explorer
   - Navigate to the `sources` folder
   - Copy the `install.wim` file to a convenient location on your PC, for example, `C:\wim`

### 3.3: Inject Drivers into Install.wim

**This is the critical step that ensures Windows will have all necessary drivers after installation.**

1. **Open Command Prompt as Administrator:**
   - Right-click on the Windows icon and select "Command Prompt (Admin)"

2. **Create a Mount Directory:**
   ```cmd
   mkdir C:\wimmount
   ```

3. **Mount the Install.wim file:**
   ```cmd
   dism /mount-wim /wimfile:C:\wim\install.wim /index:1 /mountdir:C:\wimmount
   ```
   
   **Note:** `/index:1` refers to the Windows edition in the WIM file. If your WIM contains multiple editions, you may need to change this number.

4. **Inject drivers into the mounted image:**
   
   **⚠️ CRITICAL: Use the NORMAL drivers (not WinPE drivers) that you exported in Step 1**
   
   - Make sure your drivers backup from Step 1 is available at `C:\Drivers`
   - Run the following command to inject ALL drivers recursively:
   
   ```cmd
   dism /image:C:\wimmount /add-driver /driver:C:\Drivers /recurse
   ```
   
   This will scan all subdirectories in `C:\Drivers` and inject compatible drivers into the Windows image.
   
   **What this does:**
   - Integrates drivers directly into the Windows installation
   - Ensures hardware will be recognized during and after Windows setup
   - Eliminates the need to manually load drivers during installation

5. **Unmount and commit changes:**
   ```cmd
   dism /unmount-wim /mountdir:C:\wimmount /commit
   ```
   
   **⚠️ IMPORTANT:** Do NOT close the Command Prompt window until this process completes. It may take several minutes.

6. **Verify the modified WIM file:**
   - The modified `install.wim` with integrated drivers should now be at `C:\wim\install.wim`

### 3.4: Prepare the Second USB Drive

1. **Download the scripts:**
   - Download the [CreatePartitions script](https://github.com/Acercandr0/GalaxyBook-4-Edge-Fresh-Windows-reinstall/blob/main/CreatePartitions.txt) and [ApplyImage script](https://github.com/Acercandr0/GalaxyBook-4-Edge-Fresh-Windows-reinstall/blob/main/ApplyImage.bat)

2. **Format the second USB:**
   - Format your second USB drive as **NTFS**

3. **Copy the files to the second USB:**
   - Copy your **modified** `install.wim` file (from `C:\wim\install.wim`) to the root of the second USB
   - Copy `CreatePartitions.txt` to the root of the second USB
   - Copy `ApplyImage.bat` to the root of the second USB
   
   Your second USB should now contain:
   ```
   USB:\
   ├── install.wim (with drivers injected)
   ├── CreatePartitions.txt
   └── ApplyImage.bat
   ```

## Step 4: Install Windows on Your Snapdragon Device

1. **Plug both USB drives:**
   - With the PC off, connect your USB HUB with both USB drives:
     - **First USB:** WinPE bootable drive (from Step 2)
     - **Second USB:** Drive with install.wim and scripts (from Step 3)

2. **Enter the BIOS:**
   - Turn on your computer and press **F2** repeatedly

3. **Configure BIOS settings:**
   - Disable **Secure Boot**
   - Disable **Fast Boot**

4. **Change the boot order:**
   - Set the **WinPE USB** (first USB) as the primary boot device

5. **Save changes and reboot:**
   - Save BIOS settings and restart
   - The system should boot into WinPE (you'll see a command prompt on a blue background)

6. **Identify the USB drive letter:**
   - In the WinPE command prompt, run:
   
   ```cmd
   diskpart
   list volume
   ```
   
   - Look for the volume that shows your second USB drive (the one with install.wim)
   - Note the drive letter (e.g., `D:`, `E:`, `F:`, etc.)
   - Run `exit` to exit diskpart

7. **Execute installation commands:**
   - Replace `D:` in the following commands with the correct drive letter you identified:
   
   ```cmd
   diskpart /s D:\CreatePartitions.txt
   D:\ApplyImage.bat D:\install.wim
   ```
   
   - The first command creates the proper partition layout
   - The second command applies the Windows image with drivers to your disk
   - This process may take 10-20 minutes

8. **Finalize installation:**
   - When the process completes successfully, you'll see a message
   - Run `exit` to close the command prompt
   - **IMPORTANT:** Disconnect both USB drives BEFORE the system restarts
   - The system will reboot and start the Windows 11 ARM setup with all drivers pre-installed! 🎉

## Troubleshooting

### If Windows doesn't boot after installation:
- Make sure you disconnected both USB drives before the reboot
- Re-enter BIOS and re-enable Secure Boot if needed
- Verify you used the correct drivers for your device model

### If hardware isn't recognized:
- Double-check that you injected the correct drivers in Step 3.3
- Verify the drivers were from the same device model or compatible Snapdragon X device
- Try re-doing Step 3 with drivers from the manufacturer's website

### If you get "disk not found" errors:
- Make sure you're using the correct drive letter in Step 4.7
- Verify the second USB is formatted as NTFS
- Check that all three files (install.wim, CreatePartitions.txt, ApplyImage.bat) are in the root of the USB

### If WinPE doesn't detect your USB drives:
- **This is the most common issue!** It means drivers weren't properly injected into WinPE in Step 2.5
- Go back to Step 2 and make sure you ran the driver injection command for boot.wim
- Verify that `C:\Drivers` contains the exported drivers from Step 1
- Try using a different USB drive or USB port (avoid USB hubs if possible during troubleshooting)
- Some USB 2.0 drives work better than USB 3.0 in WinPE environments

## Summary of What Each USB Does

- **First USB (WinPE):** Boots into a minimal Windows environment that allows you to run diskpart and deployment tools. **Now includes drivers** so it can detect your second USB and storage devices.
- **Second USB (Data):** Contains the Windows installation image (install.wim with drivers), partition script, and deployment script

## Notes

- **Driver Usage Clarification:**
  - **Step 1 (Export):** You export ALL drivers from your working Snapdragon device - we'll use the same driver pack for both WinPE and Windows installation
  - **Step 2.5 (Inject into WinPE/boot.wim):** Drivers are injected into WinPE so it can detect USB drives and storage during installation. DISM automatically filters and installs only WinPE-compatible drivers.
  - **Step 3.3 (Inject into install.wim):** The same drivers are injected into the Windows installation image so all hardware works after Windows is installed

- **Why inject drivers in both places?**
  - **boot.wim (WinPE):** Needs drivers to see USB drives and disks during the installation process
  - **install.wim (Windows):** Needs drivers so hardware works after Windows boots up
  - Using the same driver pack for both is simpler and ensures compatibility

- The `CreatePartitions.txt` script handles the disk partitioning automatically with the proper ARM64 partition layout

- The `ApplyImage.bat` script deploys the Windows image to the correct partition

- This process preserves the proper partition layout required for ARM devices

---

**Credits:** Created after I broke my laptop trying to install Windows like an x86 machine 😅
