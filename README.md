# OpenVPN-Notes
**Notes** on _installing_, configuring and managing OpenVPN

OpenVPN is an Open Source solution for implementing a VPN server. In this article we will describe how to install and setup OpenVPN, as well as OpenVPN with 2 Factor Authentication. The setup described here is using the Ubuntu distribution of Linux. The initial server setup instructions have been adapted from Ubuntu’s official guide on setting up an OpenVPN server.

# Build the initial server
Install OpenVPN on Ubuntu 16.04 using instructions from OpenVPN

Instructions: https://help.ubuntu.com/lts/serverguide/openvpn.html

```
# Become the root user
sudo -i
# Install the OpenVPN packaged and the easy-rsa package
apt-get install easy-rsa openvpn
```
Setup the Public Key Infrastructure

Make a directory for easy-rsa in the OpenVPN directory

```
mkdir /etc/openvpn/easy-rsa
```

Copy the contents of easy-rsa to easy-rsa directory in OpenVPN directory

```
cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
```
Edit the contents of the file /etc/openvpn/easy-rsa/vars. These are the values for the attributes of the certificates that will secure your connection to the server. The default value of days until expiration is 10 years. I advise choosing a value of less than then. This will keep invalid keys floating around for less time, and force people to re-authenticate/re-validate themselves to connect to the server.
```
vi /etc/openvpn/easy-rsa/vars
```

```
export CA_EXPIRE=1095
export KEY_EXPIRE=365
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="San Francisco"
export KEY_ORG="Wikimedia"
export KEY_EMAIL="techsupport@wikimedia.org"
export KEY_OU="OIT"
```

The values from the vars file will then be used to build the Certificate Authority (CA), key and necessary items to secure the VPN.
```
source vars
./clean-all
./build-ca
```
Once the master certificate and key is created you can now create keys for the server the itself.
```
./build-key-server sykes
```
To enable the server to exchange keys, we need to configure Diffie Hellman
```
./build-dh
```
After the Certificate Authority and requisite items have been put in place, we can create the necessary certificates and keys to allow a client to connect.
```
cd /etc/openvpn/easy-rsa/
Source vars
./build-key bbogaert
```
Keeping in mind that we will eventually overlay 2 Factor Authentication to this VPN we need to make sure the common name of the certificate matches the username of the person. This username can either come from a local username, or username in a directory service such as LDAP.
```
root@sykes:/etc/openvpn/easy-rsa# ./build-key bbogaert
Generating a 2048 bit RSA private key
...................+++
.........................+++
writing new private key to 'bbogaert.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [US]:
State or Province Name (full name) [CA]:
Locality Name (eg, city) [San Francisco]:
Organization Name (eg, company) [Wikimedia]:
Organizational Unit Name (eg, section) [OIT]:
Common Name (eg, your name or your server's hostname) [bbogaert]:
Name [EasyRSA]:Byron Bogaert
Email Address [techsupport@wikimedia.org]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /etc/openvpn/easy-rsa/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'US'
stateOrProvinceName   :PRINTABLE:'CA'
localityName          :PRINTABLE:'San Francisco'
organizationName      :PRINTABLE:'Wikimedia'
organizationalUnitName:PRINTABLE:'OIT'
commonName            :PRINTABLE:'bbogaert'
name                  :PRINTABLE:'Byron Bogaert'
emailAddress          :IA5STRING:'techsupport@wikimedia.org'
Certificate is to be certified until May 25 18:49:31 2017 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
root@sykes:/etc/openvpn/easy-rsa#
```
To configure the server to run the OpenVPN service we will need to make a configuration file. OpenVPN copies a template to the server to be used. The template can be found in the documentation and examples folder on the server. Another template, linked here, is available on OpenVPN’s GitHub site.
https://raw.githubusercontent.com/OpenVPN/openvpn/master/sample/sample-config-files/server.conf

Change to the directory where the sample configuration files are.
```
cd /usr/share/doc/openvpn/examples/sample-config-files/
```
List the files to see what is available
```
ls
```
Copy the file we need to the openvpn directory
```
cp server.conf.gz /etc/openvpn/
```
Decompress the server.conf file to the openvpn directory
```
gzip -d /etc/openvpn/server.conf.gz
```
Edit the server.conf settings
```
vi /etc/openvpn/server.conf
```
I changed the default OpenVPN server port from 1194 to 443 to allow for more compatibility at Coffee Shops, Hotels, other businesses, etc. The next values are the location for the Certificate Authority certificate, the server certificate, and the server key.
```
port 443
ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/sykes.crt
key /etc/openvpn/easy-rsa/keys/sykes.key
```
Save the changes to the server configuration file.
Now, edit system control to allow ip forwarding.
```
vi /etc/sysctl.conf
```
Change the value of net.ip4.ip_forward
```
net.ipv4.ip_forward=1
```
Save the changes to the file.
With the certificates created and the initial server configuration created we can now start the VPN service.
```
service openvpn@server start
```
To see if the OpenVPN service started successfully:
```
service openvpn@server status
```
Now to test out the client, we need to copy client certificate, key, and ca certificate to the device connecting. OpenVPN provides a copy of this client.conf on their GitHub page. See link. In the client file edit, at the very least the following lines to make sure you are able to connect. I have listed my client file here without the comments for brevity.
https://raw.githubusercontent.com/OpenVPN/openvpn/master/sample/sample-config-files/client.conf
Location of client certificate, key and ca certificate: `/etc/openvpn/easy-rsa/keys/`
```
client
dev tun
proto udp
remote sykes 443
resolv-retry infinite
nobind
persist-key
persist-tun
ca /Users/bbogaert/sykes-ca.crt
cert /Users/bbogaert/bbogaert.crt
key /Users/bbogaert/bbogaert.key
comp-lzo
verb 3
```

Once your client file has been created, test the connection with your OpenVPN client. On my Mac I use Viscosity. Another client for mac that could be used is Tunnelblick. Both of these clients I find easier to use then the command line prompt.
