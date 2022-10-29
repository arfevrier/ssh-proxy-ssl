# SSL/TLS Tunneling through HTTPS Proxy for SSH

Description and configuration to connect to SSH server, in case you can't directly connect using SSH (port 22) protocol. The idea is to use a proxy with HTTP CONNECT capability to transmit SSL/TLS which, does not encapsulate HTTP, but encapsulate SSH.

## Why *sslh* ?

*sslh* is just here to make valid HTTP response in case of HTTPS request on the port 443 of the server. This is optional. But without this, request on port 443 will result by weird HTTP/0.9 response from *SSHd*.

## Architecture

SSH (Client) --> *proxytunnel* (Client) --> Squid (Proxy) --> *Stunnel* :443 (Server) --> *sslh* :8022 (Server) --> SSH :22 (Server)
____________________________________________________________________________________________________________--> HTTP :80 (Server)

* *proxytunnel* (Client) <--> Squid (Proxy): **HTTP Connect**
* *proxytunnel* (Client) <--> *Stunnel* :443 (Server): **SSL/TLS**
* SSH (Client) <--> *sslh* :8022 (Server): **SSH**

## Configuration

``` yaml
# Server
version: "3"
services:
  sslh:
    image: "imartyn/sslh:alpine"
    network_mode: "host"
    restart: always
    command: -f --listen=127.0.0.1:8022 --ssh=???????????:22 --http=???????????:80
  stunnel:
    image: "dweomer/stunnel"
    network_mode: "host"
    restart: always
    volumes:
      - /etc/ssl/??????????/cert.pem:/etc/stunnel/stunnel.pem:ro
      - /etc/ssl/??????????/key.pem:/etc/stunnel/stunnel.key:ro
    environment:
      STUNNEL_SERVICE: ssh
      STUNNEL_ACCEPT: ???inet_ip????:443
      STUNNEL_CONNECT: 127.0.0.1:8022
```

``` ssh-config
# Client
Host ????????
    HostName ???inet_ip????
    User user
    Port 443
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand proxytunnel -v -p 127.0.0.1:3128 -d %h:%p -e
    LocalForward 9999 127.0.0.1:22 #Even with ProxyCommand you can still forward port etc...
```
