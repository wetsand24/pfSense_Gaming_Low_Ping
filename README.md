# pfSense gaming settings for lower ping

Five pfSense settings for smoother online games: hardware offloading, UPnP, static port NAT, Fortnite port aliases and a traffic shaper that puts game traffic first.

▶️ Video: https://youtu.be/828R_GlPiao

The video uses Fortnite, but steps 1 to 4 work the same for most games. The addresses below are the ones from the video. Change them to yours:

| | Address | What it is |
|---|---|---|
| Gaming network | `10.50.0.0/20` | the video's separate gaming VLAN (interface `GAMING`) |
| Example LAN | `192.168.0.0/24` | a normal home LAN |
| Example gaming device | `192.168.0.99` | your PC or console on that LAN |
| pfSense | `10.37.0.1` | the pfSense box (web page on port `65000` in the video) |

## What you need
- pfSense CE on a machine with:
  - a 64-bit amd64 (x86-64) CPU
  - 1 GB of RAM or more
  - an 8 GB or larger disk
  - one or more supported network cards
- For step 5 only: Intel network cards that show up as `igb`

The video uses pfSense CE 2.6.0.

## 1. Hardware offloading
**System → Advanced → Networking**, scroll down to **Network Interfaces**.

The video unchecks all three boxes, then clicks **Save**:

| Field | Value |
|---|---|
| Hardware Checksum Offloading | ☐ Disable hardware checksum offload |
| Hardware TCP Segmentation Offloading | ☐ Disable hardware TCP segmentation offload |
| Hardware Large Receive Offloading | ☐ Disable hardware large receive offload |

These boxes *disable* offloading when ticked, so unchecking them turns offloading on. This is for Intel cards. Many Realtek cards don't handle offloading well, so you may not gain anything there.

> Note (2026): the pfSense docs now say TSO and LRO should stay disabled (boxes ticked) on a router or firewall, and not to uncheck them unless Netgate support tells you to.

## 2. UPnP & NAT-PMP
**Services → UPnP & NAT-PMP**

| Field | Value |
|---|---|
| Enable | ✔ Enable UPnP & NAT-PMP |
| UPnP Port Mapping | ✔ Allow UPnP Port Mapping |
| NAT-PMP Port Mapping | ✔ Allow NAT-PMP Port Mapping |
| External Interface | `WAN` |
| Interfaces | `LAN` (the video picks its `GAMING` VLAN) |

Scroll down to **UPnP Access Control Lists → ACL Entries**. The format is:
```
allow [ext port or range] [int ipaddr or ipaddr/CIDR] [int port or range]
```
The video allows its whole gaming VLAN, because nothing else lives on it:
```
allow 1024-65535 10.50.0.0/20 1024-65535
```
On a normal LAN, don't allow the whole network. Add one line per gaming device, with `/32` (one single address):
```
allow 1024-65535 192.168.0.99/32 1024-65535
```
Second device? Click **Add** and do the same with its IP, for example `192.168.0.81/32`.

Click **Save**.

## Static port outbound NAT
**Firewall → NAT → Outbound**. The mode in the video is **Hybrid Outbound NAT rule generation** (automatic rules plus your own rules below). Under **Mappings**, click **Add**:

| Field | Value |
|---|---|
| Interface | `WAN` |
| Address Family | `IPv4+IPv6` |
| Protocol | `any` |
| Source | Network, `10.50.0.0` / `20` (your gaming network or device) |
| Destination | Any |
| Translation → Address | Interface Address |
| Translation → Static Port | ✔ |
| Description | `UPNP for Gaming vlan` |

Click **Save**, then **Apply Changes**.

## 3. Fortnite port aliases
**Firewall → Aliases → Ports → Add**. Make two aliases. Type a port or range (with a colon) per line, and click **Add Port** for the next one.

| Name | Description | Type | Ports |
|---|---|---|---|
| `Fortnite_TCP_Ports` | `Fortnite_TCP_Ports` | Port(s) | `433`, `3478:3479`, `5060`, `5062`, `5222`, `6250`, `12000:65000` |
| `Fortnite_UDP_Ports` | `Fortnite_UDP_ports` | Port(s) | `3478:3479`, `5060`, `5062`, `6250`, `9000:10000`, `12000:65000` |

Click **Save** after each one, then **Apply Changes**.

## 4. Traffic shaper wizard
**Firewall → Traffic Shaper → Wizards**. One internet connection: **Dedicated Links** (`traffic_shaper_wizard_dedicated.xml`). Then:

**Step 1 of 8, Shaper configuration**

| Field | Value |
|---|---|
| Local interface | `LAN` (the video uses `GAMING`), `PRIQ` |
| WAN Interface | `WAN`, `PRIQ` |
| Upload | `30` `Mbit/s` |
| Download | `800` `Mbit/s` |

Set upload and download to about 85% of your real speed test. In the video upload tests at about 32, so it's set to 30. Download is 1 Gbit on paper and about 900 in tests, so it's set to 800 to be safe.

Click **Next** through the next pages without changes. On **Peer to Peer networking** leave **Lower priority of Peer-to-Peer traffic** unticked.

**Step 5 of 8, Network Games**
- Enable: ✔ **Prioritize network gaming traffic**
- Tick: **BattleNET**, **EAOrigin**, **PlayStationConsoles**, **Steam**, **WiiConsoles**, **XboxLive**, **GoogleStadia**

Then **Next**, **Next**, **Finish**.

## Floating rules for Fortnite ports
**Firewall → Rules → Floating**. The wizard made a lot of rules here. Make two more for the Fortnite aliases by copying ones it made, so everything else stays the same.

**UDP rule.** Find `m_Game Battle.NET-game1-udp outbound` and click its copy icon. Change:

| Field | Value |
|---|---|
| Quick | ✔ Apply the action immediately on match |
| Address Family | `IPv4` |
| Protocol | `UDP` (already set) |
| Destination Port Range | From (other) `Fortnite_UDP_Ports`, To (other) `Fortnite_UDP_Ports` |
| Description | `Fortnite UDP ports` |
| Ackqueue / Queue | `none` / `qGames` |

Click **Save**. Don't apply yet.

**TCP rule.** Copy `m_Game Battle.NET-game1-tcp outbound`. Change:

| Field | Value |
|---|---|
| Quick | ✔ Apply the action immediately on match |
| Address Family | `IPv4` |
| Protocol | `TCP` (already set) |
| Destination Port Range | From (other) `Fortnite_TCP_Ports`, To (other) `Fortnite_TCP_Ports` |
| Description | `Fortnite TCP ports` |
| Ackqueue / Queue | `qACK` / `qGames` |

Leave Action (`Match`), Interface (`WAN`) and Direction (`any`) as copied. Click **Save**, then **Apply Changes**.

## 5. loader.conf.local (Intel `igb` cards only)
Optional. These values are for Intel cards that pfSense calls `igb`. Check yours under **Interfaces → Assignments**: the network port column shows `igb0`, `igb1` and so on. Realtek and other cards need different values, so skip this step if you don't see `igb`.

**Turn on SSH:** **System → Advanced → Admin Access** → **Secure Shell Server** = ✔ Enable Secure Shell → **Save**.

**Log in** with PuTTY: Host Name = your pfSense IP (`10.37.0.1` in the video), Port `22`, user `admin`. In the menu, type `8` (Shell) and press Enter.

Install nano (a text editor, not there by default):
```
pkg install nano
```
Open the file. If it doesn't exist yet, nano creates it:
```
nano /boot/loader.conf.local
```
Paste these 18 lines (also in [loader.conf.local](loader.conf.local)):
```
kern.cam.boot_delay=10000
kern.ipc.nmbclusters="1000000"
kern.ipc.nmbjumbop="524288"
kern.ipc.nmbjumbo9="524288"
hw.igb.num_queues="2"
net.inet.tcp.tso="1"
hw.ix.flow_control="0"
hw.bce.tso_enable="0"
hw.igb.fc_setting="0"
hw.igb.max_interrupt_rate="30000"
hw.igb.rxd="1096"
hw.igb.txd="1096"
hw.igb.rx_process_limit="-1"
hw.igb.enable_msix="1"
autoboot_delay="3"
net.isr.maxthreads="-1"
net.isr.bindthreads="1"
machdep.hyperthreading_allowed="0"
```
Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`. Check it:
```
cat /boot/loader.conf.local
```
Then reboot:
```
reboot
```
These values are only read at boot. Once pfSense is back up, start your game and see how it plays.

## Files
| File | What it is |
|---|---|
| [loader.conf.local](loader.conf.local) | the 18 lines for step 5 |

## Links
- pfSense: https://www.pfsense.org
- pfSense docs: https://docs.netgate.com/pfsense/en/latest/

More pfSense:
- Tailscale on pfSense: https://youtu.be/HDFw4EVDZj8
- pfSense to Azure site-to-site VPN: https://youtu.be/sEgKqQ8Gm3Y
- pfSense to AWS site-to-site VPN: https://youtu.be/p83RmeT2Q-A
