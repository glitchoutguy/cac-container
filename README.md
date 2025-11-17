# cac-container
### 🦊 Distrobox Firefox Setup for CAC/Smart Card Use

This guide will walk you through setting up a self-contained environment for using Firefox with a smart card or Common Access Card (CAC) reader, utilizing Distrobox on a Linux host.

## 1. Introduction

This guide focuses on Distrobox, a utility that leverages Podman or Docker to create development and application environments that integrate closely with the host system.

Note: All examples and commands here are run using Distrobox. However, the underlying concepts (e.g., mounting USB devices, installing pcsc-lite, managing services) are generally applicable to other container applications like Docker or Podman.

## 2. Prerequisites and Host System Check

Before creating the container, ensure your smart card reader is connected and recognized by your Linux host machine.

### ✅ Prerequisite Check (Run on Host)

Check if the card reader is connected and seen by the host:

    lsusb

Look for an entry that corresponds to your card reader (e.g., "ACS" or "Gemalto").

Check if the device nodes exist:

    ls -l /dev/bus/usb/

Verify that the directory structure where the USB devices are mapped exists.

## 3. Container Creation

Use the following command to create the Distrobox container. This command specifies a recent Fedora image and includes necessary flags for device integration and service management.

    distrobox create -r --init -n sc-firefox -i registry.fedoraproject.org/fedora:42 --additional-packages "systemd dbus dbus-daemon" --volume /dev/bus/usb:/dev/bus/usb:rw

⚙️ Explanation of Flags

| Flag | Description | Why is it Used? |
| :--- | :--- | :--- |
| **`-r`** | Rootful Mode | Required for system-level functions like running `systemd` and managing system services (`pcscd`). |
| **`--init`** | Run Init System | Allows you to use `systemctl` inside the container for enabling and starting the `pcscd` background service. |
| **`-n sc-firefox`** | Name | Assigns the name **`sc-firefox`** for easy reference. |
| **`-i ...`** | Image | Specifies the base operating system image (e.g., **Fedora 42**). |
| **`--additional-packages`** | Install Packages | Installs core packages (`systemd`, `dbus`, `dbus-daemon`) needed to run and manage system services. |
| **`--volume /dev/bus/usb:/dev/bus/usb:rw`** | Mount USB | **Critical:** This mounts the host's USB device directory into the container, giving the container read/write access to the physical card reader. |

## 4. Entering the Container

Once created, enter the container using its assigned name:

    distrobox enter -r sc-firefox

Note: All subsequent commands are run inside the Distrobox container unless explicitly stated otherwise.

## 5. Installing Software

Update the container's package list and install Firefox along with the smart card utilities.

    sudo dnf update -y
    sudo dnf install -y firefox pcsc-lite pcsc-lite-ccid pcsc-tools

Package	Purpose

| Package Name | Purpose |
| :--- | :--- |
| **`firefox`** | The web browser to be used with the CAC/smart card. |
| **`pcsc-lite`** | The **daemon** that manages communication between the physical smart card reader and the applications (like Firefox). |
| **`pcsc-lite-ccid`** | The **driver bundle** providing support for CCID-compliant smart card readers. |
| **`pcsc-tools`** | Provides utilities, such as **`pcsc_scan`**, used for testing the smart card reader connection and functionality. |

## 6. Enabling and Checking Services

The pcscd daemon must be running inside the container to communicate with the smart card reader.

Enable and Start the pcscd service:

    sudo systemctl enable pcscd.socket
    sudo systemctl start pcscd.socket
    sudo systemctl start pcscd

Check the service status:

    systemctl status pcscd

The output should show the service as active (running).

## 7. Checking Smart Card Detection

Use the pcsc_scan utility to confirm the container can successfully detect the reader and the smart card.

    pcsc_scan

Success: The output will list the reader and, if a card is inserted, show Card state: Card inserted along with its ATR value.

Failure: If it fails, proceed to the Troubleshooting section.

## 8. Configuring Firefox

Open Firefox

    firefox &

Navigate to Settings > Find in settings search "security"

Click the "Security Devices" option which will open the Device Manager window.

On the right, click the "Load" button.

Name your module something like "P11 kit proxy"

Enter the path to your security device driver:
#Example default path in Fedora

    /usr/lib64/p11-kit-proxy.so

Download the DoD Certs from: https://www.cyber.mil/pki-pke/document-library

DoD Approved External PKI Certificate Trust Chains - Version ##.# (Download the latest ones)

Unzip the file and locate the _DoD folder, you do not need any of the other ones

Install all certificates in this folder (.cer files) by searching "Cert" in the settings search bar and clicking "View Certificates"

You will then need to import each certificate one-by-one, checking both permission boxes in the window. (There is certainly a way to automate this I just havent figured it out)

## 9. Troubleshooting 🛠️

Host System Conflict (Device Lock)

If the container cannot access the reader, another process on the host might be locking the USB device node.

### Identify the process holding the device (Run on HOST):

First, use lsusb to find your device's Bus and Device numbers (e.g., Bus 005 Device 002).

Run fuser on the host, substituting your numbers:

    sudo fuser /dev/bus/usb/005/002

This will output the PID (Process ID) of the locking process.

Kill the offending process (Run on HOST):

    sudo kill -9 <PID_Number>

### Disabling Host pcscd

If you are using the reader only inside the Distrobox container, disable the host's smart card service to prevent conflicts.

# Run this on the HOST MACHINE
    sudo systemctl stop pcscd
    sudo systemctl disable pcscd

### Disabling Polkit

After confirming pcsc_scan is working, launch Firefox.

# Run this INSIDE the Distrobox container
    sudo /usr/sbin/pcscd --disable-polkit

--disable-polkit: This flag is often necessary when launching applications from a container to prevent issues with PolicyKit authentication requests that cannot easily be resolved across the container boundary.
