### Synology NAS

As Synology within DSM now supports Docker (with a neat UI), you can simply install Home Assistant using Docker without the need for command-line. 

<div class='note'>
  For details about the package see [Synology Add-on Packages](https://www.synology.com/en-us/dsm/packages?search=container)

  The package has been rebranded for DSM7
  
  * DSM6 Docker 
  * DSM7 Container Manager
  
</div>

The steps would be:

1. Install Docker "Container Manager" package on your Synology NAS
1. Launch Container Manager-app and move to "Registry"-section
1. Find "homeassistant/home-assistant" within registry and click on "Download". Choose the "stable" tag.
  -  Wait for some time until your NAS has pulled the image
1. Move to the "Image"-section of the "Container Manager"-app
  - Click on "Launch"
  - Within "Network" select "Use same network as Docker Host" and click Next
  - Choose a container name you want (e.g., "homeassistant")
- Set "Enable auto-restart" if you like
- Click on "Advanced Settings". To ensure that Home Assistant displays the correct timezone go to the "Environment" tab and click the plus sign then add `variable` = `TZ` & `value` = `Europe/London` choosing [your correct timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). Click Save to exit Advanced Settings.
- Click Next
- Within "Volume Settings" click on "Add Folder" and choose either an existing folder or add a new folder (e.g. in "docker" shared folder, add new folder named "homeassistant" and then within that new folder add another new folder "config"), then click Select. Then edit the "mount path" to be "/config". This configures where Home Assistant will store configs and logs.
- Ensure "Run this container after the wizard is finished" is checked and click Done
- Your Home Assistant within Docker should now run and will serve the web interface from port 8123 on your Docker host (this will be your Synology NAS IP address - for example `http://192.168.1.10:8123`)

If you are using the built-in firewall, you must also add the port 8123 to allowed list. This can be found in "Control Panel -> Security" and then the Firewall tab. Click "Edit Rules" besides the Firewall Profile dropdown box. Create a new rule and select "Custom" for Ports and add 8123. Edit Source IP if you like or leave it at default "All". Action should stay at "Allow".

#### Z-Wave Control

To use a Z-Wave USB stick for Z-Wave control, the HA Docker container needs extra configuration to access to the USB stick. Since the Synology package does not allow mapping host devices in the UI, you will need another way to start the container. While there are multiple ways to do this, the least privileged way of granting access can only be performed via the Terminal, at the time of writing.

<div class='note'>

[(Synology) Configuration settings Terminal access on your Synology NAS](https://www.synology.com/en-global/knowledgebase/DSM/help/DSM/AdminCenter/system_terminal)
[(Synology) How can I sign in to DSM/SRM with root privilege via SSH?](https://www.synology.com/en-global/knowledgebase/DSM/tutorial/General_Setup/How_to_login_to_DSM_with_root_permission_via_SSH_Telnet)

</div>

Since you are starting a container via the command line, you also need to take care of the settings you otherwise configure in the UI.

Adjust the Terminal command as follows:

- Replace `CONTAINER_NAME` with the name you want (e.g., "homeassistant")
- Replace `/PATH_TO_YOUR_CONFIG` with the folder where you want to store your configuration -  make sure that you keep the `:/config` part
- Replace `/PATH_TO_YOUR_USB_STICK` matches the path for your USB stick (e.g., `/dev/ttyACM0` for most Synology users)
- Replace "Australia/Melbourne" with [your timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

Run it in Terminal.

```bash
sudo docker run --restart always -d --name CONTAINER_NAME -v /PATH_TO_YOUR_CONFIG:/config --device=/PATH_TO_YOUR_USB_STICK -e TZ=Australia/Melbourne --net=host {{ site.installation.container }}:stable
```

Example
```bash
sudo docker run --restart always -d --name home-assistant -v /volume1/docker/config:/config --device=/dev/ttyACM0 -e TZ=Australia/Melbourne --net=host {{ site.installation.container }}:stable
```

<div class='note'>
If this does not allow you to use a USB Bluetooth adapter or Z-Wave USB Stick with Home Assistant on Synology Docker, a more [in-depth guide](https://philhawthorne.com/installing-home-assistant-io-on-a-synology-diskstation-nas/) is written by Phil Hawthorne.
</div>

Complete the remainder of the Z-Wave configuration by [following the instructions here.](/integrations/zwave_js)

##### Additional steps for DSM7+

Since DSM7, Synology has removed support for specific USB devices.
The kernel is now missing the driver needed for the Z-Wave stick. This means you will not be able to find the correct device name (e.g. `/dev/ttyACM0`).

Symptom to look for 
- The output of command `lsusb` looks something like this, showing the USB device is correctly identified. Here the Z-Wave stick is connected to usb1/1-1
```console
admin@synology:~$ lsusb
|__usb1          1d6b:0002:0404 09  2.00  480MBit/s 0mA 1IF  (Linux 4.4.302+ xhci-hcd xHCI Host Controller 0000:00:15.0) hub
  |__1-1         10c4:ea60:0100 00  2.00   12MBit/s 100mA 1IF  (Nabu Casa SkyConnect v1.0 fc06c840be96ed11b477be98a7669f5d)
  |__1-4         f400:f400:0100 00  2.00  480MBit/s 200mA 1IF  (Synology DiskStation 65003349BA326570)
|__usb2          1d6b:0003:0404 09  3.00 5000MBit/s 0mA 1IF  (Linux 4.4.302+ xhci-hcd xHCI Host Controller 0000:00:15.0) hub
```  
- The command `ll /sys/class/tty | grep 'usb'` does not show any output.

<div class='note'>
These steps are based off the [following community topic](https://community.home-assistant.io/t/how-set-up-z-wave-usb-stick-with-ha-on-synology-nas-docker-dsm-7-1/478035/9)
</div>

[The `dsm7-usb-serial-drivers` repository](https://github.com/robertklep/dsm7-usb-serial-drivers) lists the missing USB device drivers.

1. Identify your current DSM version and [Synology architecture](https://kb.synology.com/en-uk/DSM/tutorial/What_kind_of_CPU_does_my_NAS_have)
2. Follow the [installation instructions](https://github.com/robertklep/dsm7-usb-serial-drivers/tree/main#installation) and run the `usb-serial-drivers.sh` script
3. Plug in your USB stick

The output of `dmesg` should look something like this, showing the USB driver correctly attached the USB device to a `tty*`
```console
admin@synology:~$ dmesg
[564412.094606] usb 1-1: new full-speed USB device number 7 using xhci_hcd
[564412.239740] cp210x 1-1:1.0: cp210x converter detected
[564412.245657] usb 1-1: cp210x converter now attached to ttyUSB0
```
The command `ll /sys/class/tty | grep 'usb'` should also show you the mapped tty device.
```console
admin@synology:~$ ll /sys/class/tty | grep 'usb'
lrwxrwxrwx  1 root root 0 Nov 25 13:03 ttyUSB0 -> ../../devices/pci0000:00/0000:00:15.0/usb1/1-1/1-1:1.0/ttyUSB0/tty/ttyUSB0
```

#### Updating the Home assistant image 

To update your Home Assistant on your Docker within Synology NAS, you just have to do the following:

1. Go to the "Container Manager"-app and move to "Image"-section
1. Find "homeassistant/home-assistant" within Image and click on "Update".
1. Wait until the system-message/-notification comes up, that the download is finished (there is no progress bar)
1. Move to "Container"-section
1. Stop your container if it's running
1. Right-click on it and select "Action"->"Reset". You won't lose any data, as all files are stored in your configuration-directory
1.  Start the container again - it will then boot up with the new Home Assistant image

#### Restarting the Home Assistant within Synology NAS
To restart your Home Assistant within Synology NAS, you just have to do the following:

1. Go to the "Container Manager"-app and move to "Container"-section
1. Right-click on it and select "Action"->"Restart".


### QNAP NAS

As QNAP within QTS now supports Docker (with a neat UI), you can simply install Home Assistant using Docker without the need for command-line. For details about the package (including compatibility-information, if your NAS is supported), see <https://www.qnap.com/solution/container_station/en/index.php>

The steps would be:

- Install "Container Station" package on your Qnap NAS
- Launch Container Station and move to "Create Container"-section
- Search image "homeassistant/home-assistant" with Docker Hub and click on "Install"
  Make attention to CPU architecture of your NAS. For ARM CPU types the correct image is "homeassistant/armhf-homeassistant"
- Choose "stable" version and click next
- Choose a container-name you want (e.g., "homeassistant")
- Click on "Advanced Settings"
- Within "Shared Folders" click on "Volume from host" > "Add" and choose either an existing folder or add a new folder. The "mount point has to be `/config`, so that Home Assistant will use it for the configuration and logs.
- Within "Network" and select Network Mode to "Host"
- To ensure that Home Assistant displays the correct timezone go to the "Environment" tab and click the plus sign then add `variable` = `TZ` & `value` = `Europe/London` choosing [your correct timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
- Click on "Create"
- Wait for some time until your NAS has created the container
- Your Home Assistant within Docker should now run and will serve the web interface from port 8123 on your Docker host (this will be your Qnap NAS IP address - for example `http://192.xxx.xxx.xxx:8123`)

Remark: To update your Home Assistant on your Docker within Qnap NAS, you just remove container and image and do steps again (Don't remove "config" folder).
