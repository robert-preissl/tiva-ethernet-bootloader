# tiva-ethernet-bootloader
Instructions for an ethnernet based bootloader for the Tiva TM4C129X series
<br><br> 

Concrete client hardware this tutorial is based on: DK-TM4C129X
Server: Macbook Pro

some documentation from TI: http://www.ti.com/lit/ug/spmu301d/spmu301d.pdf

very useful description for modifying the Bootloader and eflash tool: https://github.com/kroesche/



In order to get the DK-TM4C129X working with enet updates, **three** apps are involved:

<br><br>
**1.) the bootloader on the device : "boot_emac_flash"**

- make sure it will be deployed at the beginning of flash

- adapt in bl_emac.c the port definitions according to: https://github.com/kroesche/stellaris_eflash : i.e, add 40000 to all port numbers to be non-priveliged ports.

set:

define BOOTP_SERVER_PORT       (67 + PORT_OFFSET)
define BOOTP_CLIENT_PORT       (68 + PORT_OFFSET)
define TFTP_PORT               (69 + PORT_OFFSET)

+

include "/SW-TM4C-2.1.3.156/third_party/uip-1.0/uip/uip_.c"

+ edit uip_.c to have: HTONS(40000+69))


- in bl_config.h:

define ENET_BOOTP_SERVER       "stellaris"

- make sure to set an IP address statically in bl_emac.c (we have no DHCP server) :
uip_ipaddr_t addrh;
uip_ipaddr(&addrh, 192,168,22,109);
uip_sethostaddr(&addrh);

(I have set it after we do "uip_udp_new")

- double check all ports: client & server



<br><br> 
**2.) the example app on the device. which jumps to the bootloader if something happens (like a button press or a magic packet comes in)**

- make sure to have a code structure like this:

SoftwareUpdateInit(SoftwareUpdateRequestCallback);

while(!g_bFirmwareUpdate) // this will be left when the magic packet arrives.
{
  // do stuff
}

SoftwareUpdateBegin(g_ui32SysClock); // Transfer control to the bootloader.


- make sure to #define APP_BASE 0x00004000 for the program app. so that it does not interfere with the bootloader's memory layout in flash.

- make sure to when running in CCS. go to Run Debug configuration. and choose erase flash only in certain range:

start = 0x00004000
since the length of my program is : "0001c884" (in used column of .map file)
end   = 0x00020884 (using hex calculator): http://www.csgnetwork.com/hexaddsubcalc.html

- make sure that in "SoftwareUpdateInit" we can receive from any IP address the magic packet:  udp_bind(g_psMagicPacketPCB, IP_ADDR_ANY, MPACKET_PORT);





<br><br> 
**3.) the server app "eflash" to start the BOOTP process and then send the image via TFTP**

- somehow I had to create an additional socket to "sBOOTP" in order to send the BOOTP reply. sBOOTP still does the recvfrom for the BOOTP request from the bootloader.

- to compile: gcc -o eflash bootp_server.c eflash.c

- to run:     ./eflash -i 192.168.22.109 --mac 00:1a:b6:00:00:01 -l 192.168.22.1 blinky.out --verbose

- make sure to check on the right device name when verifying the BOOTP packet. (it can be "tiva" or "stelaris"):  (strcasecmp(pPacket->pcSName, "stellaris")





<br><br> 
**Tricks to debug:**

- when in assembly mode. go to run debug config. load symbols only. (make sure not to terminate when loading/ or reset). ok to stop current debug session.

- use Wireshark

- double check all ports: client & server

- use UniFlash to check Mac addresses

- use UniFlash to check that the bootloader is correctly at beginning of flash
