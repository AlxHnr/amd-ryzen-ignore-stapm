This repository contains a service for bypassing AMD STAPM (Skin Temperature-Aware Power
Management). It was written with Framework 13 Ryzen 7040 laptops in mind, which have two major
problems with their power management:

* They reduce APU wattage dynamically to maintain a certain chassis temperature, but that logic
  reacts too slowly to prevent soft-throttling
* Once soft-throttling kicks in, it stays enabled until you pause all workloads and let the laptop
  cool off completely

The service in this repository keeps chassis-based throttling disabled. This makes it possible for
the APU to run at whatever wattage the current airflow permits. It does not disable hard throttling
(PROCHOT).

If you switch away from your systems "performance" power profile, the service will pause and do
absolutely nothing until you switch back to "performance". Power profiles can be set by either
running `powerprofilesctl set performance`, or via GNOME shell:

![screenshot](https://github.com/user-attachments/assets/d7382465-a156-4d30-aebd-bf5280214d77)

The APU's wattage will self-regulate to maintain a stable <=100°C under full load. If k10temp is
hitting 101°C repeatedly, your setup lacks the airflow to max out your machine. Ensure that the air
vents are far enough away from the table. If you only use external monitors and the laptop is always
closed, flip it upside down to expose the air vents upwards. If you can't, or don't want to, you can
lower the power limits
[here](https://github.com/AlxHnr/amd-ryzen-ignore-stapm/blob/f9ffee17f6e2f84ab763795b3462b8aa9498703e/amd-ryzen-ignore-stapm.sh#L64-L66).

**Note**: The chassis will warm up under full load around the WASD keys. It may not feel very hot,
but after a few hours your fingertips will feel sore. Use an external keyboard for gaming.

## Setup (Fedora)

```sh
sudo dnf install bc dkms gcc openssl power-profiles-daemon xxd
git clone https://github.com/amkillam/ryzen_smu # Kernel module for changing power limits
(cd ryzen_smu/ && sudo make dkms-install)
rm -vrf ryzen_smu/
```

The installed module will be recompiled automatically with every kernel update.

### Custom UEFI keys

Secure boot is almost certainly enabled on your system. This means it only loads custom kernel
modules when they are signed with your own UEFI keys. The previous setup step has already done this
for us, including the creation of a key. This key must be added to your machines UEFI key database,
which can be done with the following command. The command will ask you to set a custom password. The
password is not very important and can be reset as often as you want. It is used _only one single
time_ later in this guide, and then never again.

```sh
sudo mokutil --import /var/lib/dkms/mok.pub
```

Restart your system. This will boot into the MOK manager. Choose `Enroll MOK`, enter your custom
password and then reboot.
[Here](https://github.com/dell/dkms/blob/f7f526c145ecc01fb4ac4eab3009b1879b14ced4/README.md#secure-boot)
are some screenshots describing the process. After rebooting, the module will be loaded and shows up
in dmesg. It will print a message about the kernel being tainted, but this just means it loaded an
unknown binary blob.

### Systemd service

```sh
sudo cp amd-ryzen-ignore-stapm.sh /usr/local/bin/
sudo cp amd-ryzen-ignore-stapm.service /etc/systemd/system/
sudo systemctl enable --now amd-ryzen-ignore-stapm
```

## Files to backup

```
/var/lib/dkms/mok.*
/usr/src/ryzen_smu*/
/usr/local/bin/amd-ryzen-ignore-stapm.sh
/etc/systemd/system/amd-ryzen-ignore-stapm.service
```

If you reinstall your system, run these commands after restoring the backed-up files:

```sh
sudo dkms add ryzen_smu/0.1.7
sudo dkms build ryzen_smu/0.1.7
sudo dkms install ryzen_smu/0.1.7
sudo systemctl enable --now amd-ryzen-ignore-stapm
```

## Uninstalling

```sh
sudo dkms remove ryzen_smu/0.1.7 --all
sudo rm -vrf /usr/src/ryzen_smu*

# Enter a single-use MOK manager password of your choice:
sudo mokutil --delete /var/lib/dkms/mok.pub
reboot # This will boot into the MOK manager where you can delete the key

sudo rm -vrf /var/lib/dkms/
sudo systemctl disable --now amd-ryzen-ignore-stapm
sudo rm /etc/systemd/system/amd-ryzen-ignore-stapm.service
sudo rm /usr/local/bin/amd-ryzen-ignore-stapm.sh
sudo dnf remove bc dkms gcc openssl power-profiles-daemon xxd
```
