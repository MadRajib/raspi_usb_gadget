# raspi_usb_gadget

## Enable the USB driver in raspbery pi zero 
```bash
sudo nano /boot/config.txt

# Scroll to the bottom and append the line below:

dtoverlay=dwc2

sudo nano /etc/modules
# Append the line below :
dwc2
```
Reboot the raspbery pi
## Follow these 
```bash
MANUFACTURER=“Some Company” # manufacturer attribute
SERIAL=“Frosted Flakes” # device serial number
IDPRODUCT=“0xa4ac” # hex product ID, issued by USB Group
IDVENDOR=“0x0525” # hex vendor ID, assigned by USB Group
PRODUCT=“Emulated HID Keyboard” # cleartext product description
CONFIG_NAME=“Configuration 1” # name of this configuration
MAX_POWER_MA=120 # max power this configuration can consume in mA
PROTOCOL=1 # 1 for keyboard. see usb spec
SUBCLASS=1 # it seems either 1 or 0 works, dunno why
REPORT_LENGTH=8 # number of bytes per report
DESCRIPTOR=/config/usb_gadget/keyboardgadget/kybd-descriptor.bin # binary blob of report descriptor, see HID class spec
UDC=ci_hdrc.0 # name of the UDC driver to use (found in /sys/class/udc/)

// not required if built-in - modprobe libcomposite # load configfs
// configfs already mounted on /sys/kernel/config - mount none /sys/kernel/config -t configfs
mkdir /sys/kernel/config/usb_gadget/keyboardgadget
cd /config/usb_gadget/keyboardgadget # cd to gadget dir
mkdir configs/c.1 # make the skeleton for a config for this gadget
mkdir functions/hid.usb0 # add hid function (will fail if kernel < 3.19, which hid was added in)
// cannot write when configfs build as module
// can write when configfs build in
echo $PROTOCOL > functions/hid.usb0/protocol # set the HID protocol
echo $SUBCLASS > functions/hid.usb0/subclass # set the device subclass
echo $REPORT_LENGTH > functions/hid.usb0/report_length # set the byte length of HID reports
cat $DESCRIPTOR > functions/hid.usb0/report_desc # write the binary blob of the report descriptor to report_desc; see HID class spec
mkdir strings/0x409 # setup standard device attribute strings
mkdir configs/c.1/strings/0x409
echo $IDPRODUCT > idProduct
echo $IDVENDOR > idVendor
echo $SERIAL > strings/0x409/serialnumber
echo $MANUFACTURER > strings/0x409/manufacturer
echo $PRODUCT > strings/0x409/product
echo $CONFIG_NAME > configs/c.1/strings/0x409/configuration
echo $MAX_POWER_MA > configs/c.1/MaxPower
ln -s functions/hid.usb0 configs/c.1 # put the function into the configuration by creating a symlink

CONNECT USB CABLE TO HOST

// binding
// echo $UDC > UDC
ls /sys/class/udc > UDC
```
## In device
```bash
mkdir /dev/ffs-diag
mount -t functionfs diag /dev/ffs-diag
```
Run app with the dev node
```bash
./chat_app /dev/ffs-diag 
```

## In Host side
To display all the usb devices connected
```bash
lsusb -v
```

## Setp by step in device

## The Usage Pattern:
1. The user creates a separate directory per each gadget they want to have.
```bash
# go to configfs directory for USB gadgets
CONFIGFS_ROOT=/sys/kernel/config # adapt to your machine
cd "${CONFIGFS_ROOT}"/usb_gadget

# create gadget directory and enter it
mkdir g1
cd g1
```
2. Gives their gadgets a personality by specifying vendor id, product id and USB strings (visible e.g after running lsusb -v as root).
```bash
# USB ids
echo 0x1d6b > idVendor
echo 0x104 > idProduct

# USB strings, optional
mkdir strings/0x409 # US English, others rarely seen
echo "Collabora" > strings/0x409/manufacturer
echo "ECM" > strings/0x409/product
```
3. Then under that directory creates the configurations they want and instantiates USB funcsts they want (Both by creating respective directories)
```bash
# create the (only) configuration
mkdir configs/c.1 # dot and number mandatory

# create the (only) function
mkdir functions/ecm.usb0 # .
```
4. And finally associates functiona to configurations with symbolic links.
```bash
# assign function to configuration
ln -s functions/ecm.usb0/ configs/c.1/
```
5. At this point gadget's composition is already in memory, but is not bound to any UDC.
6. To activate the gadget one must write UDC name to the UDC attribute in the gadget's configfs directory.
```bash
# bind!
echo 12480000.hsotg > UDC # ls /sys/class/udc to see available UDCs
```
7. The gadget then becomes bound to this particular UDC (and the UDC cannot be used by more than one gadget).
8. Available UDC names are in /sys/class/udc. 
9. Only after a gadget is bound to a UDC can it be successfully enumerated by the USB host. 
10. If the devices is connected to usb host, we can see the following logs in dmesg:
```bash
usb 3-1.2.1.4.4: New USB device found, idVendor=1d6b, idProduct=0104
usb 3-1.2.1.4.4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
usb 3-1.2.1.4.4: Product: ECM
usb 3-1.2.1.4.4: Manufacturer: Collabora
cdc_ether 3-1.2.1.4.4:1.0 usb0: register 'cdc_ether' at usb-0000:3c:00.0-1.2.1.4.4, CDC Ethernet Device, d2:c2:2d:b7:8e:6b
```
> Note the Product and Manufacturer strings which are exactly what has been written to configfs.
11. A new host should appear at the host side ..
```bash
ifconfig -a

usb0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether d2:c2:2d:b7:8e:6b  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1  bytes 90 (90.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# why not configure it?
ifconfig usb0 192.168.1.2 up
```
...and at the device
```bash
ifconfig -a

usb0: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether f2:40:e6:d3:01:2c  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# why not configure this one as well?
ifconfig usb0 192.168.1.3 up

# and ping the host?
ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=1.40 ms
```
Similarly, the device can be pinged from the host:
```bash
ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=1.06 ms
```