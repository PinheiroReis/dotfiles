1. Install Windows (version Home) on whole disk
2. Use the `diskmgmt` to shrink C: disk (leaving space enough to Linux)
3. Disable hibernation

        powercfg /H off

4. Fast startup should not appear on Control Panel

5. Power off (not reboot) the machine
6. Check if hibernation is disabled

        (GP "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power")."HiberbootEnabled"
   
        # it should be 0, so it's disabled.

7. Follow the [Linux Installation](./INSTALLATION-MANUAL.md) to install linux in the unallocated space
