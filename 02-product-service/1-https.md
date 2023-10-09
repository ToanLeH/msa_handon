# Create Self-Signed certificate support HTTPS in ubuntu 20.04

- Typically, self-singed certificate is not working in Linux Environment. Especially the official guide from Microsoft [linux-certificate-problems](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl?view=aspnetcore-7.0&tabs=visual-studio%2Clinux-ubuntu#troubleshoot-certificate-problems-such-as-certificate-not-trusted). See section Ubuntu trust the certificate for service-to-service communication, Microsoft did mention the instruction don't work for some Ubuntu versions, such as 20.04. below are some workarounds steps for supporting ubuntu 20.04. In case you use other version, please skip this section

## Create folder **aspnet** via src folder
- Create folder aspnet via src folder
- Create folder https via aspnet folder (structure after this will be src/aspnet/https)

## Create **localhost.conf** file via **aspnet/https** folder
- Create **localhost.conf** file via **aspnet/https** folder and paste below content to it
``` conf
[req]
default_bits       = 2048
default_keyfile    = localhost.key
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_ca

[req_distinguished_name]
commonName                  = Common Name (e.g. server FQDN or YOUR name)
commonName_default          = localhost
commonName_max              = 64

[req_ext]
subjectAltName = @alt_names

[v3_ca]
subjectAltName = @alt_names
basicConstraints = critical, CA:false
keyUsage = keyCertSign, cRLSign, digitalSignature,keyEncipherment

[alt_names]
DNS.1   = localhost
DNS.2   = 127.0.0.1
DNS.3   = identity-service
DNS.4   = product-service

```

- Run the following command to check if OpenSSL is installed
``` bash
which openssl
```

- If OpenSSL is not installed, install OpenSSL with brew
``` bash
sudo apt install openssl -y
```

- In case this is the first time you create the certificate, you have to create the certificate store via ubuntu
``` bash
mkdir -p $HOME/.pki/nssdb
certutil -N -d $HOME/.pki/nssdb --empty-password
```

- Open new terminal via above conf path, Run the following 2 commands using openssl to create a self-signed certificate in Ubuntu Linux
``` bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -config localhost.conf -passin pass:MsaFundamental

sudo openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt
```

- Then, run the following certutil command to add the certificate to your trusted CA root store 
``` bash
certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n "localhost" -i localhost.crt
```

Note: in case above command return error "certutil: could not add certificate to token or database: SEC_ERROR_ADDING_CERT: Error adding certificate to database.", it's because you had added the same cert name before. Trying to remove the previous one with below command and run above command again
``` bash
#list existing certs in nssdb
certutil -L -d sql:$HOME/.pki/nssdb

#remove same name cert
certutil -D -n localhost -d sql:$HOME/.pki/nssdb

```

- Trust above certificate in Edge/Chrome
``` bash
mkdir -p ~/.mozilla/certificates
certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n localhost -i ./localhost.crt
certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n localhost -i ./localhost.crt

```

- Trust service to service invocation
``` bash
sudo cp localhost.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
```

- Grant full permission for this file
``` bash
chmod +X ./localhost.pfx
```