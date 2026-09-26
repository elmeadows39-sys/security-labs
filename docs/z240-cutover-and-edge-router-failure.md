# Z240 Cutover Attempt, the Hidden ONT, and Losing the Edge Router

*August 18 – September 25, 2026*

This is the story of a planned hardware migration that didn't go to plan: moving OPNsense off a Proxmox VM onto dedicated hardware (an HP Z240), rolling it back, discovering a device on the network nobody knew was there, and ultimately losing the household's edge router. It's documented in full, including what was never solved, so the next attempt starts from evidence instead of guesses.

Follow-up: [Router Replacement & the Dual-Homed OPNsense Routing Bug](router-replacement-opnsense-routing.md)

---

## Background

OPNsense has run as a VM (VM 111) on the Proxmox host, behind a consumer router (ASUS RT-AC68U), creating a double NAT. Earlier in the year, recurring state-table drop outages were traced to that architecture. The long-term plan was to move OPNsense onto dedicated hardware and eventually make it the true edge router.

**Hardware chosen:** HP Z240 SFF (Pentium G4400T with AES-NI, 8GB DDR4, 5x Intel gigabit NICs), bought used for about $150.

The Z240 was built and staged in advance with the VM's configuration imported, at a separate staging address, before any cutover.

---

## Part 1: The cutover attempt (Aug 18–20)

### Pre-cutover prep
- Compared firewall rules between VM 111 and the Z240, interface by interface (LAN, WAN, Trusted, IoT, DMZ). They matched, except for one auto-generated "Wazuh agent blocklist" rule on the VM. That turned out to come from an OPNsense plugin installed only on the VM, not from the imported config, so it was left as-is.

### What went wrong during execution

**1. A dead WAN port.** With the ISP cable moved into the Z240's assigned WAN port (onboard `em0`), there was no link at all. `ifconfig -a` showed `em0: no carrier` while a different, unassigned port (`igb0`) showed `status: active`. This was the second port-mapping problem on this hardware; the first was a LAN port mix-up during initial setup. WAN was reassigned to `igb0` from the console.

**2. Reassignment silently broke the VLANs.** Right after the interface reassignment, the Trusted, IoT and DMZ VLAN interfaces showed as unassigned. Their *Enable* checkboxes had reverted to unchecked, and DMZ's subnet mask had reset from `/24` to `/32`. Each was fixed by re-saving the interface by hand.

**3. The ASUS was the DHCP server all along.** When the ASUS was switched to AP mode, the whole flat network lost DHCP. OPNsense had only ever served DHCP to the VLANs. Fixed by enabling Kea DHCP on the Z240's LAN.

**4. VLANs sent traffic but received nothing.** Interface statistics showed Trusted/IoT/DMZ with climbing *Bytes Out* but **zero Bytes In**. Cause: the Z240's switch port had never been tagged for VLANs 10/20/30. That step had been deliberately deferred to avoid an IP conflict with the VM and was never circled back to. Tagging the port fixed the zero-bytes symptom.

### The unsolved problem

Even after the switch port was tagged, a VLAN 10 container (Pi-hole) and the Z240's Trusted gateway (`192.168.10.1`) could not talk at Layer 3:

| Test | Result |
|---|---|
| `ping 192.168.10.1` from Pi-hole | Destination Host Unreachable |
| `ping 192.168.10.110` from the Z240 | Host is down |
| `nc -zv 192.168.10.1 80` from Pi-hole | No route to host |
| `tcpdump -i vlan01 -n arp` on the Z240 | **ARP working correctly in both directions** |
| `tcpdump -i vlan01 -n icmp` on the Z240 | **Zero packets arrived** |

Layer 2 was proven working, yet no real traffic ever arrived. Everything else was checked and ruled out:

- DNS forwarding, NAT (automatic, correct), firewall rules on every interface
- Firewall optimization, reply-to, packet normalization
- Kea/Dnsmasq DHCP conflict
- Switch MAC table staleness (ruled out with a full switch power-cycle)
- Gateway configuration (only WAN gateways present)
- A physical port swap on the switch: **the failure followed the device, not the port**
- Pi-hole's own firewall, routes and addressing, all correct
- Proxmox bridge config: `bridge-vlan-aware yes`, `bridge-vids 2-4094`, correct

### Decision: roll back

With the core problem unsolved and WAN-side connectivity also becoming erratic, the cutover was rolled back to VM 111. The Z240 remains staged and intact. The leading plan for the next attempt is a **clean OPNsense reinstall** rather than continuing on top of two sessions of accumulated state, plus a **packet capture on the Proxmox bridge itself** (`vmbr0`), which was never tried.

---

## Part 2: The hidden device on .254

During rollback, loading `192.168.1.254`, the address OPNsense had used for months, showed an unfamiliar login page branded "SFU." The ARP table showed a MAC prefix belonging to no lab hardware. A vendor lookup identified it as a manufacturer of fiber ONT/ONU equipment.

The ISP's (EPB) residential fiber ONT had been sitting on `.254` the entire time. OPNsense had apparently been winning the address through ARP timing, until the cutover's repeated reboots and interface changes exposed the conflict.

**Resolution:** OPNsense's LAN moved permanently to `192.168.1.252`, and `.254` is now reserved: never assign it to lab gear. The ONT is managed centrally by the ISP and isn't user-configurable.

*(Later, in September, OPNsense's LAN address was removed entirely as part of a routing fix. See the follow-up write-up.)*

---

## Part 3: Losing the ASUS

### August: recovery
After the rollback, the ASUS's admin GUI was unreachable. Standard reset-button presses only rebooted it. Research found the RT-AC68U needs a model-specific hard reset: **unplug, hold reset, plug power back in while still holding, and keep holding until the power LED's blink pattern changes.** That produced a real factory reset, and the router was reconfigured and working again.

The cost: every custom setting was wiped, including DNS pointing at Pi-hole, the WireGuard port forward, and a static route to VLAN 10 that had never been documented.

### September: permanent failure
The ASUS failed again, for good:
- Hard-reset repeatedly, using both the power-on-hold method and a WPS-button-while-powering-on method. Both were confirmed genuine (the bare default network name appeared each time).
- Tested **fully isolated**, with only the ISP cable connected.
- It obtained a fresh public IP from the ISP every time, but **never passed traffic**; `tracert` died at hop 1.
- The ISP confirmed their side was healthy, and a device plugged straight into the ISP cable got internet immediately.

That isolates the fault to the router itself. It was retired. A firmware reflash was never attempted.

### The stopgap and the replacement
1. **Netgear R6250 (temporary):** switched out of AP mode into router mode. It defaulted to a `10.0.0.0/24` LAN, which didn't match the lab, but was accepted as temporary.
2. **TP-Link Archer AX1300 (permanent, about $32):** gigabit on all wired ports, WiFi 6. The LAN was changed from its default `192.168.0.1` to `192.168.1.1`, and the VLAN 10 static route was recreated by hand.

Swapping routers then exposed a separate, long-hidden routing flaw in OPNsense, covered in the [follow-up write-up](router-replacement-opnsense-routing.md).

---

## Lessons learned

- **Know every device on your network.** A device the ISP installed occupied the firewall's address for months, unnoticed.
- **Know which box does what.** The edge router, not OPNsense, was the flat network's DHCP server. A cutover plan has to account for every service the old device quietly provided.
- **Don't defer "small" prep steps.** The untagged switch port cost hours of troubleshooting. Pre-cutover checklists should include switch tagging and verifying WAN link before cutover day.
- **After reassigning a physical interface in OPNsense, re-check every VLAN interface.** Enable flags and subnet masks can silently revert.
- **Replacing a router loses every custom setting.** Keep a written checklist: LAN address, static routes, DHCP reservations, DNS and port forwards.
- **Consumer routers can have model-specific reset procedures.** A normal reset press may only reboot.
- **Rolling back is a valid outcome.** Documenting what was ruled out is what makes the next attempt faster.

## Status

| Item | Status |
|---|---|
| HP Z240 | Staged, offline. Next attempt: clean reinstall and a bridge-level packet capture |
| Z240 Layer 3 mystery (ARP works, traffic doesn't) | **Unsolved** |
| EPB ONT on `.254` | Identified, address reserved |
| ASUS RT-AC68U | Retired (reflash never attempted) |
| Netgear R6250 | Retired from router duty; possible future IoT access point |
| TP-Link AX1300 | Current edge router |
