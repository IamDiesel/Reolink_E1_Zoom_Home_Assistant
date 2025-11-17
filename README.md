# Reolink_E1_Zoom_Home_Assistant
## Motivation
Reolink has a very good home assistant integration for the E1 Zoom IP Camerpa. However, the only downside is that the two way talk feature is only accessible via the reolink mobile or desktop app. The downside on these apps is that the baby cry sound detection is not implemented in these apps.
So you can either use homeassistant without two way talk but with baby cry detection or you can use the mobile/desktop app without baby cry sound detection but with two way talk.
Since I don't like to install additional apps - here is how to get two way talk operational in homeassistant. And along side we will be solving another issue: How can we make home assistant reachable via http AND https at the same time.

A) Enable https for home assistant (in this how-to https and http will be operational at the same time)
B) Install Homeassistant addons Frigate and Advanced Camera Card for Video support and Two Way Talk


# Enable http AND https via reverse proxy NGINX
Follow this description: https://gist.github.com/tiagofreire-pt/4920be8d03a3dfa8201c6afedd00305e
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

Next start the plugin and hit the checkbox start at boot then restart home assistant.
<img width="895" height="427" alt="image" src="https://github.com/user-attachments/assets/2b06454e-7d24-4b64-9958-7ad639d531ea" />





