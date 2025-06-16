# Alpine Linux NUT and SNMP Installation on Raspberry Pi
- NUT and SNMP install on Alpine Linux on Raspberry Pi Instructions
- As seen on the XtendedGreg YouTube Live Stream: 

# Server Side
## Description
Here is the complete, consolidated guide for setting up your Raspberry Pi with Alpine Linux to act as a NUT network server and SNMP host for a Cyberpower CP1500PFCLCDa UPS.

### Objective
- Monitor a Cyberpower CP1500PFCLCDa UPS via USB on a Raspberry Pi running Alpine Linux.
- Configure the Raspberry Pi as a NUT (Network UPS Tools) server.
- Expose the UPS status via SNMP for network monitoring.
- Name the UPS resource "UPS-Central01".

## Server Installation
Starting from a base installation of Alpine Linux loaded to an SD Card with setup-alpine already run with LBU configured and internet connectivity established.

### Step 1: Install Software Packages
First, connect to your Raspberry Pi's terminal. Update your package manager and install NUT and net-snmp.

```
# Update the package list
apk update

# Install NUT and Net-SNMP
apk add nut net-snmp

# Install Nano text editor and Net SNMP Tools to check our configuration later
apk add nano net-snmp-tools

# Save with LBU
lbu commit -d
```

### Step 2: Configure udev for USB Permissions
You must create a udev rule to grant the nut user permission to access the USB device.

Create the udev rule file:
```
nano /etc/udev/rules.d/99-nut-ups.rules
```

Add the following line. This line specifically targets your Cyberpower CP1500PFCLCDa model.
```
SUBSYSTEM=="usb", ATTR{idVendor}=="0764", ATTR{idProduct}=="0601", MODE="0660", GROUP="nut"
```
Save and exit the editor (Ctrl+X, then Y, then Enter).

Reload the udev rules to apply the new rule without rebooting:
```
udevadm control --reload-rules
```
### Step 3: Configure the NUT Driver (ups.conf)
This file tells NUT what driver to use and how to find your UPS.

Edit the main NUT configuration file:
```
nano /etc/nut/ups.conf
```
Add the following configuration block. This defines your UPS, names it "UPS-Central01", and includes the specific vendor/product IDs and a recommended polling interval for stability.
```
[UPS-Central01]
    driver = usbhid-ups
    port = auto
    desc = "Cyberpower CP1500PFCLCDa"
    vendorid = 0764
    productid = 0601
    pollinterval = 15
```
Save and exit the editor.

Save with LBU
```
lbu commit -d
```

### Step 4: Configure NUT Server Mode
You need to tell NUT to run in netserver mode so other computers on your network can access the UPS data.

Edit the nut.conf file:
```
nano /etc/nut/nut.conf
```

Find and set the MODE variable:
```
MODE=netserver
```
Save and exit.

### Step 5: Configure the NUT Server Daemon (upsd.conf)
This file controls how the NUT server behaves and who can connect.

Edit the upsd.conf file:
```
nano /etc/nut/upsd.conf
```
Add a LISTEN directive. To allow other devices on your network to connect, make the server listen on all network interfaces. Add this line at the top of the file.
```
LISTEN 0.0.0.0
```
Note: The default port is 3493. If you have a firewall, ensure this port is open.
Save and exit.

Save with LBU
```
lbu commit -d
```

### Step 6: Create a NUT Monitoring User (upsd.users)
Create a user account that monitoring clients (including our own SNMP script) can use to access NUT data.

Edit the upsd.users file:
```
nano /etc/nut/upsd.users
```
Add a user definition. Replace your_secure_password with a strong password of your choice.
```
[monitor]
    password = your_secure_password
    upsmon slave
```
Save and exit.

### Step 7: Start and Test NUT Services
Now it's time to start the services and ensure they work correctly.

Start the NUT server:
```
rc-service nut-upsd start
```

Enable the service to start on boot:
```
rc-update add nut-upsd default
```

Test the connection to the UPS. Run the upsc (UPS Client) command:
```
upsc UPS-Central01@localhost
```
You should see a list of status variables from your UPS, such as battery.charge, input.voltage, ups.load, and ups.status. If you see an error, double-check the previous steps, especially the udev rule and ups.conf.

Save with LBU
```
lbu commit -d
```

### Step 8: Configure the SNMP Service
With NUT working, you can now configure snmpd to get data from it.

Create a script to bridge NUT to SNMP:
```
nano /usr/local/bin/nut-snmp.sh
```

Paste the following script into the file. This script uses the upsc command to fetch specific metrics.
```
#!/bin/sh

# This script provides NUT UPS data to the snmpd daemon

# The name of the UPS defined in your ups.conf
readonly UPS_NAME="UPS-Central01"
# The host where upsd is running
readonly UPS_HOST="localhost"

if ! command -v upsc >/dev/null 2>&1; then
  echo "Error: upsc command not found." >&2
  exit 1
fi

# Use a case statement to return the requested value
case "$1" in
  "battery.charge")
    upsc ${UPS_NAME}@${UPS_HOST} battery.charge 2>/dev/null || echo "0"
    ;;
  "battery.runtime")
    upsc ${UPS_NAME}@${UPS_HOST} battery.runtime 2>/dev/null || echo "0"
    ;;
  "input.voltage")
    upsc ${UPS_NAME}@${UPS_HOST} input.voltage 2>/dev/null || echo "0"
    ;;
  "ups.load")
    upsc ${UPS_NAME}@${UPS_HOST} ups.load 2>/dev/null || echo "0"
    ;;
  "ups.status")
    # SNMP can't easily handle strings, so we map status to an integer.
    # OL=1, OB=2, LB=3, Other=0
    STATUS=$(upsc ${UPS_NAME}@${UPS_HOST} ups.status 2>/dev/null)
    case "$STATUS" in
      "OL") echo "1" ;;       # Online
      "OB") echo "2" ;;       # On Battery
      "LB") echo "3" ;;       # Low Battery
      *)    echo "0" ;;       # Unknown or other status
    esac
    ;;
  *)
    echo "Unknown parameter: $1" >&2
    exit 1
    ;;
esac

exit 0
```

Make the script executable:
```
chmod +x /usr/local/bin/nut-snmp.sh
```

Add script to LBU to persist past reboots
```
lbu add /usr/local/bin/nut-snmp.sh
```

Save with LBU
```
lbu commit -d
```

### Step 9: Configure and Extend snmpd
Now, configure the main SNMP daemon and tell it to use your new script.

Edit the snmpd.conf file:
```
nano /etc/snmp/snmpd.conf
```
Scroll down to find an existing entry of ```rocommunity``` and comment it out using '#' and add the following below it. Configure a read-only community string. Replace your_community_string with a secure, private string that your monitoring system will use.
```
rocommunity your_community_string
```

Add extend directives. These lines tell snmpd to run your script to get values for specific OIDs (Object Identifiers). Add these to the end of the file.
```
#--- NUT UPS Monitoring ---
# .1.3.6.1.4.1.8072.1.3.2.3.1.1.14.98.97.116.116.101.114.121.46.99.104.97.114.103.101 -> battery.charge
# .1.3.6.1.4.1.8072.1.3.2.3.1.1.15.98.97.116.116.101.114.121.46.114.117.110.116.105.109.101 -> battery.runtime
# .1.3.6.1.4.1.8072.1.3.2.3.1.1.13.105.110.112.117.116.46.118.111.108.116.97.103.101 -> input.voltage
# .1.3.6.1.4.1.8072.1.3.2.3.1.1.8.117.112.115.46.108.111.97.100 -> ups.load
# .1.3.6.1.4.1.8072.1.3.2.3.1.1.10.117.112.115.46.115.116.97.116.117.115 -> ups.status (1=OL, 2=OB, 3=LB)

extend battery.charge /usr/local/bin/nut-snmp.sh battery.charge
extend battery.runtime /usr/local/bin/nut-snmp.sh battery.runtime
extend input.voltage /usr/local/bin/nut-snmp.sh input.voltage
extend ups.load /usr/local/bin/nut-snmp.sh ups.load
extend ups.status /usr/local/bin/nut-snmp.sh ups.status
```
Save and exit.

### Step 10: Start and Test SNMP Service
Finally, start the SNMP service and test it.

Start the SNMP daemon:
```
rc-service snmpd start
```

Enable the service to start on boot:
```
rc-update add snmpd default
```

Test the SNMP setup. From another computer on your network that has SNMP tools, run an snmpwalk. Replace your_pi_ip and your_community_string.
```
snmpwalk -v2c -c your_community_string your_pi_ip NET-SNMP-EXTEND-MIB::nsExtendOutputFull
```
You should see output similar to this, showing the values retrieved from your UPS:
```
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."battery.charge" = STRING: 100
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."battery.runtime" = STRING: 2400
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."input.voltage" = STRING: 121.5
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."ups.load" = STRING: 15
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."ups.status" = STRING: 1
```
Save with LBU
```
lbu commit -d
```

Your Raspberry Pi is now fully configured to monitor your "UPS-Central01" and serve its status via NUT and SNMP.

# Client Side

## Description
These instructions will guide you through the process of configuring a computer on your network to act as a NUT client. This client will monitor the status of your "UPS-Central01" UPS by connecting to the NUT server you've set up on your Raspberry Pi. If the UPS loses power, the client machine can be configured to shut down gracefully.

### Prerequisites
- Network Connectivity: The client machine must be on the same network as your Raspberry Pi NUT server and be able to reach it via its IP address.
- Server Information: You will need the following information from your server configuration:
- Server IP Address: The IP address of your Raspberry Pi (referred to as your_pi_ip).
- UPS Name: UPS-Central01
- Username: monitor
- Password: The your_secure_password you configured in the server's upsd.users file.

## Client Installation
This will cover installation steps for connecting various linux distributions to the NUT server you just created.

### Step 1: Install NUT Client Software
First, you need to install the NUT package on the client machine. The package name may vary depending on the Linux distribution your client is running.

For Alpine Linux clients:
```
# Update the package list
apk update

# Install the NUT package
apk add nut

# Install nano text editor if it's not already present
apk add nano

# Save the new package installation
lbu commit -d
```

For Debian/Ubuntu clients:
```
# Update the package list
sudo apt update

# Install the NUT client package
sudo apt install nut-client
```

For Fedora/CentOS/RHEL clients:
```
# Install the NUT client package
sudo dnf install nut-client
```

### Step 2: Configure NUT for Slave Mode
On the client machine, you need to configure NUT to run in slave mode. This tells the system that it will be monitoring a UPS that is connected to another machine on the network.

Edit the nut.conf file:
```
sudo nano /etc/nut/nut.conf
```

Find the MODE line and set it to slave:
```
MODE=netclient
```
Save and exit the editor (Ctrl+X, then Y, then Enter).

### Step 3: Configure the Monitor Daemon (upsmon.conf)
This is the most critical step. You will tell the client's monitor daemon (upsmon) how to find and authenticate with your NUT server.

Edit the upsmon.conf file:
```
sudo nano /etc/nut/upsmon.conf
```
Scroll to the end of the file and add a MONITOR line. This line defines the remote UPS to be monitored. The format is MONITOR <system>@<hostname> <powervalue> <username> <password> <type>.

Add the following line, replacing your_pi_ip and your_secure_password with the actual values from your server setup:
```
MONITOR UPS-Central01@your_pi_ip 1 monitor your_secure_password primary
```
- UPS-Central01@your_pi_ip: Specifies the UPS name and the IP address of your NUT server.
- 1: Is the powervalue, which is standard for a single UPS system.
- monitor: The username you created on the server.
- your_secure_password: The password for the monitor user.
- primary: Indicates that this is the primary power supply.
Save and exit the editor.

### Step 4: Start and Test the NUT Client Service
Now, you can start the NUT client service and verify that it's successfully communicating with the server.

Start the NUT client service:

For Alpine Linux:
```
# The service is named 'nut-upsmon' in Alpine
rc-service nut-upsmon start
```

For Debian/Ubuntu/Fedora/CentOS:
```
# The service is typically named 'nut-client'
sudo systemctl start nut-client.service
```

Enable the service to start on boot:

For Alpine Linux:
```
rc-update add nut-upsmon default

# Save with LBU
lbu commit -d
```

For Debian/Ubuntu/Fedora/CentOS:
```
sudo systemctl enable nut-client.service
```

Test the connection:

After a few seconds, run the upsc (UPS Client) command on the client machine to poll the status from your server.
```
upsc UPS-Central01@your_pi_ip
```
You should see the same list of status variables from your UPS that you saw when you tested on the server itself (battery.charge, input.voltage, ups.status, etc.).

If you see an "Access denied" error, double-check the password in upsmon.conf. If it times out, ensure your_pi_ip is correct and that no firewall is blocking TCP port 3493 between the client and the server.

### Step 5: (Optional) Configure Automatic Shutdown
The primary benefit of a NUT client is to allow the machine to shut down safely when the UPS battery is low. The upsmon daemon handles this automatically based on the status flags (OB - On Battery, LB - Low Battery) sent by the server.

You can adjust shutdown behavior in the upsmon.conf file, but the default settings are generally sufficient. By default, upsmon will initiate a system shutdown when the UPS it is monitoring reaches a "Low Battery" state.

Your client is now fully configured. It will monitor the "UPS-Central01" server and automatically initiate a shutdown if the UPS reports a low battery condition, protecting the client machine from a sudden power loss.
