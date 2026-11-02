# Telnet Setup Guide for Linux

## 1. Introduction

Telnet is a command-line network protocol used to connect to remote systems and test TCP port connectivity.

This guide covers:

1. Installing the Telnet client.
2. Configuring the Telnet server.
3. Starting and enabling the service.
4. Connecting from a Telnet client.
5. Testing network ports.
6. Troubleshooting common issues.

> **Security Note:** Telnet transmits usernames, passwords, and data in plain text. Use SSH instead of Telnet for secure remote administration. Telnet is best suited for lab environments and network troubleshooting.

---

## 2. Install Telnet Client

Choose your Linux distribution.

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install telnet -y
```

### RHEL / CentOS / Rocky Linux / AlmaLinux

```bash
sudo dnf install telnet -y
```

### Older CentOS

```bash
sudo yum install telnet -y
```

### Arch Linux

```bash
sudo pacman -S inetutils
```

### Verify Installation

```bash
telnet
```

---

## 3. Connect to a Remote Server

### Syntax

```bash
telnet <IP_ADDRESS> <PORT>
```

### Example

```bash
telnet 192.168.1.100 23
```

If the connection succeeds, you should see a connection message or a Telnet login prompt.

---

## 4. Test Network Connectivity

Telnet can be used to check whether a remote TCP port is accessible.

### Test HTTP Port (80)

```bash
telnet google.com 80
```

### Test HTTPS Port (443)

```bash
telnet google.com 443
```

A successful connection indicates that the TCP connection was established. It does not verify that the application protocol is functioning correctly.

---

## 5. Configure Telnet Server (Incoming Connections)

If you want another Linux machine to connect to your Linux server using Telnet, you need to install and configure the Telnet server separately.

### Ubuntu / Debian

Install the Telnet server:

```bash
sudo apt update
sudo apt install inetutils-telnetd
```

Enable Telnet through inetd:

```bash
sudo update-inetd --enable telnet
```

Restart the inetd service:

```bash
sudo systemctl restart inetutils-inetd.service
```

Check service status:

```bash
sudo systemctl status inetutils-inetd
```

### RHEL / CentOS / Rocky Linux

Install the Telnet server:

```bash
sudo dnf install telnet-server -y
```

Enable and start the Telnet socket:

```bash
sudo systemctl enable --now telnet.socket
```

Check service status:

```bash
sudo systemctl status telnet.socket
```

> Note: Service configuration can differ depending on the Linux distribution and version.

---

## 6. Configure Firewall

Telnet uses TCP port 23 by default.

### Ubuntu / Debian (UFW)

Allow Telnet traffic:

```bash
sudo ufw allow 23/tcp
```

Check firewall status:

```bash
sudo ufw status
```

### RHEL / CentOS (Firewalld)

Allow Telnet traffic:

```bash
sudo firewall-cmd --permanent --add-port=23/tcp
```

Reload firewall:

```bash
sudo firewall-cmd --reload
```

Verify the allowed port:

```bash
sudo firewall-cmd --list-ports
```

> Security recommendation: Only allow port 23 from trusted lab or management IP addresses. Avoid exposing Telnet to public networks.

---

## 7. Verify Telnet Server

### Check Whether Port 23 Is Listening

```bash
sudo ss -tlnp | grep :23
```

Expected output resembles:

```text
LISTEN 0 10 0.0.0.0:23 0.0.0.0:*
```

### Test Local Connection

```bash
telnet localhost 23
```

### Test Connection From Another Linux Machine

```bash
telnet SERVER_IP 23
```

Replace `SERVER_IP` with the actual IP address of your Telnet server.

---

## 8. Troubleshooting

| Error | Possible Solution |
|---|---|
| `telnet: command not found` | Install the Telnet client |
| `Connection refused` | Check whether the Telnet server is running and listening on port 23 |
| `Connection timed out` | Check firewall rules and network connectivity |
| `No route to host` | Check routing and network configuration |
| `Login incorrect` | Check the server account and authentication configuration |

### Useful Diagnostic Commands

Check network interfaces:

```bash
ip addr
```

Check routing:

```bash
ip route
```

Test network connectivity:

```bash
ping SERVER_IP
```

Check listening ports:

```bash
sudo ss -tlnp
```

Check inetd service:

```bash
sudo systemctl status inetutils-inetd
```

Check system logs:

```bash
sudo journalctl -xe
```

---

## 9. Telnet vs SSH

| Feature | Telnet | SSH |
|---|---|---|
| Default Port | 23 | 22 |
| Encryption | No | Yes |
| Secure Login | No | Yes |
| Recommended for Production | No | Yes |

For secure remote administration, use SSH.

### SSH Connection Example

```bash
ssh username@SERVER_IP
```

---

## 10. Quick Reference Commands

| Operation | Command |
|---|---|
| Install Telnet on Ubuntu | `sudo apt install telnet -y` |
| Install Telnet on RHEL | `sudo dnf install telnet -y` |
| Connect to remote server | `telnet SERVER_IP 23` |
| Check port 23 | `sudo ss -tlnp \| grep :23` |
| Allow port 23 (UFW) | `sudo ufw allow 23/tcp` |
| Allow port 23 (Firewalld) | `sudo firewall-cmd --permanent --add-port=23/tcp` |
| Test localhost | `telnet localhost 23` |
| Check network interfaces | `ip addr` |
| Check routing | `ip route` |

---

## 11. Conclusion

Telnet can be installed and configured on Linux for basic network connectivity testing and controlled lab environments.

For production systems and remote server administration, SSH is strongly recommended because it provides encrypted communication and secure authentication.

**Default Telnet Port:** TCP 23

**Default SSH Port:** TCP 22

