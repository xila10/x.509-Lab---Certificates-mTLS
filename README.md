# x.509-Lab - Certificates-mTLS
A lab highlighting the importances of safe and verifiable communication and information exchange, using mTLS.

# Objectives:
- Setup a X.509 CA using the "easy-rsa" utility (version >3)
- Generate and sign a "server certificate" for the web server
- Generate and sign a "client certificate" for the client script
- Configure web server to support HTTPS, trust the CA and require client certificate authentication
- Configure client script to use HTTPS, trust the CA and authenticate using the client
certificate

# Documentation:
**Lab report/documentation:**
- Description of which programs/commands were used to generate certificates
- Documentation of changes to the client script and server configuration
- Demonstration of how the implemented changes improve security of the service
("before and after")

# 1. Creating Certificate Authority (CA)
Copy easy-rsa to home:
- cp -r /usr/share/easy-rsa ~/easy-rsa-lab
- cd ~/easy-rsa-lab
Keeping it cleaner.

Initiate PKI:
Location: ~/easy-rsa-lab
Command: ./easyrsa init-pki

**Result:**
This creates the katalogue: pki/ (pki = public key infrastructure)
**Building CA:**
Command: ./easyrsa build-ca

**Promts me to:**
- Enter a password
- Enter Common Name (CN)

**Purpose:**
- Used to sign and verify all certificates in the lab.

**It signs:**
- server certificates (nginx)
- client certificates (curl)

**It is used by:**
- the client to trust the server
- the server to verify the client (mTLS)

**Result of:** *./easyrsa build-ca:*
"No Easy-RSA 'vars' configuration file exists!"

- In modern Easy-RSA versions, it is not always necessary anymore, but the script still
warns if it is missing.
- Passphrase: "hello123"
- Choosing CA: "x509_Lab" - The name contained within the certificate.

**Result:**
- /home/vagrant/easy-rsa-lab/pki/ca.crt - This is the name of the actual file.

**We now have:**
- pki/ca.crt → public CA (to be shared with client + server)
- pki/private/ca.key → private CA (should never be shared)

**Copying ca.crt to each catalogue:**
- cp pki/ca.crt /vagrant/x509_tls/server_share/
- cp pki/ca.crt /vagrant/x509_tls/client_share/

# 2. Creating server certificate
**Creating server key + CSR:**
- ./easyrsa gen-req sensitive-web-server.example.test nopass


**Creates:**
- pki/private/sensitive-web-server.example.test.key
- pki/reqs/sensitive-web-server.example.test.req (this is the CSR, will be used to create the cert by the CA)

**Asked to name cert och key:****
- Has to match webserver-URL -> "sensitive-web-server.example.test"

**Signing server certificate with my CA:**
- ./easyrsa sign-req server sensitive-web-server.example.test

'yes' to confirm.

**We now have:**
- pki/issued/sensitive-web-server.example.test.crt
- pki/private/sensitive-web-server.example.test.key

**Copying to server_share:**
- cp pki/private/sensitive-web-server.example.test.key /vagrant/x509_tls/server_share
- cp pki/issued/sensitive-web-server.example.test.crt /vagrant/x509_tls/server_share

# 3. Creating client certificate
**Creating client key + CSR:**
- cd ~/easy-rsa-lab
- ./easyrsa gen-req client1 nopass

**Choosing a name:** "lab_klient"
- This name is how the client will be identified – like writing "Alice" or "John" etc.
- ---> Common Name (CN) withing the certificate.

**client1** is used within file-/certificate names.

**Creating:**
- req: /home/vagrant/easy-rsa-lab/pki/reqs/client1.req
- key: /home/vagrant/easy-rsa-lab/pki/private/client1.key

**Signing client certificate:**
- ./easyrsa sign-req client client1

**Signed certificate:** /home/vagrant/easy-rsa-lab/pki/issued/client1.crt
‘klient_lab’ identifies with ‘client1’ related files.

**We now have the following:**
- pki/issued/client1.crt
- pki/private/client1.key

**Copying to client_share:**
- cp pki/private/client1.key /vagrant/x509_tls/client_share/
- cp pki/issued/client1.crt /vagrant/x509_tls/client_share/

**The client now has certificate which is:**
- signed by the CA
- can be used to authenticate the client (mTLS)

# 4. Configuring the webserver (nginx)

**Entering:**
- /vagrant/x509_tls/server.conf

**Changing from HTTP to HTTPS:**
- ’listen 8080’; -> ’listen 443 ssl’; (standard port for https + prot. TLS ‘Transport Layer Security’.

**Adding server certificate + key:**
In the block: server { ... };
- ssl_certificate /share/sensitive-web-server.example.test.crt;
- ssl_certificate_key /share/sensitive-web-server.example.test.key;

**And:**
- ssl_client_certificate /share/ca.crt;
- This: trusting specifik CA to validate certificates.

**Finaly:**
- ssl_verify_client on;
- demands a client certificate.

**Results:**
The server now:
- uses HTTPS (TLS)
- presents the server certificate
- demands a client certificate to authenticate.
- verifies the client certificate using its local CA trust store (ca.crt).

# Tests:
Restarting the stack, building anew: docker compose up -d –build.
Will be using curl to test functions and demonstrate the differences in in accessibility and security.

**Entering client container:** docker compose exec client bash

## Test #1: "HTTP port: 8080 failing"

Opens an interactive shell within the container.
Command: "curl http://sensitive-web-server.example.test:8080"

**Yields:**

Summary:
- Connection is denied.
- The server is no longer listening on port 8080.
- The server is no longer exposed via insecure HTTP.
- (For HTTPS to function → a client certificate is required)
-T his is an infrastructure/configuration control, not a security control.

## Test #2: "connecting with proper authentication"

Command: "curl --cert /share/client1.crt --key /share/client1.key --cacert /share/ca.crt
https://sensitive-web-server.example.test/"

Greeted with server´s response; The connection attempt succeeds and we receive the
random generator’s response. **Success!**

**The process:**
The client certificate (.crt) is sent to the server. The private key (.key) is never shared,
it stays localy. The private key is used to sign parts of the TLS handshake, the server
uses the public key in the certificate to verify the signature - this proves that the client
actually owns the certificate.

Private keys are never transmitted — they are only used locally to prove identity.

**Summary:**
- TLS is working.
- The server presents a valid certificate.
- The client is authenticated via the CA (mTLS).
- Access is granted only with valid authentication.
- This is a security control (authentication + encryption).

## Test #3: "HTTP/HTTPS + missing certificate"

Command: "curl http://sensitive-web-server.example.test/"
- Reacting as expected; "http, failed to connect”.

Command: "curl https://sensitive-web-server.example.test/"
- It shows that the absence of a valid SSL certificate is now the issue; HTTPS is being accepted.

**Principals:**
The server presents its certificate, but the client lacks a trusted CA to verify it and
therefore cannot validate the server’s identity. In mTLS, the server also requires a client
certificate, which it verifies against a trusted CA. Otherwise, access is denied. This is
why we need to configure a trusted CA in server.conf (+server_share) as well as in the
client_share folder – giving them a shared “root of trust”.

So:
- Server certificate → verified by the client using a CA.
- Client certificate → verified by the server using the same CA.
- Both sides require local ‘trust anchors’.
- Neither side automatically trusts the other.

## Test #4: ‘Certs not matching’
I have created a new set of certificate + key, not signed by the trusted CA;
demonstrating the outcome of non-matching certificates.
Command: curl --cert /share/badclient.crt --key /share/badclient.key --cacert
/share/ca.crt https://sensitive-web-server.example.test/

**Result:**
"400 SSL certificate error", "400 Bad Request", "SSL certificate error" etc.

Trying to connect using a different key and certificate → the CA trusted by the server has NOT been used to validate them, resulting in a “400 SSL… / Bad Request” response →
NGINX has rejected the TLS handshake.

**Right cert. + key:**
We gain access!

**Summary of tests:**
The server is no longer exposed via HTTP (port 8080). Access to the service is provided
via HTTPS and requires a valid client certificate signed by the trusted CA. Without a
correct certificate, access is denied. The server trusted CA must validate the client
certificate; otherwise, access is rejected.

If the client does not present any certificate, or if the certificate is signed by a different CA, the validation fails and the connection is denied.

## Test #5: "HTTP vs HTTPS" traffic’
Using tcpdump sniff trafic:
Command: sudo tcpdump -i lab-x509_tls port 443 -A

Gives:
This is encrypted HTTPS-traffic.

**Without HTTPS:**

**Terminal 1:**
- Opens a new terminal, entering an interactive client shell.
- Command: "curl http://sensitive-web-server.example.test:8080"
- Changing server och client to once again listen and send traffic on http port:8080.

**Terminal 2:**
- Command: sudo tcpdump -i lab-x509_tls port 8080 -A

**Catching the curl-request:**

Pure plain text, unsafe HTTP-traffic!

# Running docker-compose:
Wrong client settings:

We have demonstrated this using curl so far; this is what it looks like when the
application runs without a correctly configured client:
"ERROR: Failed to request a top-secret quote from http://sensitive-web-server.example.test:8080/"
Not using HTTPS yet, trying to connect to port: 8080.

**Opening the skrip: client.sh**

**Changing:**
- TARGET_URL="http://{TARGET_SERVER_ADDRESS}:8080/"
To:
- TARGET_URL="https://{TARGET_SERVER_ADDRESS}/"

**Adding:**
````
curl \
--fail --silent --show-error \
--connect-timeout 3 \
--cert /share/client1.crt \
--key /share/client1.key \
--cacert /share/ca.crt \
"${TARGET_URL}"
````
- Success! With the correct authentication information, the client is granted access.

# Wrapping up:
- Demonstrated the core principles of secure communication using TLS and mutual TLS
through practical tests with curl and a webserver.
- Compared HTTP and HTTPS to illustrate the difference between unencrypted and
encrypted communication. When HTTP was disabled, access via port 8080 was no
longer possible; demonstrating service exposure control through proper configuration.
- Introduced mTLS, where the server and the client present a certificate during
connection establishment. They share the same ‘root of trust’; certificates are signed
by the same CA. Also demonstrating a certificate and CA mismatch.
- When both sides are correctly configured the client will trust the server, the server will
trust the client. They both have access to a matching CA, which has signed their
respective certificates.

**Author: Niklas Norberg, ITHS25GS**
