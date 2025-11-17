# Reolink_E1_Zoom_Home_Assistant
## Motivation
Reolink has a very good home assistant integration for the E1 Zoom IP Camerpa. However, the only downside is that the two way talk feature is only accessible via the reolink mobile or desktop app. The downside on these apps is that the baby cry sound detection is not implemented in these apps.
So you can either use homeassistant without two way talk but with baby cry detection or you can use the mobile/desktop app without baby cry sound detection but with two way talk.
Since I don't like to install additional apps - here is how to get two way talk operational in homeassistant. And along side we will be solving another issue: How can we make home assistant reachable via http AND https at the same time.

1. Enable https for home assistant (in this how-to https and http will be operational at the same time
2. Install Homeassistant addons Frigate and Advanced Camera Card for Video support and Two Way Talk


# 1. Enable http AND https via reverse proxy NGINX
The following description is derived from: https://gist.github.com/tiagofreire-pt/4920be8d03a3dfa8201c6afedd00305e

Create Root Key
~~~
sudo openssl genrsa -des3 -out rootCA.key 4096
~~~

Create Root Certificate:
~~~
sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem
~~~

As an Example i entered the following data:
~~~
Country Name (2 letter code) [AU]:DE
State or Province Name (full name) [Some-State]:MyState
Locality Name (eg, city) []:MyCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:MyOrgFreetext
Organizational Unit Name (eg, section) []:---
Common Name (e.g. server FQDN or YOUR name) []: homeassistant_hostename.local
Email Address []:mymail@mailprovider.com
~~~

The homeassistant_hostname can be found in homeassistant under: Settings-->General-->Network-->Hostname
<img width="707" height="292" alt="image" src="https://github.com/user-attachments/assets/02e661a3-7771-47c0-a91d-b78c235f86fe" />

Create rootCA.csr.cnf file:
~~~
sudo nano rootCA.csr.cnf
~~~
with the following content (STRG+X to exit and save):
~~~
# rootCA.csr.cnf
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=DE
ST=MyState
L=MyCity
O=MyOrgFreetext
OU=---
emailAddress=mymail@mailprovider.com
CN = homeassistant_hostename.local
~~~

Create v3.ext file
~~~
sudo nano v3.ext
~~~
Enter the following content
~~~
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
extendedKeyUsage=serverAuth

[alt_names]
DNS.1 = homeassistant_hostename.local
IP.1 = 192.168.X.XX
~~~
Where 192.168.XXX.XXX is your home assistant ip address.


Create the certificate key
~~~
openssl req -new -sha256 -nodes -out hassio.csr -newkey rsa:2048 -keyout hassio.key -config <( cat rootCA.csr.cnf )
~~~

Create the certificate itself
~~~
sudo openssl x509 -req -in hassio.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out hassio.crt -days 3650 -sha256 -extfile v3.ext
~~~

Copy and rename files to home assistant ssl folder (I'm using homeassistant supervised. The folder could be different for other home assistant installation methods):
~~~
sudo cp hassio.crt  /usr/share/hassio/ssl/fullchain.pem
sudo cp hassio.key  /usr/share/hassio/ssl/privkey.pem 
~~~

Change Permissions:
~~~
sudo chmod 600 /usr/share/hassio/ssl/fullchain.pem /usr/share/hassio/ssl/privkey.pem
~~~

Now install NGINX Addon (see home assistant addons / https://github.com/home-assistant/addons/tree/master/nginx_proxy) and configure the plugin as follows:
<img width="960" height="1114" alt="image" src="https://github.com/user-attachments/assets/6bd9a55d-b85a-4412-b8a5-478858948292" />

Next start the plugin and hit the checkbox "start at boot" then restart home assistant.
<img width="895" height="427" alt="image" src="https://github.com/user-attachments/assets/2b06454e-7d24-4b64-9958-7ad639d531ea" />

Copy and install rootCA.pem to all devices that will access homeassistant. I used WinSCP to connect to my homeassistant server and exported the file to my pc and smartphone.
Under Android the certificate can be installed by searching "install certificate" in the settings menue and then install rootCA.pem

Now homeassistant can be reached either via:
1. http://homeassistant_hostename.local
2. https://homeassistant_hostename.local

If you want to access home assistant via https from the companion app just open the companion app on your mobile phone, go to settings --> your phone --> and set the homeassistant url to:
 ~~~
https://homeassistant_hostename.local
~~~

# 2. Install Homeassistant addons Frigate and Advanced Camera Card for Video support and Two Way Talk

Preparation:

- Configure a static ip for your reolink camera in your router
- enter your cameras ip in a browser and login as admin (initially no password is set)
- create standard user with username and password (and also change the admin password (gear icon -> system -> user management)
- configure reolink network settings (gear icon -> network -> advanced -> server settings)
<img width="1056" height="961" alt="image" src="https://github.com/user-attachments/assets/ab342dad-9c40-4305-8438-acd8d2137344" />


A. Install the frigate home assistant addon via HCAS. Details see
~~~
https://docs.frigate.video/integrations/home-assistant/
~~~
B. In the frigate addon go to settings -> Configuration editor
<img width="335" height="641" alt="image" src="https://github.com/user-attachments/assets/7abf952a-0e29-4a81-825a-b80856e8de16" />


Enter the following config (192.168.2.101 should of course be the ip of your camera)
~~~
mqtt:
  enabled: false

go2rtc:
  streams:
    # example for connecting to a standard Reolink camera
    your_reolink_camera:
       - rtsp://youruser:yourpassword@192.168.2.101/Preview_01_sub
       - ffmpeg:rtsp://youruser:yourpassword@192.168.2.101/Preview_01_sub#audio=pcm#audio=volume
#      - ffmpeg:http://192.168.2.101/flv?port=1935&app=bcs&stream=channel0_ext.bcs&user=youruser&password=yourpassword#backchannel=1#audio=opus#unicast=true#proto=Onvif
#      - rtsp://youruser:yourpassword!@192.168.2.101/Preview_01_sub
#    your_reolink_camera_sub:
#      - ffmpeg:rtsp://192.168.2.101/flv?port=1935&app=bcs&stream=channel0_ext.bcs&user=youruser&password=yourpassword#backchannel=1

cameras:
  your_reolink_camera:
    ffmpeg:
      inputs:
        #- path: rtsp://youruser:yourpassword@192.168.2.101:554?video=copy&audio=opus#backchannel=0
        - path: rtsp://youruser:yourpassword@192.168.2.101/Preview_01_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
            - record
            - audio
      output_args:
          record: preset-record-generic-audio-copy
#        - path: rtsp://youruser:yourpassword@192.168.2.101:554?video=copy#backchannel=1
#          input_args: preset-rtsp-restream
#          roles:
#            - detect


detect:
  enabled: true
version: 0.16-0
camera_groups:
  Test:
    order: 1
    icon: LuAlbum
    cameras: your_reolink_camera
~~~

C. Install advanced camera card addon via HACS (https://github.com/dermotduffy/advanced-camera-card/)
D. In your Dashboard create a camera card with the following code:
~~~
type: custom:advanced-camera-card
cameras:
  - camera_entity: camera.your_reolink_camera
    live_provider: go2rtc
    go2rtc:
      modes:
        - webrtc
menu:
  buttons:
    microphone:
      enabled: true
      type: toggle
grid_options:
  columns: 27
  rows: auto
~~~

Two way audio should now be possible when home assistant is accessed via https:

<img width="1053" height="590" alt="image" src="https://github.com/user-attachments/assets/a727794d-4a81-472f-b1b7-3855650632e8" />



The rest of the functionality can be made available by installing the reolink integration and setting up the camera: https://www.home-assistant.io/integrations/reolink/

<img width="466" height="393" alt="image" src="https://github.com/user-attachments/assets/8ae8b62e-5064-4028-bbb5-c04747275c02" />









