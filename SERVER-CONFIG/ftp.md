# FTP Server Installation and Usage on Linux

This guide covers installing and configuring **vsftpd**, creating an FTP
user, opening the firewall, connecting to the server, and transferring
files.

> **Security note:** Traditional FTP sends credentials and data without
> encryption. Use SFTP (over SSH) or configure FTPS with TLS for
> production or untrusted networks.

## 1. Install the FTP server

### Ubuntu / Debian

``` bash
sudo apt update
sudo apt install vsftpd -y
```

### CentOS / RHEL / Rocky Linux / AlmaLinux

``` bash
sudo dnf install vsftpd -y
```

## 2. Start and enable the service

``` bash
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
sudo systemctl status vsftpd
```

## 3. Configure vsftpd

Back up the configuration file, then edit it:

``` bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
sudo nano /etc/vsftpd.conf
```

Set or update these options:

``` ini
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
```

For a typical standalone IPv4 server, also ensure the following listener
settings are appropriate for your setup. Do not enable both `listen` and
`listen_ipv6` at the same time:

``` ini
listen=YES
listen_ipv6=NO
```

Save the file and restart vsftpd:

``` bash
sudo systemctl restart vsftpd
```

## 4. Create an FTP user

Create a Linux user and set its password:

``` bash
sudo adduser ftpuser
sudo passwd ftpuser
```

Use this account to log in to the FTP server. The account's home
directory is its default starting directory.

## 5. Open the firewall

### Ubuntu with UFW

``` bash
sudo ufw allow 21/tcp
sudo ufw reload
sudo ufw status
```

### RHEL-family systems with firewalld

``` bash
sudo firewall-cmd --permanent --add-service=ftp
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

**Passive FTP:** Many clients use passive mode. If transfers or
directory listings fail through a firewall/NAT, configure a passive port
range in `/etc/vsftpd.conf`, for example:

``` ini
pasv_min_port=40000
pasv_max_port=40100
```

Then allow TCP ports `40000-40100` in the server firewall and forward
the same range on any intervening NAT/router. Configure `pasv_address`
when the server is behind NAT and clients connect using its public
address. Restart vsftpd after changing the configuration.

## 6. Verify the service and port

``` bash
sudo systemctl status vsftpd
sudo ss -tlnp | grep ':21'
```

From a client machine, test connectivity:

``` bash
ping SERVER_IP
nc -zv SERVER_IP 21
```

Replace `SERVER_IP` with the FTP server's IP address. A successful ping
is not required if ICMP is blocked; the TCP port test is more relevant.

## 7. Connect using the interactive FTP client

Install the client if needed.

### Ubuntu / Debian

``` bash
sudo apt install ftp -y
```

Connect:

``` bash
ftp SERVER_IP
```

Enter the FTP username and password when prompted.

### Common interactive FTP commands

  Command                    Purpose
  -------------------------- -------------------------------------------------
  `open SERVER_IP`           Connect to a server
  `user username`            Log in as a user
  `ls`                       List remote files
  `dir`                      List remote files (detailed listing)
  `pwd`                      Show the remote working directory
  `cd folder`                Change the remote directory
  `lcd folder`               Change the local directory
  `get file.txt`             Download one file
  `put file.txt`             Upload one file
  `mget *.txt`               Download multiple matching files
  `mput *.txt`               Upload multiple matching files
  `mkdir folder`             Create a remote directory
  `rmdir folder`             Remove an empty remote directory
  `delete file.txt`          Delete a remote file
  `rename old.txt new.txt`   Rename a remote file
  `binary`                   Use binary transfer mode
  `ascii`                    Use text transfer mode
  `prompt`                   Toggle confirmation for multiple-file transfers
  `bye` or `quit`            Disconnect

Use `binary` for ZIP, ISO, PDF, image, and other non-text files.

## 8. Examples

### Download a file

``` text
ftp SERVER_IP
Name: ftpuser
Password: ********
ftp> cd upload
ftp> binary
ftp> get backup.zip
ftp> bye
```

The downloaded file is saved in the client's current local directory.
Check it with `lpwd` (if supported) or `lcd`.

### Upload a file

``` text
ftp SERVER_IP
Name: ftpuser
Password: ********
ftp> cd upload
ftp> binary
ftp> put backup.zip
ftp> bye
```

The remote `upload` directory must exist and the FTP user must have
permission to write to it.

### Upload multiple text files

``` text
ftp SERVER_IP
ftp> cd upload
ftp> prompt
ftp> mput *.txt
ftp> bye
```

### Download multiple text files

``` text
ftp SERVER_IP
ftp> cd upload
ftp> prompt
ftp> mget *.txt
ftp> bye
```

## 9. Transfer files with curl

You can also transfer files without entering the interactive FTP prompt.

### Upload

``` bash
curl --user ftpuser ftp://SERVER_IP/upload/ -T backup.zip
```

Curl prompts for the password when it is not included in the URL. Avoid
putting passwords directly in shell commands because they may be saved
in shell history or visible to other local users.

### Download

``` bash
curl --user ftpuser ftp://SERVER_IP/upload/backup.zip -O
```

## 10. Troubleshooting checklist

-   Check the service: `sudo systemctl status vsftpd`
-   Check the listening port: `sudo ss -tlnp | grep ':21'`
-   Check firewall rules for TCP port 21.
-   If login works but directory listings or transfers hang, check
    passive FTP port-range and NAT/firewall configuration.
-   Confirm the user can access the target directory and has write
    permission for uploads.
-   Review logs, commonly with: `sudo journalctl -u vsftpd`

## 11. Security recommendations

-   Do not use plain FTP over the public internet.
-   Prefer **SFTP** (provided by OpenSSH) when you need secure file
    transfer without running an FTP daemon.
-   If FTP compatibility is required, configure **FTPS with TLS** and
    use clients configured for explicit FTP over TLS.
-   Disable anonymous access unless there is a specific, controlled
    reason to allow it.
-   Use dedicated accounts and restrict their directory access
    appropriately.
