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
