# Creating a full chain 

## Root CA pk

```bash
openssl genrsa -out rootca.key.pem 4096
```

## Root CA cert pem
```bash
openssl req -x509 -new -nodes -key rootca.key.pem -sha256 -days 3650 \
  -subj "/CN=RootCA" \
  -out rootca.cert.pem \
  -extensions v3_ca \
  -config <(echo "
  [req]
  distinguished_name=req
  [v3_ca]
  basicConstraints=critical,CA:TRUE
  keyUsage=critical,keyCertSign,cRLSign
  subjectKeyIdentifier=hash
  ")
```

## Int CA pk
```bash
openssl genrsa -out intermediate.key.pem 4096
```

## csr int
```bash
openssl req -new -key intermediate.key.pem -sha256 -subj "/CN=IntermediateCA" \
  -out intermediate.csr.pem
```

## create the int pem
```bash
openssl x509 -req -in intermediate.csr.pem -CA rootca.cert.pem -CAkey rootca.key.pem \
  -CAcreateserial -out intermediate.cert.pem -days 1825 -sha256 \
  -extfile <(echo "
  basicConstraints=critical,CA:TRUE
  keyUsage=critical,keyCertSign,cRLSign
  subjectKeyIdentifier=hash
  authorityKeyIdentifier=keyid
  ")
```

## create server pk
```bash
openssl genrsa -out server.key.pem 2048
```

## csr for srv
```bash
openssl req -new -key server.key.pem -sha256 -subj "/CN=server.test.redhat.com" \
  -out server.csr.pem
```

## srver pem
```bash
openssl x509 -req -in server.csr.pem -CA intermediate.cert.pem -CAkey intermediate.key.pem \
  -CAcreateserial -out server.test.redhat.com.cert.pem -days 825 -sha256 \
  -extfile <(echo "
  basicConstraints=CA:FALSE
  keyUsage=digitalSignature,keyEncipherment
  extendedKeyUsage=serverAuth
  subjectKeyIdentifier=hash
  authorityKeyIdentifier=keyid
  ")
```