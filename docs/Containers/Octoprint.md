# OctoPrint – the snappy web interface for your 3D printer

## References

* [OctoPrint home page](https://octoprint.org)
* [OctoPrint Community Forum](https://community.octoprint.org)
* DockerHub [octoprint/octoprint](https://hub.docker.com/r/octoprint/octoprint)
* GitHub [OctoPrint/octoprint-docker](https://github.com/OctoPrint/octoprint-docker)

## Device mappings

When you select "OctoPrint" in the IOTstack menu, the service definition in your `docker-compose.yml`, contains the following under the `devices:` heading:

```
devices:
  - /dev/ttyAMA0:/dev/ttyACM0
# - /dev/video0:/dev/video0
```

### *the `/dev/ttyAMA0:/dev/ttyACM0` mapping*

The `/dev/ttyAMA0:/dev/ttyACM0` mapping should be read as saying "the physical Raspberry Pi device `/dev/ttyAMA0` is mapped to the logical OctoPrint container device `/dev/ttyACM0`".

The `/dev/ttyAMA0` device is used as a default because it is always present on Raspbian. If you bring up your container like that, the mapping will succeed and the container is unlikely to go into a restart loop.

However, the OctoPrint container is unlikely to be able to connect to your 3D printer via `/dev/ttyAMA0` for the very simple reason that that is not how 3D printers usually appear on the Raspberry Pi. You need to work out *how* your printer presents itself and change the device mapping accordingly.

#### option 1 - `/dev/ttyUSBn`

Using "ttyUSBn" will "work" but, because of the inherent variability in the name, this approach is not recommended.
 
The "n" in the "ttyUSBn" can vary depending on which USB devices are attached to your Raspberry Pi and the order in which they are attached. The "n" may also change as you add and remove devices.

If the OctoPrint container is up when the device number changes, the container will crash, and it will either go into a restart loop if you try to bring it up when the expected device is not "there", or will try to communicate with a device that isn't your 3D printer.

#### option 2 - `/dev/serial/by-id/xxxxxxxx`

The "xxxxxxxx" is (usually) unique to your 3D printer. To find it, connect your printer to your Raspberry Pi, then run the command:

```
$ ls -1 /dev/serial/by-id
```

You will get an answer like this:

```
usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_3b14eaa48a154d5e87032d59459d5206-if00-port0
```

Note:

* If you have multiple serial devices attached, you will get multiple lines in the output. It is up to you to sort out which one belongs to your 3D printer, possibly by disconnecting and re-attaching the printer and observing how the list changes.
* The uniqueness of device IDs is under the control of the device manufacturer. Each manufacturer *should* ensure their devices are unique but some manufacturers are more diligent than others.

Assuming the above example output was the answer, edit `docker-compose.yml` to look like this:

```
devices:
  - /dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_3b14eaa48a154d5e87032d59459d5206-if00-port0:/dev/ttyACM0
```

Notes:

* device *by-id* names follow the device. In other words, if you have two or more Raspberry Pis and a collection of serial devices (3D printers, Zigbee adapters, UARTs, and so on), a 3D printer will always get the same by-id name, irrespective of which Raspberry Pi it is attached to.
* device *by-id* names do not persist if the physical device is disconnected. If you switch off your 3D printer or disconnect the USB cable while the OctoPrint container is running, the container will crash.

#### option 3 - `/dev/humanReadableName`

Suppose your 3D printer is a MasterDisaster5000Pro, and that you would like to be able to set up the device to use a human-readable name like:

```
/dev/MasterDisaster5000Pro
```

Start by disconnecting your 3D printer from your Raspberry Pi. Next, run this command:

```
$ tail -f /var/log/messages
```

Connect your 3D printer and observe the log output. You are interested in messages that look like this:

```
mmm dd hh:mm:ss mypi kernel: [423839.626522] cp210x 1-1.1.3:1.0: device disconnected
mmm dd hh:mm:ss mypi kernel: [431265.973308] usb 1-1.1.3: new full-speed USB device number 10 using dwc_otg
mmm dd hh:mm:ss mypi kernel: [431266.109418] usb 1-1.1.3: New USB device found, idVendor=dead, idProduct=beef, bcdDevice= 1.00
mmm dd hh:mm:ss mypi kernel: [431266.109439] usb 1-1.1.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
mmm dd hh:mm:ss mypi kernel: [431266.109456] usb 1-1.1.3: Product: CP2102N USB to UART Bridge Controller
mmm dd hh:mm:ss mypi kernel: [431266.109471] usb 1-1.1.3: Manufacturer: Silicon Labs
mmm dd hh:mm:ss mypi kernel: [431266.109486] usb 1-1.1.3: SerialNumber: cafe80facefeed
mmm dd hh:mm:ss mypi kernel: [431266.110657] cp210x 1-1.1.3:1.0: cp210x converter detected
mmm dd hh:mm:ss mypi kernel: [431266.119225] usb 1-1.1.3: cp210x converter now attached to ttyUSB0
```

and, in particular, these two lines:

```
… New USB device found, idVendor=dead, idProduct=beef, bcdDevice= 1.00
… SerialNumber: cafe80facefeed
```

Terminate the `tail` command by pressing Control+C.

Use this line as a template:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="«idVendor»", ATTRS{idProduct}=="«idProduct»", ATTRS{serial}=="«SerialNumber»", SYMLINK+="«sensibleName»"
```

Replace the «delimited» values with those you see in the log output. For example, given the above log output, and the desire to associate your 3D printer with the human-readable name of "MasterDisaster5000Pro", the result would be:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="dead", ATTRS{idProduct}=="beef", ATTRS{serial}=="cafe80facefeed", SYMLINK+="MasterDisaster5000Pro"
```

Next, ensure the required file exists by executing the following command:

```
$ sudo touch /etc/udev/rules.d/99-usb-serial.rules
```

> If the file does not exist already, the `touch` command creates an empty file, owned by root, with mode 644 (rw-r--r--) permissions (all of which are correct).

Use `sudo` and your favourite text editor to edit `/etc/udev/rules.d/99-usb-serial.rules` and insert the "SUBSYSTEM==" line you prepared earlier into that file, then save the file.

> Rules files are read on demand so there is no `start` or `reload` command to execute.

Check your work by disconnecting, then re-connecting your 3D printer, and then run:

```
$ ls /dev
``` 

You should expect to see the human-readable name you chose in the list of devices. You can then edit `docker-compose.yml` to use the name in the device mapping.

```
devices:
  - /dev/MasterDisaster5000Pro:/dev/ttyACM0
```

Notes:

* device names follow the device. In other words, if you have two or more Raspberry Pis and a collection of serial devices (3D printers, Zigbee adapters, UARTs, and so on), you can build a single `99-usb-serial.rules` file that you install on *all* of your Raspberry Pis. Then, you can attach a named device to any of your Raspberry Pis and it will always get the same name.
* device names do not persist if the physical device is disconnected. If you switch off your 3D printer or disconnect the USB cable while the OctoPrint container is running, the container will crash.

### *the `/dev/video0:/dev/video0` mapping*

The `/dev/video0` device is assumed to be an official Raspberry Pi camera attached via ribbon cable.

> See the [Webcams topic of the Octoprint Community Forum](https://community.octoprint.org/c/support/support-webcams/18) for help configuring other kinds of cameras.

The OctoPrint docker image includes an MJPG streamer. You do not need to run another container with a streamer unless you want to.

To activate a Raspberry Pi camera attached via ribbon cable:

1. Follow the instructions at [raspberrypi.org](https://www.raspberrypi.org/documentation/configuration/camera.md) to connect and test the camera. There are guides on YouTube ([example](https://www.youtube.com/watch?v=T8T6S5eFpqE)) if you need help working out how to insert the ribbon cable.
2. Confirm the presence of `/dev/video0`.
3. Edit `docker-compose.yml` and uncomment **all** of the commented-out lines in the following:

	```
	devices:
	# - /dev/video0:/dev/video0
	environment:
	# - ENABLE_MJPG_STREAMER=true
	# - MJPG_STREAMER_INPUT=-r 640x480 -f 10 -y
	# - CAMERA_DEV=/dev/video0
	```

	Note:
	
	* The device path on the right hand side of the `CAMERA_DEV` environment variable corresponds with the right hand side (ie *after* the colon) of the device mapping. There should be no reason to change either.

Use the following values to configure the camera in the OctoPrint web interface:

* Stream URL: /webcam/?action=stream
* Snapshot URL: http://localhost:8080/?action=snapshot
* Path to FFMPEG: /usr/bin/ffmpeg

> For those who normally hear alarm bells when they see "localhost" in a Docker context, rest assured that this is correct. The MJPG streamer runs inside the *same* container as OctoPrint so "localhost" is appropriate.

The three environment variables are required:

```
environment:
  - ENABLE_MJPG_STREAMER=true
  - MJPG_STREAMER_INPUT=-r 640x480 -f 10 -y
  - CAMERA_DEV=/dev/video0
```

The "640x480" `MJPG_STREAMER_INPUT` settings will probably result in your camera feed being "letterboxed" but they will get you started. A full list of options is at [mjpg-streamer-configuration-options](https://community.octoprint.org/t/available-mjpg-streamer-configuration-options/1106).

The typical specs for a baseline Raspberry Pi camera are:

* 1080p 720p 5Mp Webcam
* Max resolution: 2592x1944
* Max frame rate: VGA 90fps, 1080p 30fps
* CODEC: MJPG H.264 AVC

For that type of camera, the following is probably more appropriate:

```
  - MJPG_STREAMER_INPUT=-r 1152x648 -f 10
```

The resolution of 1152x648 is 60% of 1080p 1920x1080 and does not cause letterboxing. The resolution and rate of 10 frames per second won't over-tax your communications links, and the camera is MJPEG-capable so it does not need the `-y` option.

## Practical usage

### starting the OctoPrint container

To start a print session:

1. Turn the 3D printer on.
2. Bring up the container:

	```
	$ cd ~/IOTstack
	$ docker-compose up -d octoprint
	```

If you try to start the OctoPrint container before your 3D printer has been switched on and the USB interface has registered with the Raspberry Pi, the container will go into a restart loop.

### stopping the OctoPrint container

Unless you intend to leave your printer switched on 24 hours a day, you will also need to be careful when you switch off the printer:

1. Terminate the container:

	```
	$ cd ~/IOTstack
	$ docker-compose stop octoprint
	$ docker-compose rm -f octoprint
	```

2. Turn the 3D printer off.

If you turn the printer off without terminating the container, you will crash the container.

### Connecting to OctoPrint

Use a browser to point to port 9980 on your Raspberry Pi. For example:

```
http://raspberrypi.local:9980
```

The first time you do this, you will need to create an administrative account.

### Video feed (built-in camera interface)

You can view the video feed independently of the OctoPrint web interface like this:

```
http://raspberrypi.local:9981/?action=stream
```

### Silencing the security warning

OctoPrint assumes it is running "natively" rather than in a container. From a data-communications perspective, OctoPrint (the process running inside the OctoPrint container) sees itself as running on a computer attached to the internal Docker network. When you connect to OctoPrint's web interface from a client device attached to an external network, OctoPrint sees that your source IP address is not on the internal Docker network and it issues a security warning.

To silence the warning:

1. Terminate the container if it is running:

	```
	$ cd ~/IOTstack
	$ docker-compose stop octoprint
	$ docker-compose rm -f octoprint
	```
	
2. use `sudo` and your favourite text editor to open the following file:

	```
	~/IOTstack/volumes/octoprint/octoprint/config.yaml
	```

3. Implement the following pattern:

	```
	server:
	    …
	    ipCheck:
	        enabled: true
	        trustedSubnets:
	        - 192.168.1.0/24
	```

	Notes:
	
	* It is likely that the `server:`, `ipCheck:` and `enabled:` directives are already in place but the `trustedSubnets:` directive may not be. Add it, and then add your local subnet(s) where you see the "192.168.1.0/24" example.
	* Remember to use spaces in YAML files. Do not use tabs.

4. Save the file.
5. Bring up the container: 

	```
	$ cd ~/IOTstack
	$ docker-compose up -d octoprint
	```

### routine container maintenance

You can check for updates like this:

```
$ cd ~/IOTstack
$ docker-compose pull octoprint
$ docker-compose up -d octoprint
$ docker system prune
```

### if all else fails…

If you forget your administrative password or the OctoPrint container seems to be misbehaving, you can get a "clean slate" by:

```
$ cd ~/IOTstack
$ docker-compose stop octoprint
$ docker-compose rm -f octoprint
$ sudo rm -rf ./volumes/octoprint
$ docker-compose up -d octoprint
```