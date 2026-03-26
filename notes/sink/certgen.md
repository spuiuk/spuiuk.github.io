# CA & Certificate generation for use with Cephfs-smb

## Create Cetificates

### Generate private key for CA
```
openssl genrsa -out ca.key 2048
```

### Generate CA certificate
```
openssl req -x509 -noenc -key ca.key -days 1826 -out ca.crt
```
- x509: Generates a certificate insted of a request.
- noenc: Do not encrypt.
- key: The private key to use. Created int he earlier step
- sha256:
- days: number of days valid for
- out: output filename for the cert file. 

### Generate a certificate signing request
```
openssl req -new -noenc -out mycephfs11.csr -newkey rsa:2048 -keyout mycephfs11.key -addext "subjectAltName = DNS:mycephfs11.domain1.sink.test,IP:192.168.145.11"

```
- new: new certificate
- noenc: Do not encrypt private key
- out: output to file name
- newkey rsa:4096: new private key to be generated
- keyout: the filename to write the key to.
- addext "subjectAltName = DNS:mycephfs11.domain1.sink.test,IP:192.168.145.11": Add the Subject Alternative Name extension

### Sign the certificate
```
openssl x509 -req -in mycephfs11.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out mycephfs11.crt -days 730 -copy_extensions copy
```
- req: It is a request
- in: input csr file
- CA: The ca certificate
- CAkey: The private key for the CA
- CAcreateserial: create a serial file to keep track of serial nums used. Use -CAserial ca.srl for next calls.
- out: output crt file.
- days: days the cert is valid for
- copy_extensions copy: Copy over all the extensions from the certificate signing request

### Verify the signed certificate
```
openssl x509 -in mycephfs11.crt -noout -text
```

You should see a section similar to the following lines which indicate thatthe Subject Alternative Name is set for the certificate.
```
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:mycephfs11.domain1.sink.test, IP Address:192.168.145.11
            X509v3 Subject Key Identifier: 
                F8:B6:CD:85:19:55:7C:8E:63:63:5F:96:E5:7E:31:C2:34:1E:17:82
            X509v3 Authority Key Identifier: 
                8B:37:E1:9C:2E:5F:1A:66:F7:27:EC:11:13:D5:AA:8A:D2:2D:43:89
```

## Verify Certificates

### Copy over CA certificate

Copy the ca.crt file to /etc/pki/ and update ca-trust
```
cp ssl/ca/ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
```

### Verify certificate
```
openssl verify ssl/mycephfs11.crt
```
