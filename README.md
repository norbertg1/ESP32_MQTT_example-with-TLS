# ESP32 MQTT example
TLS Secured connection example between Mosquitto broker (MQTT) and ESP32 board with possible client authentication.\
For the following guide I used Ubuntu system, but its possible to run on any other system.

### Prerequisite:
 1. Installed Arduino IDE with ESP32 framework\
    or Visual Studio Code with PlatfoemIO with ESP32 support\
    https://github.com/espressif/arduino-esp32
 3. PubSubClient library for Arduino - https://github.com/knolleary/pubsubclient/
 4. OpenSSL - https://www.openssl.org/
 5. Mosquitto broker - https://mosquitto.org/

## Guide

```git clone https://github.com/norbertg1/ESP32_MQTT_example-with-TLS.git```

### 1. step - Generate the certificates:
Open the *certificates/certificate_generator.sh* script **modify the Mosquitto_borker_adress to your server adress!**\
Run the script:\
```./certificates/certificates_generator.sh```\
If you need add execute permission to it:\
```chmod -c +x certificate_generator.sh```

The script generates esp_certificates.h file with const char CA_cert[], const char ESP_CA_cert[] and const char ESP_RSA_key[] These arrays contains the certificates needed for connection to your Mosquitto server.

Or alternatevly you can use these commands (Modify them if you need):
```
openssl req -new -x509 -days 365 -extensions v3_ca -keyout ca.key -out ca.crt -subj '/CN=TrustedCA.net'  
#If you generate self-signed certificates the CN can be anything

openssl genrsa -out mosquitto.key 2048
openssl req -out mosquitto.csr -key mosquitto.key -new -subj '/CN=Mosquitto_borker_adress'    
#!!!!Its necessary to set the CN to the adress of which the client calls your Mosquitto server (eg. yourserver.com)!!!
openssl x509 -req -in mosquitto.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out mosquitto.crt -days 365 

#This is only needed if the mosquitto broker requires a client autentithication (require_certificate is set to true in mosquitto config)
openssl genrsa -out esp.key 2048
openssl req -out esp.csr -key esp.key -new -subj '/CN=localhost'
openssl x509 -req -in esp.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out esp.crt -days 365
```

### 2. step - Install and setup Mosquitto broker
```sudo apt-get install mosquitto```
#### config

Copy the generated certificates ca.crt to /etc/mosquitto/ca_certificates/ and mosquitto.crt, mosquitto.key to /etc/mosquitto/certs/
```
sudo cp certificates/ca.crt /etc/mosquitto/ca_certificates/
sudo cp certificates/mosquitto.crt /etc/mosquitto/certs/
sudo cp certificates/mosquitto.key /etc/mosquitto/certs/
```
Edit the config file:\
```sudo nano /etc/mosquitto/conf.d/default.conf```\
add to it the following lines:
```
#listen to localhost without needing of certificate
listener 1883 localhost
allow_anonymous true

listener 8883
cafile /etc/mosquitto/ca_certificates/ca.crt
keyfile /etc/mosquitto/certs/mosquitto.key
certfile /etc/mosquitto/certs/mosquitto.crt
require_certificate true #or false if you dont need client authentication
log_type all  #for logging in /var/log/mosquitto/
```
stop and start a mosquitto broker with this config file:
```
sudo service mosquitto stop
mosquitto -c /etc/mosquitto/conf.d/default.conf
```
Dont forget forward on you router incoming connections on 8883 port to your Mosquitto broker!

### 3. step - Compile the program and start Mosquitto listeners
Compile and upload ESP32 sketch.

Install Mosquitto clients for listening communications and topics.
```sudo apt  install mosquitto-clients```
start the client to listen port 1883 and the topic LivingRoom/Temperature
```mosquitto_sub -h "server_local_adress" -p 1883 -t "LivingRoom/Temperature"```
