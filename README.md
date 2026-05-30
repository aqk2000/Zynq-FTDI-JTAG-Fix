Fixing Zynq FTDI JTAG Not Recognized in Vivado (EEPROM Programming Guide)

📌 The Problem

Many custom Zynq development boards (like ALINX, QMTech, or custom-designed PCBs) feature an integrated USB port labeled UART/JTAG/Power. This port is usually driven by a dual-channel FTDI chip (e.g., FT2232H):

Channel A: Intended for JTAG (FPGA programming and ARM debugging).

Channel B: Intended for Serial UART (Linux console).

The Issue: Out of the box, Windows might recognize the chip as two generic serial ports (USB Serial Converter A and B), but Vivado's hw_server completely fails to detect the board. This happens because the board manufacturer left the FTDI EEPROM unprogrammed or blank, missing the proprietary Xilinx/Digilent JTAG signature.

❌ Common Unsuccessful Workarounds (And Why They Fail)

Before diving into the solution, it's important to understand why standard troubleshooting steps usually fail:

Attempt 1: Changing Strings via FT_PROG

Many engineers try using FTDI's official FT_PROG utility to change the Manufacturer to Digilent and the Product Description to Digilent USB Device.

Why it fails: Vivado's hw_server doesn't just look at USB strings. It specifically looks for a proprietary binary signature inside the EEPROM. FT_PROG cannot generate this Xilinx-specific signature.

Attempt 2: Forcing Drivers via Zadig or Device Manager

Another common approach is forcing Windows to use WinUSB or Xilinx USB Cable drivers on Channel A.

Why it fails: While this satisfies Windows, Vivado still rejects the connection due to the missing EEPROM signature. Furthermore, having these "advanced" drivers installed prevents the official Xilinx EEPROM flashing tool (which we will use below) from communicating with the chip!

✅ The Ultimate Solution: AMD/Xilinx program_ftdi Tool

Xilinx provides a hidden command-line utility inside the Vivado installation path designed exactly for this purpose. Here is the step-by-step guide to fixing your board.

Step 1: Revert to the Raw FTDI Driver (Crucial Prerequisite)

The program_ftdi tool communicates using low-level FTDI libraries. If you have any JTAG, WinUSB, or Digilent drivers installed on Channel A, the tool will throw a 0 devices found error.

Open Windows Device Manager.

Locate Channel A (might be under Programming cables or Universal Serial Bus controllers).

Right-click and select Uninstall device.

⚠️ Critical: Check the box that says "Attempt to remove the driver software for this device".

Unplug and replug the USB cable. Repeat this until the device shows up exactly as USB Serial Converter A.

Step 2: Set the Correct Boot Mode

To prevent the Zynq ARM processor from locking up the JTAG chain during boot:

Locate the BOOT dip-switches or jumpers on your board.

Set them to JTAG Mode.

Power cycle the board.

Step 3: Detect the FTDI Chip in Vivado Tcl Shell

Do not use the standard Windows CMD. You must use the Vivado environment.

Open the Start Menu and launch Vivado Tcl Shell.

Run the following command to scan for the raw FTDI chip (the exec keyword is required to run OS commands):

exec program_ftdi -read


You should see an output like this. Note down your specific Serial number (e.g., FTBL98PJ):

INFO: Detected 1 devices
Device location = 466
ftdi part = FT2232H
Serial = FTBL98PJ
vendor =
board =
manufacturer = Digilent
Board Description = Digilent USB Device


Step 4: Flash the Xilinx Signature to EEPROM

Now, use the serial number you just found to program the proprietary JTAG signature.

In the same Vivado Tcl Shell, run the following command (replace <Your_Serial> with your actual serial number):

exec program_ftdi -write -ftdi FT2232H -serial <Your_Serial> -vendor "Custom" -board "Zynq7020" -desc "JTAG Cable"


If successful, the tool will flash the EEPROM instantly and output:

INFO: ftdi part = FT2232H
INFO: Serial = FTBL98PJ
INFO: Detected 1 devices
INFO: Device location = 466
INFO: EEPROM(SPROM)
INFO: FTDI Programming Passed


Step 5: Power Cycle and Connect to Vivado

Close the Tcl Shell.

Unplug the USB cable, wait 5 seconds, and plug it back in. Windows will now detect the new hardware signature and automatically load the correct JTAG drivers.

Open Vivado.

Go to Open Hardware Manager -> Open Target -> Auto Connect.

🎉 Result: Vivado will instantly recognize the board, displaying the ARM core (arm_dap) and the FPGA fabric (xc7z020) in the hardware tree. Channel B remains untouched and is fully available as a standard COM port for your serial terminal.

Documented after a successful deep-dive troubleshooting session. Feel free to contribute or open issues if your board behaves differently!
