---
title: Configuring Docker Daemon for Remote Execution
tags:
  - docker
  - daemon
  - ssl
---
For using docker image as a remote python interpreter, it is required to setup daemon and allow access to it remotely.

### CA, server, and client keys with openssl
You can find instruction for certificate in [Official Docker Document](https://docs.docker.com/engine/security/https/) and [Guide by Eric Draken](https://ericdraken.com/phpstorm-docker-daemon-https-aws/).
```bash
$ cd ~/.docker
# Generate CA private key
$ openssl genrsa -aes256 -out ca-key.pem 4096
# Generate CA public certificate
$ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
# Generate server private key
$ openssl genrsa -out server-key.pem 4096
# Generate server public certificate
$ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
# Allow IP / Server authentication
$ echo "subjectAltName = DNS:$HOST,IP:0.0.0.0" > extfile.cnf
$ echo "extendedKeyUsage = serverAuth" >> extfile.cnf
# Sign public server certificate
$ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf
```
In case you experience error looking for .rnd file from openssl when generating certificate, open /etc/ssl/openssl.cnf and remove randfile lines. [Solution](https://github.com/openssl/openssl/issues/7754#issuecomment-444063355)
```bash
$ sudo openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
Enter pass phrase for ca-key.pem:
Cant load /home/USER/.rnd into RNG 139862534840768:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/home/USER/.rnd
$ vi /etc/ssl/openssl.cnf
```

Now it's time to make client certificate in remote server (we'll transfer it later).
```bash
# Generate client private key
$ openssl genrsa -out client-key.pem 4096
# Generate client public certificate
$ openssl req -subj "/CN=$HOST" -new -key client-key.pem -out client.csr
# Client authentication
$ echo "extendedKeyUsage = serverAuth,clientAuth" > extfile-client.cnf
# Sign public client certificate
$ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem \
  -CAkey ca-key.pem -CAcreateserial -out client-cert.pem -extfile extfile-client.cnf
```
Then, clean the folder.
```bash
$ rm -v *.srl *.csr *.cnf
$ chmod -v 0400 ca-key.pem client-key.pem server-key.pem
$ chmod -v 0444 ca.pem client-cert.pem server-cert.pem
$ sudo mkdir -p /etc/docker/certs
$ sudo mv -t /etc/docker/certs *.pem
$ sudo chown root:root /etc/docker/certs/*
```
When it is done, copy ca.pem, client-cert.pem, and client-key.pem to the client machine and rename them to ca, cert, key, respectively.

### Docker daemon setting
```bash
$ sudo vi /etc/docker/daemon.json
{
    "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
    "tls": true,
    "tlsverify": true,
    "tlscacert": "/etc/docker/certs/ca.pem",
    "tlscert": "/etc/docker/certs/server-cert.pem",
    "tlskey": "/etc/docker/certs/server-key.pem"
}
$ sudo service docker restart
Job for docker.service failed because the control process exited with error code.
See "systemctl status docker.service" and "journalctl -xe" for details.
```
This is beacuse of conflict between systemctl and daemon.json. We need to configure systemctl for restarting.
[Reference](https://gist.github.com/ivan-pinatti/6ad05557e526f1f32ca357d15139df83)
```bash
$ sudo vi /etc/systemd/system/docker.service.d/docker.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
$ sudo systemctl daemon-reload
$ sudo service docker restart
```
Now it restarts properly. You can check it by grabbing port 2376 on netstat and trial connection. Beware the firewall(ufw) is open.
```bash
$ sudo netstat -tlpn | grep ":2376"
$ sudo docker --tlsverify \
  --tlscacert=/etc/docker/certs/ca.pem \
  --tlscert=/etc/docker/certs/client-cert.pem \
  --tlskey=/etc/docker/certs/client-key.pem \
  -H=0.0.0.0:2376 version
```
