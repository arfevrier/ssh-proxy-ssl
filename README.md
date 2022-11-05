# SSL/TLS Tunneling through HTTPS Proxy for SSH

Description and configuration to connect to SSH server, in case you can't directly connect using SSH (port 22) protocol. The idea is to use a proxy with HTTP CONNECT capability to transmit SSL/TLS which, does not encapsulate HTTP, but encapsulate SSH.

# Architecture I

![schema1](https://user-images.githubusercontent.com/23292338/200145871-265ec062-2eb4-4171-bedb-d5d78dcae614.png)

``` apache
# Apache2 Remote proxy
<VirtualHost *:443>
	ServerAdmin me@domain.fr
	Redirect permanent "/" "https://domain.fr/"
	
	ProxyRequests On
	AllowCONNECT 22 443

	<Proxy "*">
		AuthName "Proxy Server Authentification"
		AuthType Basic
		AuthBasicProvider file
		AuthUserFile "/var/auth/.htpasswd"
		Require valid-user
	</Proxy>
	
	SSLEngine on
	SSLCertificateFile		/etc/ssl/domain.fr/cert.pem
	SSLCertificateKeyFile	/etc/ssl/domain.fr/key.pem
	SSLCACertificateFile	/etc/ssl/domain.fr/fullchain.pem
</VirtualHost>
```


``` ssh-config
# Client
Host domain
    HostName domain.fr
    User user
    Port 22
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand proxytunnel -v -p 127.0.0.1:3128 -r domain.fr:443 -R username:password -X -d %h:%p
```

# Architecture II

![schema2](https://user-images.githubusercontent.com/23292338/200146083-2a67f5dc-bda9-4f85-89e3-dba579808004.png)

## Why *sslh* ?

*sslh* is just here to make valid HTTP response in case of HTTPS request on the port 443 of the server. This is optional. But without this, request on port 443 will result by weird HTTP/0.9 response from *SSHd*.

## Configuration

``` yaml
# Server side
version: "3"
services:
  sslh:
    image: "imartyn/sslh:alpine"
    network_mode: "host"
    restart: always
    command: -f --listen=127.0.0.1:8022 --ssh=146.59.238.203:22 --http=146.59.238.203:80
  stunnel:
    image: "dweomer/stunnel"
    network_mode: "host"
    restart: always
    volumes:
      - /etc/ssl/domain.fr/cert.pem:/etc/stunnel/stunnel.pem:ro
      - /etc/ssl/domain.fr/key.pem:/etc/stunnel/stunnel.key:ro
    environment:
      STUNNEL_SERVICE: ssh
      STUNNEL_ACCEPT: 146.59.238.203:443
      STUNNEL_CONNECT: 127.0.0.1:8022
```

``` ssh-config
# Client
Host domain.fr
    HostName domain.fr
    User user
    Port 443
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand proxytunnel -v -p 127.0.0.1:3128 -d %h:%p -e
```
