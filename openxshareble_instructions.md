Instructions for using Dexcom G4 Share receiver over bluetooth with OpenAPS 
I have only tested using an Edison running Ubilinux updated with this Jessie upgrade: https://github.com/oskarpearson/mmeowlink/wiki/Prepare-the-Edison-for-OpenAPS#initial-setup

```
cd ~/src
git clone -b wip/bewest/custom-gatt-profile https://github.com/bewest/Adafruit_Python_BluefruitLE.git
cd Adafruit_Python_BluefruitLE
sudo python setup.py develop
```

```
cd ~/src
git clone https://github.com/bewest/openxshareble.git
cd openxshareble
sudo python setup.py develop
```

`cd ~/src`

The steps below describe how to install BlueZ 5.37 on a Raspberry Pi running its Raspbian operating system. In a terminal on the Raspberry Pi run:

```
sudo apt-get update
sudo apt-get -y install libusb-dev libdbus-1-dev libglib2.0-dev libudev-dev libical-dev libreadline-dev
wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.37.tar.gz
tar xvfz bluez-5.37.tar.gz
cd bluez-5.37
./configure --enable-experimental --disable-systemd
```

Then:

```
make
sudo make install
sudo cp ./src/bluetoothd /usr/local/bin/
```

Finally you'll need to make sure the bluetoothd daemon runs at boot and is run with the --experimental flag to enable all the BLE APIs.

To do this edit the /etc/rc.local file:

`sudo nano /etc/rc.local`  (add the following line before the exit 0 at the end):

`/usr/local/bin/bluetoothd --experimental &`

Comment out the other bluetoothd line using a # in front of the line

Alternately/additionally, you may need to edit /etc/init.d/bluetooth if your system uses that (for example, running Ubilinux Jessie on Edison).

`sudo nano /etc/init.d/bluetooth`

Comment out the line that reads `DAEMON=/usr/sbin/bluetoothd` and replace it with:

`DAEMON=/usr/local/bin/bluetoothd`

Comment out the line that reads `#SSD_OPTIONS="--oknodo --quiet --exec $DAEMON -- $NOPLUGIN_OPTION"` and replace it with:

`SSD_OPTIONS="--oknodo --quiet --exec $DAEMON -- --experimental $NOPLUGIN_OPTION"`

Reboot and confirm the correct version of bluetoothd is running:

`ps aux | grep bluetoothd`

Output should contain `/usr/local/bin/bluetoothd --experimental`, not `/usr/sbin/bluetoothd`.

Next, you'll need to give permissions to users in the dialout group to run bluez stuff.

`sudo nano /etc/dbus-1/system.d/bluetooth.conf`

Add the following:
```
<!-- allow users of dialout group (chosen by bewest) to 
       communicate with bluetoothd -->
  <policy group="dialout">
    <allow own="org.bluez"/>
    <allow send_destination="org.bluez"/>
    <allow send_interface="org.bluez.Agent1"/>
    <allow send_interface="org.bluez.MediaEndpoint1"/>
    <allow send_interface="org.bluez.MediaPlayer1"/>
    <allow send_interface="org.bluez.ThermometerWatcher1"/>
    <allow send_interface="org.bluez.AlertAgent1"/>
    <allow send_interface="org.bluez.Profile1"/>
    <allow send_interface="org.bluez.HeartRateWatcher1"/>
    <allow send_interface="org.bluez.CyclingSpeedWatcher1"/>
    <allow send_interface="org.bluez.GattCharacteristic1"/>
    <allow send_interface="org.bluez.GattDescriptor1"/>
    <allow send_interface="org.freedesktop.DBus.ObjectManager"/>
    <allow send_interface="org.freedesktop.DBus.Properties"/>
  </policy>
```

Next:

`sudo apt-get install python-dbus`

Use (OpenAPS must be installed), from OpenAPS init dir:

`openaps vendor add openxshareble`

`openaps device add share openxshareble`  (substitute any name for ‘share’)

`openaps use share configure --serial SM123456` (substitute the SM# off the back of your receiver)

`openaps use share -h`

To pair the receiver, open setttings, share, forget device (if previously paired), then turn sharing back on so that it can be paired with the Edison
On the Edison, in your openaps init directory, type `openaps use share iter_glucose 1`
The device should successfully pair, but may or may not pull the reading.  Subsequent attempts to pulll readings should now succeeed.
Readings run manually also print a lot of text before the actual readings appear.

***Note: currently this command fails:    `openaps use cgm iter_glucose_hours` #    
Instead use    `openaps use cgm iter_glucose` #      (this pulls any number of previous readings successfully, e.g., 300)
Remember to create your reports using the number of records (iter_glucose #).

The following are examples to set up a share receiver within the openaps init directory.  The Share receiver is named 'cgm' below
```
openaps device add cgm openxshareble 
openaps use cgm configure --serial $share_serial
git add cgm.ini
```

```
openaps report add monitor/cgm-glucose.json JSON cgm iter_glucose 300
openaps report add monitor/ns-glucose.json text ns-glucose shell
openaps alias add monitor-cgm "report invoke monitor/cgm-glucose.json"
openaps alias add get-bg '! bash -c "( openaps monitor-cgm 2>/dev/null | tail -1 && grep -q glucose monitor/cgm-glucose.json && rsync -rtu monitor/cgm-glucose.json monitor/glucose.json ) || ( openaps get-ns-glucose && cat monitor/ns-glucose.json | json -c \"minAgo=(new Date()-new Date(this.dateString))/60/1000; return minAgo < 10 && minAgo > -5 && this.glucose > 30\" | grep -q glucose && mv monitor/ns-glucose.json monitor/glucose.json )"'
```
