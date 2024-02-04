# tn-scale

Initial draft config steps for enabling Truecharts Frigate chart on a TrueNAS Scale Cobia system.

**NOTE:**
This is a total hack, no warranties, ymmv, etc. etc. etc.

Here goes:

1) FIRST CHECK IF APEX DRIVER IS ALREADY PRESENT:
If your kernel version is 4.19 or higher, now check if you have a pre-build Apex driver installed:
Run command
lsmod | grep apex

If it prints nothing, then you're okay and continue to install our PCIe driver.
This is likely if you are running TrueNAS Scale.

If it does print an Apex module name, stop here and follow the workaround to disable Apex and Gasket (https://coral.ai/docs/m2/get-started/#workaround-to-disable-apex-and-gasket)


2) INSTALLING PRE-REQS:
Run following script as root / run individual commands with sudo:

--
#!/bin/sh
chmod +x /usr/bin/apt*
chmod +x /usr/bin/dpkg*
chmod +x /usr/bin/dirmngr*
apt-get install gpg-agent
mkdir /etc/apt/keyrings

echo "deb [signed-by=/etc/apt/keyrings/coral.gpg] https://packages.cloud.google.com/apt coral-edgetpu-stable main" | tee /etc/apt/sources.list.d/coral-edgetpu.list

wget -O- https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor | sudo tee /etc/apt/keyrings/coral.gpg > /dev/null

apt-get update
apt-get install gasket-dkms libedgetpu1-std
sh -c "echo 'SUBSYSTEM==\"apex\", MODE=\"0660\", GROUP=\"apex\"' >> /etc/udev/rules.d/65-apex.rules"
groupadd apex
modprobe apex
udevadm control --reload-rules
--

3) REBOOT, then
CHECK FOR SUCCESS:

lspci -nn | grep 089a

You should see something like this:
03:00.0 System peripheral: Device 1ac1:089a


Also verify that the PCIe driver is loaded:
ls /dev/apex_0
You should simply see the name repeated back:
/dev/apex_0


4) CONFIGURE Truechart Frigate pod to map PCI device as USB device:

https://media.discordapp.net/attachments/1190320793775788163/1190332333585350696/image.png?ex=65cf8edf&is=65bd19df&hm=8e30fc5f37f1c353319e99e9d80cfb92a368a69b2d8e2ab3d54fae581674a946&=&width=1038&height=744

E.g.:

Host Device Path:
/dev/apex_0

mapped to:

Container Device Path:
/dev/ttyACM0


5) Configure Frigate as per its instructions, referencing the correct device, e.g.:

detectors:
  coral1:
    type: edgetpu
    device: pci:0

6) RINSE & REPEAT per each TN upgrade :)
   
