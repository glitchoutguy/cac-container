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
    sudo dnf install -y firefox pcsc-lite pcsc-lite-ccid pcsc-tools openssl

Package	Purpose

| Package Name | Purpose |
| :--- | :--- |
| **`firefox`** | The web browser to be used with the CAC/smart card. |
| **`pcsc-lite`** | The **daemon** that manages communication between the physical smart card reader and the applications (like Firefox). |
| **`pcsc-lite-ccid`** | The **driver bundle** providing support for CCID-compliant smart card readers. |
| **`pcsc-tools`** | Provides utilities, such as **`pcsc_scan`**, used for testing the smart card reader connection and functionality. |
| **`openssl`** | Tool used to check certificate validity. Used to verify downloaded certificates against DoD hosted certificates. |

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

### Security Device

Open Firefox

    firefox &

Navigate to Settings > Find in settings search "security"

Click the "Security Devices" option which will open the Device Manager window.

On the right, click the "Load" button.

Name your module something like "P11 kit proxy"

Enter the path to your security device driver:
#Example default path in Fedora

    /usr/lib64/p11-kit-proxy.so

### DoD Certificates

Download the DoD Certs from: https://www.cyber.mil/pki-pke/document-library

PKI CA Certificate Bundles: PKCS#7 for DoD PKI Only - Version #.## (Download the latest ones)

Unzip the file and within the certificates folder run the following command to determine the SHA1 Fingerprint of the Root CA:

    openssl x509 -in DoD_PKE_CA_chain.pem -subject -issuer -fingerprint -noout

Note the SHA1 Fingerprint value and check it against the values posted at: https://crl.gds.disa.mil/

Select the same CA identified from your command i.e.
    
    CN=DoD Root CA 6

Submit selection, click "View", and scroll to the bottom and verify the value of the SHA1 Fingerprint. **These values should be matching, do not use certificates that do not match**

Verify the S/MIME signatures in your .sha256 file with the following command:

    openssl smime -verify -in Certificates_PKCS7_v5_14_DoD.sha256 -inform DER -CAfile DoD_PKE_CA_chain.pem | dos2unix | sha256sum -c

There may be some errors, but as long as each .p7b file reads as succcessful then you may move on.

To import the certificates into firefox, run the following command:

    for n in *der.p7b; do certutil -d sql:$HOME/.pki/nssdb -A -t TC -n $n -i $n; done

You can verify the certificates are installed by restarting firefox then navigating to "Certificates" in settings. All of the certificates should be listed under "U.S. Government" 

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
