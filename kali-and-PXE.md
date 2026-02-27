## Boot Kali as PXE through TFTP
Guide to creating a PXE booted Kali Linux.
  
### 1. Preparations
- A Linux server (e.g. Debian/Ubuntu), DHCP/TFTP support, at least 4 GB RAM for the client (due to `filesystem.squashfs`).
- Download Kali ISO  
  `wget https://kali.download/kali-images/kali-2025.1/kali-linux-2025.1-live-amd64.iso`  
- **Requirements**:  
  `sudo apt install -y build-essential git syslinux-common`  
  `sudo apt install -y isc-dhcp-server tftpd-hpa`

### 2. Download and mount ISO and copy files
```bash
cd ~/Downloads
wget https://kali.download/kali-images/kali-2025.1/kali-linux-2025.1-live-amd64.iso
sudo mkdir /mnt/kali-iso
sudo mount -o loop ~/Downloads/kali-linux-2025.2-live-amd64.iso /mnt/kali-iso
sudo mkdir -p /srv/tftp/kali
sudo cp /mnt/kali-iso/live/{vmlinuz,initrd.img,filesystem.squashfs} /srv/tftp/kali/
sudo umount /mnt/kali-iso
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

subnet 192.168.185.0 netmask 255.255.255.0 {
  range 192.168.185.100 192.168.185.200;
  option routers 192.168.185.1;
  option domain-name-servers 8.8.8.8;
  option broadcast-address 192.168.185.255;
  next-server 192.168.185.1;
  filename "pxelinux.0";
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


### 5. Configure the PXE boot
```bash
sudo mkdir /srv/tftp/pxelinux.cfg
sudo apt install syslinux-common
sudo cp /usr/lib/syslinux/modules/bios/{pxelinux.0,vesamenu.c32,libutil.c32,ldlinux.c32,libcom32.c32,menu.c32} /srv/tftp/
```
Note: 
If `pxelinux.0` is missing, run `sudo apt install pxelinux`.
The requested file should then be located under `/usr/lib/PXELINUX/pxelinux.0`.
Copy the file to `/srv/tftp`.

```bash
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
```

#### Create menu file
Make sure to use the correct IP for the `tftp` server.
```bash
sudo vi /srv/tftp/pxelinux.cfg/default  
```
```conf
DEFAULT vesamenu.c32
PROMPT 0
TIMEOUT 300
MENU TITLE PXE Boot Menu

LABEL kali
  MENU LABEL Kali Linux Live
  KERNEL kali/vmlinuz
  APPEND initrd=kali/initrd.img boot=live components username=root hostname=kali-live fetch=tftp://192.168.185.1/kali/filesystem.squashfs
```

### 6. Start services
There are two different services that need to be started for everything to work. These are the dhcp and tftp services.
```bash
sudo systemctl restart isc-dhcp-server tftpd-hpa
sudo systemctl enable isc-dhcp-server tftpd-hpa
```

### 7. Client configuration
There are some settings for the PXE machine that need to be set in VMware Workstation.
- **Network Adapter** → `host-only`
- **PXE-boot** → `BIOS`  
  VM Settings → Options → Advanced → Firmware type

### 8. Troubleshooting
- **Firewall** 
  → Make sure that traffic to `69 UDP` and `80 TCP` is allowed - or turn off the firewall.
- **Client don't get an IP** 
  → Check DHCP configuration and firewall.  
  `sudo tcpdump -i any port 67 or port 68 or port 80 -n -vv`  
  `journalctl -u isc-dhcp-server.service -f`  
  The DHCP service should show `DHCPDISCOVER, DHCPOFFER, DHCPREQUEST, DHCPACK`.


