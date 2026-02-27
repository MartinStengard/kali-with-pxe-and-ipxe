## Boot Kali as iPXE through HTTP
Guide to creating a iPXE booted Kali Linux.

### 1. Preparations
- A Linux server (e.g. Debian/Ubuntu), DHCP/TFTP support, at least 4 GB RAM for the client (due to `filesystem.squashfs`).
- Download Kali ISO  
  `wget https://kali.download/kali-images/kali-2025.1/kali-linux-2025.1-live-amd64.iso`  
- **Requirements**:  
  `sudo apt install -y build-essential git syslinux-common`  
  `sudo apt install -y apache2 isc-dhcp-server tftpd-hpa`

### 1. Web server
The files for the boot process need to be availabe through a HTTP resource. 
We solve it by using `apache2` web server on the host machine.

#### Installation
```bash
sudo apt install apache2
```
Root folder for the `apache2` web server is `/var/www/html`.

### 2. Download and mount ISO and copy files
```bash
cd ~/Downloads
wget https://kali.download/kali-images/kali-2025.1/kali-linux-2025.1-live-amd64.iso
sudo mkdir /mnt/kali-iso
sudo mount -o loop ~/Downloads/kali-linux-2025.2-live-amd64.iso /mnt/kali-iso
sudo mkdir -p /var/www/html/kali
sudo cp /mnt/kali-iso/live/vmlinuz /var/www/html/kali/
sudo cp /mnt/kali-iso/live/initrd.img /var/www/html/kali/
sudo cp /mnt/kali-iso/live/filesystem.squashfs /var/www/html/kali/
```

### 3. DHCP
Use `host-only` for the machine's network adapter. By default, the virtual interface `vmnet1` on the host machine is used.  

In `dhcpd.conf`, the IP address of the virtual network interface `vmnet1` (e.g. `192.168.185.1`) should be used, 
since it is the one that acts as the DHCP/TFTP server in VMware's `host-only` network.  

It is also the interface `vmnet1` that later should then be activated.  

#### Installation
```bash
sudo apt install isc-dhcp-server
```

#### Configuration
Example of configuration for `vmnet1`:  
```bash
$ ifconfig
vmnet1: inet 192.168.185.1  netmask 255.255.255.0  broadcast 192.168.185.255
```  

The above information gives the following configuration:
```bash
sudo vi /etc/dhcp/dhcpd.conf  
```
```conf
allow booting;
allow bootp;

option client-architecture code 93 = unsigned integer 16;

subnet 192.168.185.0 netmask 255.255.255.0 {
  range 192.168.185.100 192.168.185.200;
  option routers 192.168.185.1;
  option domain-name-servers 8.8.8.8;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.185.255;
  default-lease-time 600;
  max-lease-time 7200;
  next-server 192.168.185.1;
}

# If ipxe.efi can be loaded with HTTP (not working for vmware workstation)
# if option client-architecture = 7 {
#   option vendor-class-identifier "HTTPClient";
#   filename "http://192.168.185.1/ipxe.efi";
# }

# For vmware workstation, the ipxe.efi is loaded from TFTP
# and the actual boot.ipxe from HTTP.
if exists user-class and option user-class = "iPXE" {
  filename "http://192.168.185.1/boot.ipxe";
} else {
  filename "ipxe.efi";
}
```

#### Activate
Change or add the correct DHCP network interface to activate.  
```bash
sudo vi /etc/default/isc-dhcp-server  
```
```conf
INTERFACESv4="vmnet1"
```

### 4. TFTP
TFTP is used to download all the files needed to start Kali.  

#### Installation
```bash
sudo apt install tftpd-hpa
```

#### Configuration
```bash
sudo vi /etc/default/tftpd-hpa  
```
```conf
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure --create --verbose --blocksize 1024"
```

#### Firewall
Make sure that any firewall is not blocking traffic to `port 69 UDP`.

### 5. Build iPXE file for UEFI
```bash
git clone https://github.com/ipxe/ipxe.git
cd ipxe/src
make bin-x86_64-efi/ipxe.efi

# Copy to TFTP root if it needs to be loaded with TFTP
sudo cp bin-x86_64-efi/ipxe.efi /mnt/kali-iso/

# Copy file to web server root if it can be loaded with HTTP
sudo cp bin-x86_64-efi/ipxe.efi /var/www/html/
```

### 6. Create iPXE script
Create `boot.ipxe` with configurations on the web server. Below to versions are tested successfully.  
```bash
sudo vi /var/www/html/boot.ipxe  
```
```bash
#!ipxe
kernel http://192.168.185.1/kali/vmlinuz boot=live components
initrd http://192.168.185.1/kali/initrd.img
imgargs vmlinuz boot=live components fetch=http://192.168.185.1/kali/filesystem.squashfs
boot
```
```bash
#!ipxe
kernel http://192.168.185.1/kali/vmlinuz boot=live fetch=http://192.168.185.1/kali/filesystem.squashfs username=kali hostname=kali
initrd http://192.168.185.1/kali/initrd.img
boot
```

### 7. Start services
There are three different services that need to be started for everything to work. These are the web server, dhcp and tftp services.
```bash
sudo systemctl restart apache2 isc-dhcp-server tftpd-hpa
sudo systemctl enable apache2 isc-dhcp-server tftpd-hpa
```

### 8. Client configuration
There are some settings for the PXE machine that need to be set in VMware Workstation.
- **Network Adapter** → `host-only`
- **PXE-boot** → `UEFI`  
  VM Settings → Options → Advanced → Firmware type  
  If `UEFI` is disabled add below line in the `.vmx` file:  
  `firmware="efi"`
- **Guest Operating System** → `64-bit`  
  VM Settings → Options → General - Guest Operating System
  `Ubuntu 64-bit (för Kali Linux)`  
- **Extra settings**  
  If large files can't be loaded add below line in the `.vmx` file:  
  `ethernet0.virtualDev = "e1000"`

### 9. Troubleshooting
- **Firewall** 
  → Make sure that traffic to `69 UDP` and `80 TCP` is allowed - or turn off the firewall.
- **Timeout**
  → Extend timeout for web server →  `/etc/apache2/apache2.conf`
  `Timeout 2400`
- **Client don't get an IP** 
  → Check DHCP configuration and firewall.  
  `sudo tcpdump -i any port 67 or port 68 or port 80 -n -vv`  
  `journalctl -u isc-dhcp-server.service -f`  
  The DHCP service should show `DHCPDISCOVER, DHCPOFFER, DHCPREQUEST, DHCPACK`.


