# Serial console

The serial console is the first thing to connect: it shows the bootloader, U-Boot, kernel log and a login shell. Settings for all RZ boards: **115200 baud, 8N1, no flow control**.

## Variables

```bash
export TTY=/dev/ttyUSB0
```

## Find the port

=== "Linux"

    Plug in the board's debug USB port, then:

    ```bash
    sudo dmesg | tail
    ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
    ```

    ```
    usb 1-2: FTDI USB Serial Device converter now attached to ttyUSB0
    ```

    Stable names that do not change when other adapters are plugged in:

    ```bash
    ls -l /dev/serial/by-id/
    ```

=== "Windows"

    *Device Manager → Ports (COM & LPT)* shows the port, e.g. `USB Serial Port (COM5)`. If it shows up with a warning icon, install the USB-serial driver (FTDI / CP210x, depending on the board).

## Permissions (Linux)

Add my user to the `dialout` group once, then log out and back in:

```bash
sudo usermod -aG dialout $USER
```

Without this, every terminal program needs `sudo`.

## Connect

=== "picocom"

    ```bash
    sudo apt-get install -y picocom
    picocom -b 115200 $TTY
    ```

    Exit: ++ctrl+a++ then ++ctrl+x++.

    Save everything to a log file as well:

    ```bash
    picocom -b 115200 --logfile ~/serial-$(date +%F-%H%M).log $TTY
    ```

=== "minicom"

    ```bash
    sudo apt-get install -y minicom
    minicom -D $TTY -b 115200
    ```

    Disable hardware flow control once: ++ctrl+a++ ++o++ → *Serial port setup* → `F` → *Save setup as dfl*.
    Exit: ++ctrl+a++ then ++x++. Log: ++ctrl+a++ then ++l++.

=== "screen"

    ```bash
    screen $TTY 115200
    ```

    Exit: ++ctrl+a++ then ++k++, confirm with ++y++.

=== "Tera Term (Windows)"

    1. *File → New connection → Serial*, select the COM port.
    2. *Setup → Serial port*: Speed `115200`, Data `8 bit`, Parity `none`, Stop `1 bit`, Flow control `none`.
    3. *Setup → Save setup* to keep the settings.

    Log: *File → Log...*. Send a file (Flash Writer, SREC): *File → Send file...*.

## Serial port in a VirtualBox VM

When the host OS is Windows and Linux runs in VirtualBox, pass the USB-serial adapter through to the VM:

1. Power off the VM. *Settings → USB*: enable the USB controller and add a filter for the adapter (the `+` icon lists connected devices).
2. Start the VM. The adapter disappears from Windows and appears as `/dev/ttyUSB0` in Linux.

!!! tip
    Only one side can use the port. If Tera Term on Windows is open, the VM cannot capture the device, and vice versa.

## Send a file from Linux

Flash Writer and the bootloader `.srec`/`.mot` files are plain text, so a raw copy to the port works (the same as Tera Term's *Send file*). Close the terminal program first, configure the port, then send:

```bash
stty -F $TTY 115200 cs8 -cstopb -parenb -crtscts raw
cat Flash_Writer_SCIF_RZG2L_SMARC_PMIC_DDR4_2GB_1PCS.mot > $TTY
```

Then reconnect with `picocom` to continue at the Flash Writer prompt. The full flashing procedure is in [Flashing the bootloader](../g2l/001boot.md).

## Useful on the board

Wider terminal after login (the serial console defaults to 80x24, so long lines wrap badly):

```bash
stty rows 50 cols 200
```

Kernel log level printed on the console (`1` = only emergencies, `7` = everything):

```bash
echo 4 > /proc/sys/kernel/printk
dmesg -n 4
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Garbage characters | Wrong baud rate. Use 115200 |
| No output at all | Wrong port (some boards expose two ports; try the other one), board not powered, or TX/RX swapped on a custom adapter |
| Output works but typing does nothing | Hardware flow control enabled. Disable it in the terminal settings |
| `Permission denied` on `/dev/ttyUSB0` | Not in the `dialout` group (log out/in after adding), or the port is used by another program (`sudo lsof $TTY`) |
| Port disappears in the VM | USB filter missing or Windows program still holds the port |
| Login prompt does not appear after kernel boot | `console=ttySC0,115200n8` missing in `bootargs`, or getty is on another tty |
