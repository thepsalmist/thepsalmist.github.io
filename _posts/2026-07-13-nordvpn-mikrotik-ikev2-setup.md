---
layout: post
title: "NordVPN on MikroTik with IKEv2: A Complete RouterOS 6.49 Setup Guide"
description: >-
  Step-by-step guide to connecting a MikroTik router to NordVPN using
  IKEv2/IPsec on RouterOS 6.49 — full configuration chain, from the default
  config to a full VPN tunnel with kill switch, plus PPPoE and troubleshooting.
tags: [mikrotik, routeros, nordvpn, ikev2, networking]
slug: nordvpn-mikrotik-ikev2-setup
---

I recently put my home network behind a MikroTik router running RouterOS **6.49**, with all
LAN traffic exiting through NordVPN over IKEv2/IPsec. This is the full setup, starting from a
router freshly **reset to defaults (defconf)**. Because defconf already builds the bridge,
LAN/WAN interface lists, DHCP, NAT, and a sane firewall, most of the work is
**verify + a few small changes** — nothing gets recreated from scratch.

Three phases:

1. **Phase 1** – Verify the default config, then apply 3 deltas (DNS, FastTrack, WiFi).
2. **Phase 2** – NordVPN (IKEv2/IPsec) so *all* LAN traffic exits through the VPN.
3. **Phase 3** – Switch the ISP box to **bridge mode** + authenticate with **PPPoE**.

> **Assumptions in this guide**
> - VPN server: **`uk2156.nordvpn.com`** — swap in any hostname from NordVPN's server picker
> - LAN subnet (defconf default): **192.168.88.0/24**, router at **192.168.88.1**
> - WAN = `ether1` · LAN = `bridge` (ether2–5 + wlan)
> - Do Phase 2 **while internet works** so the router can fetch the certificate.
> - Keep a **Winbox** session open (connects via MAC) so a bad firewall edit can't lock you out.

---

## ⚠️ Three gotchas that silently break this exact setup

- **FastTrack bypasses IPsec** → fixed in Delta 2 below (defconf ships a FastTrack rule).
- **DNS leaks** → fixed in Delta 1 (+ Phase 2.7).
- **No kill switch by default** → optional rule in Phase 2.8.

---

## Phase 1 — Verify defconf, then apply 3 deltas

### 1A. Verify what defconf already created

All of these ship with the default configuration — verify them, but create **nothing** here;
do not re-add.

| What | Expected value | Status |
|---|---|---|
| Interface lists | `WAN`, `LAN` (defconf) | ✅ leave as-is |
| Bridge ports | ether2–5 **+ wlan1 + wlan2** on `bridge`; ether1 outside | ✅ leave as-is |
| List members | ether1→WAN, bridge→LAN | ✅ leave as-is |
| LAN IP | `192.168.88.1/24` on bridge | ✅ leave as-is |
| DHCP server | `defconf` on bridge, pool `default-dhcp`, 10m lease | ✅ leave as-is |
| NAT | masquerade `out-interface-list=WAN` `ipsec-policy=out,none` | ✅ **do not touch** (see note) |
| Forward ipsec | rules #6/#7 accept `ipsec-policy in/out` | ✅ VPN traffic pre-authorized |
| WAN client | DHCP client on ether1, `use-peer-dns=yes` | ⚠️ changed in Delta 1 |
| FastTrack | rule **#8** `action=fasttrack-connection` | ⚠️ disabled in Delta 2 |

> **Why NAT stays untouched:** the defconf masquerade carries `ipsec-policy=out,none`, so it
> deliberately skips traffic headed into the VPN tunnel (that traffic gets its own dynamic
> IPsec NAT). This is exactly what we want — editing it would break the tunnel's NAT.

> **Before Phase 2:** make sure the WAN DHCP client is **bound** (`/ip dhcp-client print` →
> status `bound`, and a dynamic address appears under `/ip address print`). The router needs
> real internet to fetch the cert.

### 1B. The only 3 changes you make in Phase 1

#### Delta 1 — Stop ISP DNS + give the router its own resolver

`use-peer-dns=no` stops your ISP injecting DNS. The `servers` line lets the **router itself**
resolve names (needed to fetch the cert and reach the NordVPN server). Your LAN clients'
DNS is set separately in Phase 2.7 so it rides the tunnel.

```
/ip dhcp-client set [find interface=ether1] use-peer-dns=no
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes
```

#### Delta 2 — Disable FastTrack (or the VPN gets bypassed)

On a stock defconf this is **rule #8**. The command finds it by action, so you don't need
the number:

```
/ip firewall filter disable [find action=fasttrack-connection]
```

(The dynamic "dummy fasttrack counter" rule #0 stays — it does nothing once #8 is disabled.)

#### Delta 3 — Enable + configure WiFi (RouterOS 6.49 = `wireless` package)

On a dual-band router (`wlan1` + `wlan2` — defconf already puts both in the bridge, but
disabled), run all of the following. Change the SSIDs/password to your liking:

```
# set the WiFi password / encryption on the default security profile
/interface wireless security-profiles
set default mode=dynamic-keys authentication-types=wpa2-psk wpa2-pre-shared-key="YourWiFiPassword"

# 2.4 GHz radio (wlan1)
/interface wireless
set wlan1 disabled=no mode=ap-bridge ssid="MyWiFi" band=2ghz-b/g/n security-profile=default

# 5 GHz radio (wlan2)
set wlan2 disabled=no mode=ap-bridge ssid="MyWiFi-5G" band=5ghz-a/n/ac security-profile=default
```

> Both wlan interfaces are already bridge ports on defconf, so no bridge-port commands are needed.

**Stop here and confirm normal internet works** (wired + WiFi) before touching the VPN.

---

## Phase 2 — NordVPN full tunnel (IKEv2 / IPsec EAP)

IKEv2 EAP is fully supported on 6.49. You need your **NordVPN service credentials**
(NOT your account email/password): Nord Account → **NordVPN** → **Manual setup / Service
credentials** → copy **Username** + **Password**.

### 2.1 Install the NordVPN root certificate

```
/tool fetch url="https://downloads.nordcdn.com/certificates/root.der"
/certificate import file-name=root.der passphrase=""
```

### 2.2 IPsec profile, proposal, policy group + template

```
/ip ipsec profile
add name=NordVPN

/ip ipsec proposal
add name=NordVPN pfs-group=none

/ip ipsec policy group
add name=NordVPN

/ip ipsec policy
add dst-address=0.0.0.0/0 src-address=0.0.0.0/0 group=NordVPN proposal=NordVPN template=yes
```

### 2.3 Which traffic uses the tunnel (the whole LAN)

```
/ip firewall address-list
add address=192.168.88.0/24 list=LOCAL
```

### 2.4 Mode config (requests settings from Nord; bound to the LAN)

```
/ip ipsec mode-config
add name=NordVPN responder=no src-address-list=LOCAL
```

### 2.5 Peer + identity (insert YOUR service username/password)

```
/ip ipsec peer
add address=uk2156.nordvpn.com exchange-mode=ike2 name=NordVPN profile=NordVPN

/ip ipsec identity
add peer=NordVPN auth-method=eap eap-methods=eap-mschapv2 certificate="" \
  generate-policy=port-strict mode-config=NordVPN policy-template-group=NordVPN \
  username="YOUR_NORD_SERVICE_USERNAME" password="YOUR_NORD_SERVICE_PASSWORD"
```

### 2.6 Verify the tunnel

```
/ip ipsec policy print          ;# PH2 State: established
/ip ipsec active-peers print    ;# shows uk2156.nordvpn.com connected
/ip firewall nat print          ;# a dynamic IPsec NAT rule appears
```

From a LAN client, an IP-check site should show the VPN server's location (UK / London for
this server); latency will rise (expected).

### 2.7 (Recommended) Push NordVPN DNS to clients so lookups ride the tunnel

This hands your LAN clients NordVPN's DNS directly via DHCP. Because the queries are sourced
from the LAN (192.168.88.x), they travel through the tunnel instead of leaking out the WAN:

```
/ip dhcp-server network set [find address=192.168.88.0/24] dns-server=103.86.96.100,103.86.99.100
```

Clients pick this up on their next lease renewal (reconnect WiFi / `ipconfig /renew`, or wait
out the 10-minute lease).

> Note: the router's *own* DNS lookups and the IPsec negotiation use the real WAN by design —
> that's required for the tunnel to come up. Your clients' DNS is what rides the tunnel.

### 2.8 (Optional) Kill switch — drop LAN→internet when the tunnel is down

The defconf forward chain has **no explicit LAN→WAN accept rule** — it lets LAN traffic out
via the implicit accept at the end of the chain. So you just **add** this rule and it lands at
the bottom, which is exactly the right spot. No `move` needed.

```
/ip firewall filter
add chain=forward action=drop src-address=192.168.88.0/24 out-interface-list=WAN \
  ipsec-policy=out,none comment="VPN kill switch"
```

How it behaves: tunnel **up** → LAN traffic matches the ipsec accept (rule #7) before reaching
this; tunnel **down** → LAN traffic has no ipsec policy, falls to this rule, and is dropped
(fail-closed) instead of leaking to the plain internet.

> This only affects LAN client traffic (`src-address=192.168.88.0/24`, forward chain). The
> router's own IPsec negotiation and DNS are in other chains, so the tunnel can still connect.
> To temporarily allow normal internet again: `/ip firewall filter disable [find comment="VPN kill switch"]`

---

## Phase 3 — ISP router to bridge mode + PPPoE

Do this **after** Phases 1–2 are confirmed working. On defconf you don't create a WAN list —
you just re-point its member from `ether1` to the new PPPoE interface.

### 3.1 On the ISP router

Enable **bridge mode** ("modem only" / disable its routing + DHCP). Internet drops until 3.2.

### 3.2 Create the PPPoE client on ether1 (use your **ISP** PPPoE login)

```
/interface pppoe-client
add name=pppoe-out1 interface=ether1 disabled=no \
  user="ISP_PPPOE_USERNAME" password="ISP_PPPOE_PASSWORD" \
  add-default-route=yes use-peer-dns=no
```

### 3.3 Remove the old DHCP WAN and re-point the existing WAN list member

```
/ip dhcp-client remove [find interface=ether1]

/interface list member
remove [find interface=ether1 list=WAN]
add interface=pppoe-out1 list=WAN
```

NAT, firewall, and the NordVPN peer all follow the `WAN` list, so they update automatically.
The IPsec tunnel re-establishes once PPPoE is up.

### 3.4 MSS clamping (important for PPPoE + IPsec)

```
/ip firewall mangle
add chain=forward action=change-mss new-mss=clamp-to-pmtu passthrough=yes \
  protocol=tcp tcp-flags=syn out-interface-list=WAN comment="MSS clamp for PPPoE/VPN"
```

### 3.5 Verify

```
/interface pppoe-client print     ;# status: connected, has an IP
/ip ipsec active-peers print      ;# NordVPN reconnected
```

---

## Quick troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `PH2` not established | Wrong **service** creds (not account login); cert not imported; clock wrong → set NTP. |
| Internet works but IP isn't the VPN's | Traffic bypassing IPsec — usually leftover **FastTrack** (Delta 2) or wrong `LOCAL` subnet. |
| Tunnel up but no client internet | DNS — confirm Delta 1 / 2.7 and clients use `192.168.88.1`. |
| Pages half-load after PPPoE | Missing **MSS clamp** (3.4). |
| Locked out after firewall change | Reconnect with **Winbox via MAC**, fix rule order. |
| Temporarily disable VPN | `/ip ipsec identity disable [find peer=NordVPN]` (disable kill-switch rule first). |

## Switch NordVPN country later

```
/ip ipsec peer set [find name=NordVPN] address=NEWSERVER.nordvpn.com
```
