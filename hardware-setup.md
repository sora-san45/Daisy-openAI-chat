You can set up your Raspberry Pi Zero to broadcast its own WiFi access point (AP) and create a setup page by following these steps:

# Install the required software:

```
sudo apt-get update
sudo apt-get install hostapd dnsmasq
```

# Create a configuration file for dnsmasq:
```
sudo nano /etc/dnsmasq.conf

```

and add the following lines to the file:

```
interface=wlan0
dhcp-range=10.0.0.2,10.0.0.5,255.255.255.0,24h
```

 main
To configure the Raspberry Pi to have a different IP address when it's broadcasting an access point, you can use the dnsmasq DHCP server that's installed as part of the access point setup.

When a device connects to the Raspberry Pi's access point, it will automatically be assigned an IP address in the range specified in the dhcp-range setting in the dnsmasq.conf file. By default, the Raspberry Pi itself will be assigned the first IP address in the range (e.g., 10.0.0.2 if the range is 10.0.0.2,10.0.0.5). You can change this by modifying the dhcp-host setting in the dnsmasq.conf file to specify a fixed IP address for the Raspberry Pi's WiFi interface.

Determine the MAC address of the Raspberry Pi's WiFi interface by running the following command:
```
ifconfig wlan0 | grep -oE '([[:xdigit:]]{1,2}:){5}[[:xdigit:]]{1,2}'
```
This command will display the MAC address of the wlan0 interface.

Edit the dnsmasq.conf file:
```
sudo nano /etc/dnsmasq.conf
```

Add the following line to the file, replacing XX:XX:XX:XX:XX:XX with the MAC address of the Raspberry Pi's WiFi interface and 10.0.0.1 with the desired IP address:

```
dhcp-host=XX:XX:XX:XX:XX:XX,10.0.0.1
```
This configuration tells dnsmasq to always assign the IP address 10.0.0.1 to the device with the specified MAC address.

Restart the dnsmasq service:
```
sudo systemctl restart dnsmasq
```

Reboot the Raspberry Pi:
```
sudo reboot
```



Create a configuration file for hostapd:
```
sudo nano /etc/hostapd/hostapd.conf
```

and add the following lines to the file:
```
interface=wlan0
driver=nl80211
ssid=daisy
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=halfcrazy
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
country_code=US
```

Replace MyAP with your desired network name and MyPassword with your desired password.

Edit the hostapd default configuration file:
```
sudo nano /etc/default/hostapd
```

and change the following line:
```
#DAEMON_CONF=""
```

to
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Restart the dnsmasq service:
```
sudo service dnsmasq restart
```

Start the hostapd service:
```
sudo service hostapd start
```


The error message "Failed to start hostapd.service: Unit hostapd.service is masked" means that the hostapd service is currently disabled. To fix this, you need to unmask the service and enable it.

You can do this by running the following commands:
```
sudo systemctl unmask hostapd.service
sudo systemctl enable hostapd.service
sudo systemctl start hostapd.service
```

After running these commands, try starting the hostapd service again with the command:
```
sudo service hostapd start
```

The error message "Failed to start hostapd.service: Unit hostapd.service is masked" means that the hostapd service is currently disabled. To fix this, you need to unmask the service and enable it.

You can do this by running the following commands:

```
sudo systemctl unmask hostapd.service
sudo systemctl start hostapd.service
```

After running these commands, try starting the hostapd service again with the command:

```
sudo service hostapd start
```

It should now start without any issues.

If the wlan0 interface is "soft blocked", run the command below and start hostapd again.
```
rfkill unblock wifi
```


# Configure network interfaces
It should now start without any issues.




Edit the network interfaces configuration:
```
sudo nano /etc/network/interfaces
```

and change the file to look like this:
```
auto lo
iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet dhcp

auto wlan0
iface wlan0 inet dhcp
  wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

Finally, restart your Raspberry Pi:
```
sudo reboot
```

After rebooting, your Raspberry Pi should be broadcasting its own WiFi AP named "daisy". You can connect to it and then use the setup page to connect to your desired WiFi network.




# Wifi configuration page

To set up the setup page, you'll need to create an HTML form that collects the SSID and password of the desired WiFi network. You can serve this form using a lightweight web server like Flask. Here's an example:

Install Flask:
```
sudo apt-get install python3-flask
```

Create a Flask app:
```
nano wifi_setup.py
```

and add the following code:
```
from flask import Flask, request
import subprocess
import logging
from logging.handlers import RotatingFileHandler
import threading
import time
import os

app = Flask(__name__)

# Configure the logger
handler = RotatingFileHandler('app.log', maxBytes=10000, backupCount=1)
handler.setLevel(logging.INFO)
app.logger.addHandler(handler)

def reboot():
    time.sleep(15)
    subprocess.call("reboot", shell=True)

def check_wifi_status():
    # Check if wpa_supplicant.conf exists
    if not os.path.exists("/etc/wpa_supplicant/wpa_supplicant.conf"):
        return "wpa_supplicant.conf is not found."
    
    # Run wpa_supplicant and check for authentication failure message
    if subprocess.run(["wpa_supplicant", "-B", "-c/etc/wpa_supplicant/wpa_supplicant.conf", "-iwlan0"], capture_output=True, text=True).returncode == 0:
        return "Connected successfully!"
    else:
        if "Authentication with" in subprocess.run(["grep", "-q", "Authentication with", "/var/log/syslog"], capture_output=True, text=True).stdout:
            return "Incorrect password entered!"
        else:
            return "Failed to connect to network"

@app.route("/", methods=["GET", "POST"])
def index():
    app.logger.error('App requested')
    if request.method == "POST":
        ssid = request.form.get("ssid")
        password = request.form.get("password")
        if not ssid or not password:
            app.logger.error('Empty SSID or password.')
            return "SSID and password are required.", 400
        try:
            with open("/boot/wpa_supplicant.conf", "w", encoding="utf-8") as f:
                f.write(f"country=us\nupdate_config=1\nctrl_interface=/var/run/wpa_supplicant\n\nnetwork={{\n scan_ssid=1\n ssid=\"{ssid}\"\n psk=\"{password}\"\n}}")
            subprocess.call("wpa_cli -i wlan0 reconfigure", shell=True)
            app.logger.info('Successfully created wpa_supplicant.conf. Rebooting...')
            t = threading.Thread('''target=reboot''')
            t.start()
            return "<p><strong>Successfully created wpa_supplicant.conf!</strong></p><p>Connect to the target SSID and connect to http://daisy:8080. Rebooting...</p>"
        except Exception as e:
            app.logger.exception('Error connecting to Wi-Fi network: %s', e)
            return "Error connecting to Wi-Fi network.", 500
    return """
    <strong>{}</strong>
    <form method="post">
      <h1>Hi! I'm Daisy.</h1>
      <p>
          I won't work until you can connect me to the Internet.
      </p>
      <p>
          Enter credentials for the AP you would like to connect to.
      </p>
      <p>
          SSID: <input type="text" name="ssid"><br />
          Password: <input type="password" name="password">
      </p>
      <p>
          <input type="submit" value="Connect">
      </p>
    </form>
    """.format(check_wifi_status())

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

Start the Flask app:
```
FLASK_APP=wifi_setup.py flask run --host=0.0.0.0 --port=8080
```
Note: Non-admin users cannot serve Flask apps below a certain port so this must be served at 8080.










## Start Flask at startup

the Flask app does not start automatically on startup. To ensure that the app runs when you start your Raspberry Pi, you can create a systemd service file.

Here are the steps to create a systemd service file:

Create the file /etc/systemd/system/wifi-setup.service using a text editor such as nano:
```
sudo nano /etc/systemd/system/wifi-setup.service
```

Add the following lines to the file:
```
[Unit]
Description=WiFi Setup Flask App

[Service]
ExecStart=/usr/bin/env FLASK_APP=/home/daisy/wifi_setup.py flask run --host=0.0.0.0 --port=8080
WorkingDirectory=/home/daisy
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Replace /path/to with the actual path to the directory that contains wifi_setup.py. Also, replace pi with the username of the user who will run the app.

Save the file and exit the text editor.

Enable the service to run on startup:
```
sudo systemctl enable wifi-setup.service
```

Start the service:
```
sudo systemctl start wifi-setup.service
```

Now the Flask app should start automatically on startup.




# Keep in mind...

The systemctl daemon-reload command should be used after any changes are made to systemd unit files (such as service, timer, or socket files) to ensure that the changes take effect.
If you make any changes to service files, the systemctl daemon-reload command should be used after any changes are made to systemd unit files (such as service, timer, or socket files) to ensure that the changes take effect.

Here's an example of how to use systemctl daemon-reload:
```
sudo systemctl daemon-reload
```

This command should be executed with root privileges, so you'll need to use sudo or be logged in as the root user to run it.



# Automatically switch between SSID and AP mode

If the Raspberry Pi cannot connect to the SSID configured in the wpa_supplicant.conf file, you can add a script to automatically switch back to the AP mode and start broadcasting its own WiFi access point again.

One way to do this is to run a script periodically to check the network connection status and switch to the AP mode if the connection is lost. Here's an example of how to do this:

Create a script to check the network connection status and switch to the AP mode if the connection is lost. You can create a file called check_network.sh in the home directory:

```
#!/bin/bash

# Set the log file path
LOG_FILE="/home/daisy/wifi_failover.log"

# Log function
log() {
    local message="$1"
    local timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    echo "$timestamp - $message" >> "$LOG_FILE"
    echo "$timestamp - $message"
}

# Function to reconnect to the WiFi network
client_mode() {
    log "Found ${SSID}. Connecting to WiFi network."

    # Stop the hostapd and dnsmasq services used for setting up the AP
    log "RUNNING: sudo systemctl stop hostapd"
    sudo systemctl stop hostapd

    log "RUNNING: sudo systemctl stop dnsmasq"
    sudo systemctl stop dnsmasq

    sudo killall wpa_supplicant

    # Bring down the wlan0 interface
    log "RUNNING: sudo ip link set dev wlan0 down"
    sudo ip link set dev wlan0 down

    log "RUNNING: sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf"
    sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf

    log "RUNNING: sudo ip addr flush dev wlan0"
    sudo ip addr flush dev wlan0

    # Bring up the wlan0 interface
    log "RUNNING: sudo ip link set dev wlan0 up"
    sudo ip link set dev wlan0 up

    log "RUNNING: sudo dhclient wlan0"
    sudo dhclient wlan0


}

ap_mode(){

    # Bring down the wlan0 interface
    log "RUNNING: sudo ip link set dev wlan0 down"
    sudo ip link set dev wlan0 down

    # Stop the hostapd and dnsmasq services
    log "RUNNING: sudo systemctl stop hostapd"
    sudo systemctl stop hostapd

    log "RUNNING: sudo systemctl stop dnsmasq"
    sudo systemctl stop dnsmasq

    # Configure the Raspberry Pi to broadcast its own WiFi access point
    log "RUNNING: sudo ip addr flush dev wlan0"
    sudo ip addr flush dev wlan0

    log "RUNNING: sudo ip addr add ${AP_IP_ADDRESS}/24 dev wlan0"
    sudo ip addr add ${AP_IP_ADDRESS}/24 dev wlan0

    log "RUNNING: sudo ip link set dev wlan0 up"
    sudo ip link set dev wlan0 up

    log "RUNNING: sudo systemctl restart dnsmasq"
    sudo systemctl restart dnsmasq

    log "RUNNING: sudo systemctl restart hostapd"
    sudo systemctl restart hostapd
}


# Read SSID and password from wpa_supplicant.conf
SSID=$(grep -Po '(?<=ssid=")[^"]*' /etc/wpa_supplicant/wpa_supplicant.conf)
log "SSID: ${SSID}"
PASSWORD=$(grep -Po '(?<=psk=")[^"]*' /etc/wpa_supplicant/wpa_supplicant.conf)
log "Password: ${PASSWORD}"

# Get the AP SSID and IP from hostapd.conf and dnsmasq.conf
AP_SSID=daisy-dev
log "AP_SSID: ${AP_SSID}"
AP_IP_ADDRESS=10.0.0.1
log "AP_IP_ADDRESS: ${AP_IP_ADDRESS}"

# Function to check if a static IP address is set
is_static_ip() {
    local current_ip=$(ip -4 addr show wlan0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -n 1)
    log "Current IP: $current_ip"
    [ "$current_ip" = "${AP_IP_ADDRESS}" ]
}

# Function to check if connected to an SSID
is_connected_to_ssid() {
    local current_ssid=$(iwconfig wlan0 | grep -oP 'ESSID:"\K[^"]+')
    log "Connected to ${SSID}"
    [ "$current_ssid" = "$SSID" ]
}

# Check network connection status and switch to AP mode if the connection and SSID is lost
while true; do
    log ">>>CHECKING...<<<"

    #Check if static IP is set while connected to SSID
    if is_connected_to_ssid && is_static_ip; then
            log "Connected to ${SSID} with a static IP address. Reconnecting to the WiFi network with DHCP (client_mode)."
            client_mode
    fi

    #If in ap_mode (static IP is set), and target SSID is available, switch to client_mode
    if is_static_ip && sudo iwlist wlan0 scan | grep -q "${SSID}"; then
        # Call the function to reconnect to the WiFi network
        log "${SSID} found! Switching to client_mode"
        client_mode
        break
    fi

    #If SSID is lost for 2 checks, switch to ap_mode
    if ! sudo iwlist wlan0 scan | grep -q "${SSID}"; then
        log "Didnt detect ${SSID}. Checking again in 10s..."
        sleep 10
        if ! sudo iwlist wlan0 scan | grep -q "${SSID}"; then
            log "Lost ${SSID}. Broadcasting AP in ap_mode."
            ap_mode

        fi
    fi

    sleep 10
done

```

To create a service file for the script, create a file called wifi_failover.service in the /etc/systemd/system/ directory with the following content:
```
sudo nano /etc/systemd/system/wifi_failover.service 
```

```
[Unit]
Description=WiFi Failover Service
After=network.target

[Service]
ExecStart=/bin/bash /home/daisy/wifi_failover.sh

[Install]
WantedBy=multi-user.target
```

Replace /path/to/wifi_failover.sh with the actual path to the script file.

Make sure that the script file has execute permission. You can use the chmod command to add execute permission to the script file:
```
chmod +x /home/daisy/wifi_failover.sh
```

Reload the systemd daemon to pick up the changes:
```
sudo systemctl daemon-reload
```

Then, enable the service to start at boot with the following command:
```
sudo systemctl enable wifi_failover.service
```

Finally, start the service with:
```
sudo systemctl start wifi_failover.service
```

Now, the script will run continuously in the background and automatically restart at boot. You can check the status of the service with sudo systemctl status wifi_failover.service.


# Managing log files

You can configure logrotate to manage multiple log files. To do this, you can either create separate configuration files for each log file, or you can specify multiple log files within a single configuration file.

Here's an example of a logrotate configuration file that manages two log files:

```
sudo pico /etc/wifi_logrotate.conf
```

```
/home/daisy/log_wifi_setup.log /home/daisy/log_wifi_failover.log {
    size 10M
    rotate 10
    compress
    delaycompress
    missingok
    notifempty
    create 0644 username groupname
}
```

In this example, the configuration file specifies two log files: /path/to/bash.log and /path/to/other.log. The rest of the configuration options are the same as in the previous example.

When you run logrotate with this configuration file, it will rotate both log files according to the specified settings.

## Log rotate cron job

You can set up a cron job to run the logrotate command regularly. A cron job is a scheduled task that runs automatically at specified intervals.

To create a cron job for logrotate, you need to edit the crontab file. The crontab file contains a list of commands that are executed at specified times. Each user has their own crontab file, and you can edit it by running the crontab -e command.

Here's an example of a cron job that runs logrotate every day at midnight:

Open the crontab editor by running the following command:

```
crontab -e
```

Add the following line to the crontab file to run logrotate every day at midnight:

```
0 0 * * * /usr/sbin/logrotate /etc/wifi_logrotate.conf
```

This line specifies the following:
```
0 0 * * *: The time and frequency of the job. This means that the job runs every day at midnight.
/usr/sbin/logrotate: The command to run. This runs the logrotate utility.
/path/to/logrotate.conf: The configuration file to use. This specifies the location of the logrotate configuration file.
```
Save and exit the crontab editor.

With this cron job in place, logrotate will run automatically every day at midnight, rotating the specified log files according to the settings in the configuration file. You can adjust the frequency and timing of the job by modifying the cron job entry in the crontab file.



# And finally! Set up Daisy

```
git clone https://github.com/myrakrusemark/Daisy-openAI-chat.git
```

```
pip install --upgrade pip
```

```
sudo pip install -r requirements.txt
```


To set up a service for your Python file /home/daisy/daisy.py that restarts always and is run by the user "daisy", you can follow the steps below:

Create a systemd service unit file:
```
sudo nano /etc/systemd/system/daisy.service
```

Add the following content to the file, replacing <path_to_python> with the path to your Python executable:

```
[Unit]
Description=Daisy Service
After=network.target

[Service]
User=daisy
Restart=always
ExecStart=python3 /home/daisy/Daisy-openAI-chat/main.py

[Install]
WantedBy=multi-user.target
```

Reload the systemd daemon to pick up the new service:
```
sudo systemctl daemon-reload
```

Start the service:
```
sudo systemctl start daisy.service
```

Check the status of the service to make sure it is running:
```
sudo systemctl status daisy.service
```

Enable the service to start automatically at boot:
```
sudo systemctl enable daisy.service
```

Now your service is set up and will automatically start and restart as necessary, and will be run by the "daisy" user.




# Other Stuff

## OSError: libespeak.so.1: cannot open shared object file: No such file or directory

This error message suggests that the system is missing a required shared library file called "libespeak.so.1" that is needed by a program. Here are some possible steps to resolve the issue:

    Check if the library is installed: Run the command ldconfig -p | grep libespeak. If the library is installed, you should see its path printed out. If not, move on to the next step.

    Install the library: Depending on your operating system, you may be able to install the library using a package manager. For example, on Ubuntu or Debian-based systems, you can run sudo apt-get install libespeak1. On CentOS or Fedora-based systems, you can run sudo yum install espeak-ng-lib.

    Add the library path to LD_LIBRARY_PATH: If the library is installed but not in the default library search path, you can add its directory to the LD_LIBRARY_PATH environment variable. For example, you can run export LD_LIBRARY_PATH=/path/to/libespeak:$LD_LIBRARY_PATH (replace /path/to/libespeak with the actual path to the library directory).

    Rebuild the program: If you have access to the source code of the program that is giving the error, you can try rebuilding it to link against the correct library path.

Note that the specific steps required to resolve this issue may vary depending on your operating system and the specific program that is giving the error.


## Pygame Issues

Solve issues for pygame errors
https://forums.raspberrypi.com/viewtopic.php?t=136974

Switch audio output
https://forums.raspberrypi.com/viewtopic.php?t=203756

for microphone audio conversion
```
sudo apt-get install flac
```

## Choose audio output
```
raspi-config
```

## install WM89860 hat
https://github.com/waveshare/WM8960-Audio-HAT







