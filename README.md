# OpenVPN-Notes
**Notes** on _installing_, configuring and managing OpenVPN

OpenVPN is an Open Source solution for implementing a VPN server. In this article we will describe how to install and setup OpenVPN, as well as OpenVPN with 2 Factor Authentication. The setup described here is using the Ubuntu distribution of Linux. The initial server setup instructions have been adapted from Ubuntu’s official guide on setting up an OpenVPN server.

## Build the initial server
### Install OpenVPN on Ubuntu 16.04 using instructions from OpenVPN

Instructions: https://help.ubuntu.com/lts/serverguide/openvpn.html

```
# Become the root user
sudo -i
# Install the OpenVPN packaged and the easy-rsa package
apt-get install easy-rsa openvpn
```
#### Setup the Public Key Infrastructure

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
#### Configure the OpenVPN service
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

## Harden VPN Security by using PKCS #11
The great thing about OpenVPN is that it can be used with open standards such as PKCS. PKCS #11 is a standard for Cryptographic Token Interface. These devices can store certificates and keys such as the ones used by OpenVPN. In this part of the tutorial we will be using a Yubikey to store the certificate and connect to the VPN. Please, note, getting the certificate on the card needed quite a bit of hackery, but nonetheless, here is my process.
From the yubikey forum post one needs openvpn, opensc, and yubico-piv-tool or yubikey-piv-manager
Viscosity OpenVPN application:
https://www.sparklabs.com/viscosity/
OpenSc can be installed using homebrew.
http://brew.sh/
```
brew install opensc
```
OpenSc install location
```
/usr/local/Cellar/opensc/0.15.0/lib/opensc-pkcs11.so
```
Yubico piv tool
https://developers.yubico.com/yubico-piv-tool/Releases/yubico-piv-tool-1.4.0-mac.zip

Information on Yubikey 4 used for this guide:
https://www.yubico.com/products/yubikey-hardware/yubikey4/

References:
http://forum.yubico.com/viewtopic.php?f=26&t=2124

Using openssl export the client certificate, ca certificate, and key to one file
```
openssl pkcs12 -export -out bbogaert-cert-key.p12 -inkey bbogaert.key -in bbogaert.crt -certfile sykes-ca.crt -nodes
```



To successfully import the certificate onto my yubikey you may need to reset the pin and puk codes first.

https://developers.yubico.com/yubico-piv-tool/YubiKey_PIV_introduction.html

```
./yubico-piv-tool -a verify-pin -P 471112
./yubico-piv-tool -a verify-pin -P 471112
./yubico-piv-tool -a verify-pin -P 471112
./yubico-piv-tool -a verify-pin -P 471112
./yubico-piv-tool -a change-puk -P 471112 -N 6756789
./yubico-piv-tool -a change-puk -P 471112 -N 6756789
./yubico-piv-tool -a change-puk -P 471112 -N 6756789
./yubico-piv-tool -a change-puk -P 471112 -N 6756789
./yubico-piv-tool -a reset
```

After I reset the codes I was able to import the certificates onto the key.

Import the certificates using the yubico-piv-tool. Note, after the “-P” is where you would put your configured pin.
```
./yubico-piv-tool -s 9c -i ~/bbogaert-cert-key.p12 -K PKCS12 -a import-key -a import-cert -P 71234523
```

The next step is to use the OpenVPN program and the opensc library to find the identifier of the card and modify the client configuration file. Here I’m using the openvpn binary directly from within the Viscosity program and pointing to the opensc security library to find the ids of my yubikey. We will then put this information in the configuration file.

```
byronicle:bin bbogaert$ sudo /Applications/Viscosity.app/Contents/MacOS/openvpn --show-pkcs11-ids /usr/local/Cellar/opensc/0.15.0/lib/opensc-pkcs11.so
Password:

The following objects are available for use.
Each object shown below may be used as parameter to
--pkcs11-id option please remember to use single quote mark.

Certificate
       DN:             C=US, ST=CA, L=San Francisco, O=Wikimedia, OU=OIT, CN=bbogaert, name=Byron Bogaert, emailAddress=techsupport@wikimedia.org
       Serial:         02
       Serialized id:  piv_II/PKCS\x2315\x20emulated/00000000/PIV_II\x20\x28PIV\x20Card\x20Holder\x20pin\x29/02

```
Edit the client configuration file and add the following lines to the end:
```
pkcs11-providers /usr/local/Cellar/opensc/0.15.0/lib/opensc-pkcs11.so
pkcs11-id 'piv_II/PKCS\x2315\x20emulated/00000000/PIV_II\x20\x28PIV\x20Card\x20Holder\x20pin\x29/02'
```

The configuration file can then be imported into Viscosity. I have tried using tunnelblick but it does not work well with PKCS #11 providers. Also, when using viscosity “Allow unsafe OpenVPN commands to be used” needs to be checked. I’m not sure why the opensc-pkcs11.co is in a “unsafe” location, but if I could move it to a safe location I would. Any insight into amending this would be much appreciated.

Once, the configuration is in, connect to the vpn, and a window will appear asking for your pin to access the certificate on the device. After entering your pin, the program will pull your certificate off of the key and log you into the VPN.

## OpenVPN with Yubikey OTP and LDAP
OpenVPN can also be configured to use Yubikeys with OTP. When a person logs into the VPN they press the button on their Yubikey which is then authenticated against Yubikey’s server. Once this second passcode is authenticated the person can login.

In order to validate against the Yubico authentication database
Yubikey PAM Module: https://github.com/Yubico/yubico-pam

Register for Yubico API: https://upgrade.yubico.com/getapikey/

Buy a Yubikey: https://www.yubico.com/products/yubikey-hardware/

Be sure to get a Yubikey that can do OTP

To use PAM with OpenVPN we need to add a line to the OpenVPN configuration file. Use the OpenVPN PAM module, along with the location of the PAM configuration for OpenVPN:
```
plugin /usr/lib/openvpn/openvpn-plugin-auth-pam.so /etc/pam.d/openvpn
```
When using LDAP, store Yubico serial numbers by using an existing attribute for LDAP or add your own custom attribute. I found the “pager” attribute a good one to use for test purposes.

The OpenVPN file for PAM needs to “stacked” with another authentication method to insure that two-factor authentication. A mistake that could be made is that the Yubico PAM module is checking that the User’s password is valid. This is not the case. The Yubico PAM module will just evaluate that the OTP is valid for that particular key. This module will not verify the person’s LDAP password. To verify the person’s ldap password, one needs to install the LDAP module for PAM.

Article on PAM: https://www.digitalocean.com/community/tutorials/how-to-use-pam-to-configure-authentication-on-an-ubuntu-12-04-vps

The following PAM configuration works by first evaluating the password entered by the user. The Yubico PAM module will separate the password from the OTP, verify the OTP, then send the password to the next PAM module. Following the Yubico module, is the ldap module which will take the person’s password (use_first_pass) and evaluate this with LDAP.
Pam file for OpenVPN:
```
# The following is for authentication with Yubikey OTP
# Yubico documentation: https://github.com/Yubico/yubico-pam
# Check ldap for yubikey serial number with yubikey attribute and make sure
# this is required
auth  required  pam_yubico.so authfile=/etc/yubikey_mappings ldap_uri=ldaps://ldap.example.org capath=/etc/ssl/certs yubi_attr=yubikey id=12345 ldapdn=ou=people,dc=esample,dc=org ldap_filter=(cn=%u) ldap_bind_user=cn=admin,dc=example,dc=org ldap_bind_password=S0meP@ssword

# Make sure the person's ldap password is valid too. The previous PAM module for
# yubico will pass the password to this module for use
auth required pam_ldap.so use_first_pass
```

### How to “bypass2fa” by using pam_access
Pam_access allows pam to query for group membership and/or allow authentication for a particular host. In this case we will be using pam_access to check whether a person is a member of the bypass2fa group and allow them to be authenticated.

Install packages: libpam-ldap, nscd

References:

https://linux.die.net/man/8/pam_access

http://www.tldp.org/HOWTO/archived/LDAP-Implementation-HOWTO/pamnss.html

https://wiki.debian.org/LDAP/NSS
