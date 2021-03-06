# Smart home server

This repository contains the code for the my small and personal smart-home project.

# TODOs

* Display home state
* Scenarios options handling (python + ui)
    * Early wake up option
* Improve scenarios:
    * Heating based on inside and outside temp
    * Lights based on sun altitude
    * Add movie scenario?
* Add auto triggers:
    * light on based on home state + sunset
    * heating based on home state + temperatures
* Schedule events in calendar
* Better radio widget
* Voice recognition
* Velib status
* Use _Embedded Google Assistant SDK_ (yet to be released) or _Alexa Voice Service_ for voice control + feedback

# Hardware

* **Server:** Raspberry Pi
* **Devices:**
    * Philips Hue bride
    * Philips Hue color lamp bulb
    * 8 Etekcity RF-controlled power-plug adapters
    * Speakers + jack
    * NAS + wire soldered on the power-on switch
    * USB microphone
* **Electronics:**
    * 5V to 3.3V adapter
    * RF 433 MHz transmitter & receiver
    * IR transmitter & receiver
    * Temperature sensor (DHT11)
    * Motion sensor (PIR)

# Software

## Packages

```
# apt-get

## Basics
zsh python-pip git tig htop

## Routing & web
dnsmasq nginx php5-fpm

## Python
python python-pip python3 python3-pip

## Text-to-speech
lame alsa-mixer libttspico-utils sox

# Python packages

## pip
google-api-python-client requests pytz python-dateutil

## DHT library
git clone https://github.com/adafruit/Adafruit_Python_DHT.git
cd Adafruit_Python_DHT
sudo python3 setup.py install
```

## dnsmask setup

To be able to give hostnames to our local IPs, we will use the raspi as a DNS server using `dnsmask`.

This config file in `/etc/dnsmask.conf` will serve the domains in `/etc/hosts`:

```
domain-needed # Don't forward bogus request to the real DNS
bogus-priv # Don't forward bogus request to the real DNS
no-resolv # Don't look at resolv.conf
server=192.168.1.254 # Default DNS
local=/local/ # Pattern for local domains
```

**Auto-startup:** `sudo systemctl enable dnsmasq`

## Mopidy (MPD music server with Google Music)

```
wget -q -O - https://apt.mopidy.com/mopidy.gpg | sudo apt-key add -
sudo wget -q -O /etc/apt/sources.list.d/mopidy.list https://apt.mopidy.com/jessie.list
sudo apt-get install mopidy
sudo pip install mopidy-gmusic
```

## Web server

`sudo /etc/init.d/nginx start`

Comfiguration in `sites-enabled/default`:

```
server {
    
    # [...]
    
    # Enable PHP index
    index index.html index.php;

	server_name _;

    # Allow access from local IP without password, outside with password
    # Please create the /etc/nginx/htpasswd/default.htpasswd file
	satisfy any;
	allow 192.168.0.0/16;
	allow 127.0.0.1/32;
	deny all;
	auth_basic "Home";
	auth_basic_user_file /etc/nginx/htpasswd/default.htpasswd;

    # Redirect API calls
	location ~ ^/api/(.*)$ {
		proxy_pass http://<API_URL>/$1$is_args$args;
	}

    # Redirect HUE calls
	location ~ ^/hue/(.*)$ {
		proxy_pass http://<HUE_URL>/api/<HUE_USERNAME>/$1$is_args$args;
	}

    # Regular files are served
	location / {
		try_files $uri $uri/ =404;
	}

	# Handle PHP
	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/var/run/php5-fpm.sock;
	}
}
```

## lirc

`sudo apt-get install lirc`

### Configuration

Edit `/boot/config.txt`:

```
# [...]
dtoverlay=lirc-rpi,gpio_in_pin=24,gpio_out_pin=23
# [...]
```

Edit `/etc/lirc/hardware.conf`:

```
LIRCD_ARGS=""
#START_LIRCMD=false
#START_IREXEC=false
LOAD_MODULES=true

DRIVER="default"
DEVICE="/dev/lirc0"
MODULES="lirc_rpi"

LIRCD_CONF=""
LIRCMD_CONF=""
```

### Recording a remote

To test the receiver:

```
sudo /etc/init.d/lirc stop
mode2 -d /dev/lirc0
```

To record a remote: `irrecord -d /dev/lirc0 ~/lircd.conf`

### Sending signals

My config file `/etc/lircd/lircd.conf`:

```
begin remote

  name  mistlamp
  bits           16
  flags SPACE_ENC|CONST_LENGTH
  eps            30
  aeps          100

  header       8974  4509
  one           548  1706
  zero          548   605
  ptrail        549
  repeat       8980  2273
  pre_data_bits   16
  pre_data       0x1FE
  gap          107495
  toggle_bit_mask 0x0

      begin codes
          KEY_BRIGHTNESS_CYCLE     0x29D6
          KEY_POWER                0x23DC
          BTN_MODE                 0x6996
          KEY_POWER2               0x01FE
      end codes

end remote
```

* Start the service :  
  `sudo service lirc start`  
  `sudo lircd --device /dev/lirc0`
* Test remote commands: `irsend LIST mistlamp ""`
* Send command: `irsend SEND_ONCE mistlamp KEY_POWER`

## Smart-home crons

```
* * * * *    cd /home/pi/dev/smart-home && ./continuous-run.sh
*/10 * * * * curl -X GET http://<API_URL>/triggers/calendar/update
* * * * *    curl -X GET http://<API_URL>/triggers/calendar/trigger
```