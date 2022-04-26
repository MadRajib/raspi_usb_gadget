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
