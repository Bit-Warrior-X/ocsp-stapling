# Own OCSP-Stapling system
OCSP stapling is a valuable feature for improving the performance, privacy, and reliability of SSL/TLS connections. By configuring your web server to use OCSP stapling, you can ensure that clients can quickly and securely verify the revocation status of your server’s certificate without directly contacting the CA.

# Follow below step
## 1. Make necearry directories and files
```bash
mkdir -p demoCA/newcerts
mkdir demoCA/private
touch demoCA/index.txt
echo 1000 > demoCA/serial
echo 1000 > demoCA/crlnumber
```

## 2. Create openssl.cnf
```bash
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./demoCA
database          = $dir/index.txt
new_certs_dir     = $dir/newcerts
certificate       = $dir/cacert.pem
serial            = $dir/serial
crlnumber         = $dir/crlnumber
crl               = $dir/crl.pem
private_key       = $dir/private/cakey.pem
default_days      = 365
default_crl_days  = 30
default_md        = sha256
policy            = policy_anything

[ policy_anything ]
countryName            = optional
stateOrProvinceName    = optional
localityName           = optional
organizationName       = optional
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

[ v3_intermediate_ca ]
basicConstraints = CA:TRUE, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
authorityKeyIdentifier = keyid:always,issuer

[ v3_OCSP ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, nonRepudiation
extendedKeyUsage = critical, OCSPSigning
```

## 3. Create Root CA
```bash
openssl genpkey -algorithm RSA -out demoCA/private/cakey.pem
openssl req -x509 -new -key demoCA/private/cakey.pem -sha256 -days 3650 -out demoCA/cacert.pem -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=RootCA"
```

## 4. Create the Intermediate CA
```bash
openssl genpkey -algorithm RSA -out demoCA/private/intermediatekey.pem
openssl req -new -key demoCA/private/intermediatekey.pem -out demoCA/intermediate.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=IntermediateCA"
openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 -in demoCA/intermediate.csr -out demoCA/intermediate.pem -batch
```

## 5. Create OCSP CA
```bash
openssl genpkey -algorithm RSA -out demoCA/private/ocspkey.pem
openssl req -new -key demoCA/private/ocspkey.pem -out demoCA/ocsp.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=OCSP Responder"
# Sign the OCSP responder certificate using the Intermediate CA:
openssl ca -config openssl.cnf -extensions v3_OCSP -days 3650 -in demoCA/ocsp.csr -out demoCA/ocsp.pem -cert demoCA/intermediate.pem -keyfile demoCA/private/intermediatekey.pem -batch
```

## 6. Create Server Certificate
```bash
openssl genpkey -algorithm RSA -out large-test.mncdn.com.key
openssl req -new -key large-test.mncdn.com.key -out large-test.mncdn.com.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=large-test.mncdn.com"
#Sign the server certificate using the Intermediate CA:
openssl ca -config openssl.cnf -in large-test.mncdn.com.csr -out large-test.mncdn.com.crt -cert demoCA/intermediate.pem -keyfile demoCA/private/intermediatekey.pem -batch
```

## 7. Combine the Certificates into chain.pem
```bash
cat demoCA/intermediate.pem demoCA/cacert.pem > chain.pem
# Combine the Certificate information into fullchain.pem
cat large-test.mncdn.com.crt demoCA/intermediate.pem demoCA/cacert.pem > fullchain.pem
```

## 8. Run OCSP server
```bash
openssl ocsp -port 2560 -text -index demoCA/index.txt -CAfile chain.pem -rkey demoCA/private/ocspkey.pem -rsigner demoCA/ocsp.pem -CA chain.pem
```

## 9. TEST OCSP server running correctly
```bash
openssl ocsp -CAfile chain.pem -url http://localhost:2560 -resp_text -issuer demoCA/intermediate.pem -cert large-test.mncdn.com.crt
```

## 10. Test config
```bash
server {
        listen 31.3.0.21:443 ssl;

        server_name large-test.mncdn.com;

        ssl_certificate /etc/ssl/ocsp/fullchain.pem;
        ssl_certificate_key /etc/ssl/ocsp/demoCA/large-test.mncdn.com.key;
        ssl_trusted_certificate /etc/ssl/ocsp/chain.pem;

        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_stapling_responder http://localhost:2560;

        # Other SSL configurations
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA;
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;

        location / {
                root /usr/share/nginx/html;
                index index.html;
        }
}
```

### Optional Generate ocsp file from ocsp server
```bash
openssl ocsp -noverify -issuer /etc/ssl/ocsp/demoCA/intermediate.pem -cert /etc/ssl/ocsp/large-test.mncdn.com.crt -url http://31.3.0.21:2560 -respout /etc/ssl/ocsp/ocsp_response.der
```
Through this, you can use *ssl_stapling_file* directive instead of *ssl_stapling_responder*. Something like below:
```bash
ssl_stapling_file /etc/ssl/ocsp/ocsp_response.der;
```

### Check OCSP is working correctly
```bash
openssl s_client -connect large-test.mncdn.com:443 -status
```

## 11. Get OCSP information from certificate
```bash
openssl x509 -in /path/to/your_certificate.crt -noout -text | grep -A 4 "Authority Information Access"
```

