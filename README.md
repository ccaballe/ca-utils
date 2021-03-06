# ROOT CA


* Creating dir structure

```
mkdir my-root-ca
cd my-root-ca
mkdir certs db private newcerts crl
chmod 700 private
touch db/index.txt db/index.txt.attr
openssl rand -hex 16  > db/serial
echo 1001 > crl/crlnumber
wget https://raw.githubusercontent.com/ccaballe/ca-utils/master/root-ca.conf
```

* Edit the path in dir variable:

```# dir               = /path/to/my-root-ca```

* Creating root key

`openssl genrsa -aes256 -out private/stratio-ca.key.pem 4096`

* Inspect private key

`openssl rsa -in private/stratio-ca.key.pem -noout -text`


* Self signed certificate

`openssl req -config root-ca.conf -key private/stratio-ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/stratio-ca.cert.pem`

* Inspect public key

`openssl x509 -in certs/stratio-ca.cert.pem -noout -text`

* Issue CRL

`openssl ca -gencrl -config root-ca.conf -out stratio-ca.crl`

* Get CRL info

`openssl crl -inform PEM -text -noout -in stratio-ca.crl`


# Intermediate CA


* Directory Tree

```
mkdir my-intermediate-ca
cd my-intermediate-ca
mkdir db certs crl csr newcerts private
chmod 700 private
touch db/index.txt
openssl rand -hex 16  > db/serial
echo 1001 > crl/crlnumber

wget https://raw.githubusercontent.com/ccaballe/ca-utils/master/intermediate-ca.conf
```

* edit the path in dir to your my-intermediate-ca

`# dir               = /path/to/my-root-ca/my-intermediate-ca`

* Creating Intermediate Key

`openssl genrsa -aes256 -out private/numa-intermediate-ca.key.pem 4096`

* Use the intermediate key to create a certificate signing request (CSR). The details should generally match the root CA. The Common Name, however, must be different.

`openssl req -config intermediate-ca.conf -new -sha256 -key private/numa-intermediate-ca.key.pem -out csr/numa-intermediate-ca.csr.pem`

* Sign the certificate 

```
cd ..
openssl ca -config root-ca.conf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in my-intermediate-ca/csr/numa-intermediate-ca.csr.pem -out my-intermediate-ca/certs/numa-intermediate-ca.cert.pem

chmod 444 my-intermediate-ca/certs/numa-intermediate-ca.cert.pem
```

* Inspect new intermediate CA

`openssl x509 -noout -text -in my-intermediate-ca/certs/numa-intermediate-ca.cert.pem`

* Verify

`openssl verify -CAfile certs/stratio-ca.cert.pem my-intermediate-ca/certs/numa-intermediate-ca.cert.pem`

* Create chain

```
cat my-intermediate-ca/certs/numa-intermediate-ca.cert.pem certs/stratio-ca.cert.pem > my-intermediate-ca/certs/ca-chain.cert.pem
chmod 444 my-intermediate-ca/certs/ca-chain.cert.pem
```

# New certs

`cd my-intermediate-ca`

* Server Certificate

```
openssl req -new -newkey rsa:2048 -subj "/C=ES/O=Stratio Numa/CN=server00.dev.stratio.com" -keyout private/server00.key -out certs/server00.csr
openssl ca -config intermediate-ca.conf -in certs/server00.csr -out certs/server00.cert.pem -extensions server_cert
```

* Convert to keystore jks
```
openssl pkcs12 -export -inkey ../private/server00.key -in ./server00.cert.pem -out server00.p12
keytool -noprompt -importkeystore -srckeystore server00.p12 -srcstoretype PKCS12 -destkeystore server00.jks
```

* Add ca to truststore
```
keytool -import -noprompt -alias ca -keystore intermediate-truststore.jks -file numa-intermediate-ca.cert.pem
```


* Client Certificate

```
openssl req -new -newkey rsa:2048 -subj "/C=ES/O=Stratio Numa/CN=client00.dev.stratio.com" -keyout private/client00.key -out certs/client00.csr
openssl ca -config intermediate-ca.conf -in certs/client00.csr -out certs/client00.cert.pem -extensions usr_cert
```

* Read the new cert
`openssl x509 -in certs/client00.cert.pem -noout -text `

* Verify (with chain)
```
cd ../
openssl verify -CAfile my-intermediate-ca/certs/ca-chain.cert.pem my-intermediate-ca/certs/server00.cert.pem
```


