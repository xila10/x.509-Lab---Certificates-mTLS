# x.509-Lab---Certificates-mTLS

A lab highlighting the importances of safe and verifiable communication and information exchange, using *mTLS*.

---

## Objectives

### Mandatory ("G")

- Setup a X.509 CA using the *"easy-rsa"* utility (version >3)  
- Generate and sign a *"server certificate"* for the web server  
- Generate and sign a *"client certificate"* for the client script  
- Configure web server to support *HTTPS*, trust the CA and require client certificate authentication  
- Configure client script to use *HTTPS*, trust the CA and authenticate using the client certificate  

---

## Documentation

Each student should submit a lab report containing **at least** the following information ("G"):

- Description of which programs/commands were used to generate certificates  
- Documentation of changes to the client script and server configuration  
- Demonstration of how the implemented changes improve security ("before and after")  

---

## 1. Creating Certificate Authority (CA)

### Copy easy-rsa

```bash
cp -r /usr/share/easy-rsa ~/easy-rsa-lab
cd ~/easy-rsa-lab
```
-Keeping it cleaner.

Initiate PKI

Location: ~/easy-rsa-lab
Command:

./easyrsa init-pki

Result:
Creates catalogue: pki/ (public key infrastructure)

Build CA
./easyrsa build-ca

Prompts:

Enter password
Enter Common Name (CN)

Purpose:

Used to sign and verify all certificates in the lab

It signs:

server certificates (nginx)
client certificates (curl)

It is used by:

client → trust server
server → verify client (mTLS)

Result:

"No Easy-RSA 'vars' configuration file exists!"
Modern versions don’t always need it
Passphrase: "hello123"
CA name: "x509_Lab"

We now have:

pki/ca.crt → public CA (shared)
pki/private/ca.key → private CA (never shared)

Copy CA:

cp pki/ca.crt /vagrant/x509_tls/server_share/
cp pki/ca.crt /vagrant/x509_tls/client_share/

2. Creating server certificate
Generate key + CSR
./easyrsa gen-req sensitive-web-server.example.test nopass

Creates:

pki/private/sensitive-web-server.example.test.key
pki/reqs/sensitive-web-server.example.test.req (CSR)

Must match URL:
sensitive-web-server.example.test

Sign certificate
./easyrsa sign-req server sensitive-web-server.example.test

We now have:

pki/issued/... .crt
pki/private/... .key

Copy:

cp pki/private/sensitive-web-server.example.test.key /vagrant/x509_tls/server_share
cp pki/issued/sensitive-web-server.example.test.crt /vagrant/x509_tls/server_share

3. Creating client certificate
cd ~/easy-rsa-lab
./easyrsa gen-req client1 nopass
CN: "lab_klient"
Identity of client

Creates:

.req → CSR
.key → private key

Sign:

./easyrsa sign-req client client1

Copy:

cp pki/private/client1.key /vagrant/x509_tls/client_share/
cp pki/issued/client1.crt /vagrant/x509_tls/client_share/

Client certificate:

signed by CA
used for authentication (mTLS)

Configuring the webserver (nginx)

File:
/vagrant/x509_tls/server.conf

Change:

listen 8080; → listen 443 ssl;

Add:

ssl_certificate     /share/sensitive-web-server.example.test.crt;
ssl_certificate_key /share/sensitive-web-server.example.test.key;

ssl_client_certificate /share/ca.crt;
ssl_verify_client on;
Results

The server now:

uses HTTPS (TLS)
presents server certificate
requires client certificate
verifies using ca.crt

**Tests:**
Start environment
docker compose up -d --build
docker compose exec client bash

**Test #1: HTTP fails**
curl http://sensitive-web-server.example.test:8080

**Summary:**

Connection denied
No HTTP exposure
Config control (not security control)
Test #2: Valid mTLS
curl --cert /share/client1.crt \
     --key /share/client1.key \
     --cacert /share/ca.crt \
     https://sensitive-web-server.example.test/

Success!

TLS process
.crt → sent
.key → local only
key signs handshake
server verifies

Private keys are never transmitted.

Test #3: Missing certificate
curl https://sensitive-web-server.example.test/
TLS works
Auth fails
Test #4: Wrong certificate
curl --cert /share/badclient.crt \
     --key /share/badclient.key \
     --cacert /share/ca.crt \
     https://sensitive-web-server.example.test/

**Result:**

400 SSL certificate error
TLS handshake rejected
Test #5: HTTP vs HTTPS (tcpdump)
sudo tcpdump -i lab-x509_tls port 443 -A

→ Encrypted

sudo tcpdump -i lab-x509_tls port 8080 -A

→ Plaintext

Running docker-compose (client script)

**Error example:**

ERROR: Failed to request a top-secret quote from http://...
Fix client.sh
TARGET_URL="https://${TARGET_SERVER_ADDRESS}/"

Add:

curl \
--fail --silent --show-error \
--connect-timeout 3 \
--cert /share/client1.crt \
--key /share/client1.key \
--cacert /share/ca.crt \
"${TARGET_URL}"

Success!

**Wrapping up:**
Demonstrated TLS + mTLS
Compared HTTP vs HTTPS
Showed certificate validation
Demonstrated CA trust model
Verified correct vs incorrect authentication
