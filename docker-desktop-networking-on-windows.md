

# Docker networking on Windows

Note although the title is “networking on Windows” Mac is very similar.

The following diagram shows the network components in Docker Desktop and shows the data flows for



1. `docker pull`: how the Docker API call reaches the Docker Engine and how the Docker Engine downloads data from Docker Hub or other approved registry.
2. `apt-get update`: general container Internet access
3. `docker run -p`: how ports exposed on containers are accessed


![drawing](./images/image4.png)

Note that



1. The Host processes in the diagram all run as the developer, none are privileged. A privileged process` com.docker.service.exe` (not pictured) provides APIs for starting/stopping/resizing/installing/uninstalling the DockerDesktopVM.
2. The VM processes are the same for both WSL 2 (running inside the “docker-desktop” distro) and Hyper-V (running inside the “DockerDesktopVM”)
3. IDE integrations can use either the docker CLI or the underlying [Docker API](https://docs.docker.com/engine/api/)
4. The Dashboard and Tray use the Docker API and Desktop-specific APIs exposed by the com.docker.backend.exe process.
5. Host/VM communication uses `AF_VSOCK` hypervisor sockets (shared memory). It does not use Hyper-V network switches or network interfaces. All host networking is performed using normal TCP/IP sockets from the `com.docker.vpnkit.exe` and `com.docker.backend.exe` processes.


# Internet connectivity

Consider a Linux container making a connection to the Internet:


![image info](./images/image2.png)


1. The Linux container has an internal IP address associated with a “docker network” (see “docker network ls” “docker network create” and “docker network inspect”.
2. The container’s eth0 has a point-to-point network connection to a Linux bridge (“docker0”)
3. The traffic is sent using NAT via a “tap” device labeled “eth0” with internal IP 192.168.65.3 (see the setting in the Docker Desktop UI)
4. The traffic is sent via shared memory (hypervisor sockets or “AF_VSOCK”) to the host. It is not sent using the native Hyper-V networking stack. This works around Hyper-V networking bugs and firewall issues which accidentally block the traffic.
5. The traffic is received by the “com.docker.vpnkit.exe” or “com.docker.backend.exe” (depending on precise version and configuration) process and decoded.
6. “Com.docker.vpnkit.exe” or “com.docker.backend.exe” creates new TCP/IP connections by calling the Win32 network API (the same APIs used by other Windows applications) and proxies data to and from the VM. This means that outgoing traffic originates from the “com.docker.vpnkit.exe” or the “com.docker.backend.exe” process from the Windows point of view. So if the Windows firewall permits traffic from this process, the container will be able to contact the Internet.


# Exposing ports

Consider a web-browser accessing a Linux container via an exposed port (e.g. `docker run -p 80:80 nginx`)


![drawing](./images/image3.png)



1. The developer runs `docker run -p 80:80 nginx`
2. `com.docker.backend.exe` listens on 0.0.0.0:80
3. A web-browser connects to `<ip>:80`
4. `com.docker.backend.exe` accepts the connection
5. `com.docker.backend.exe` connects to a Windows named pipe created by `vpnkit-bridge.exe`
6. `vpnkit-bridge.exe` forwards from the Windows named pipe to a Unix domain socket in the Linux VM
7. `vpnkit-forwarder` accepts the Unix domain socket connection in the VM and forwards to the container’s internal TCP port 172.17.0.2:80

Note that the `docker run` command defaults to listening on all interfaces but the developer can specify a particular IP e.g. 127.0.0.1, or a WiFi adapter. Host firewall rules could be attached to the `com.docker.backend.exe` process.

Exclusions



1. This document only describes networking for Linux containers
2. Kubernetes allocates internal IP addresses inside the VM / docker-desktop distro


# HTTP(S) proxies

Docker Desktop supports using the user’s default upstream proxy and supports overrides via the Settings / Resources / Proxies UI.

Consider a `docker pull alpine:latest` in Docker Desktop with an HTTP proxy configured:


![drawing](./images/image1.png)

The Docker engine connects to an internal HTTP proxy on `http.docker.internal:3128` and writes an HTTP request:


```
HTTP CONNECT registry-1.docker.io:443 HTTP/1.1
Host: registry-1.docker.io:443
```


If there is no upstream HTTPS proxy, then `com.docker.backend.exe` will connect directly to the upstream server.

If there is an upstream HTTPS proxy URL configured e.g. [https://username:password@upstream.proxy:port](http://username:password@upstream.proxy:port) then the internal HTTP proxy will connect to `upstream.proxy:port` and write an HTTP request:


```
HTTP CONNECT registry-1.docker.io:443 HTTP/1.1
Host: registry-1.docker.io:443
Proxy-Authorization: basic <base64 encoded username:password>
```


Assuming the authentication is accepted, the upstream proxy will connect to Docker Hub and the image pull will succeed.


## Limitations



1. CLI plugins (e.g. “docker scan”) require separate proxy configurations, via the HTTP_PROXY, HTTPS_PROXY and NO_PROXY environment variables.
2. The Docker internal HTTP proxy only supports basic authentication and not (for example) protocols like NTLM. A workaround for this is to run another HTTP proxy on localhost which forwards to the authenticated proxy and point Docker Desktop at localhost. Note this also means it’s important to use a https:// proxy URL to protect proxy credentials on the network.