# Kali Linux Networking Lab — VirtualBox NAT Network

This lab documents a basic network setup for a Kali Linux virtual machine running in Oracle VirtualBox. The VM is connected to a VirtualBox **NAT Network** so it can reach external networks while using a private IPv4 subnet.

## Lab objectives

- Create and configure a VirtualBox NAT Network.
- Connect the Kali Linux VM to that NAT Network.
- Configure IPv4 address, gateway, and DNS settings in NetworkManager.
- Verify IP connectivity and DNS resolution from Kali Linux.

## Lab environment

| Component | Configuration |
|---|---|
| Guest OS | Kali Linux 2026.3 (VirtualBox AMD64 image) |
| Hypervisor | Oracle VirtualBox |
| VirtualBox network mode | NAT Network |
| NAT Network name | `NatNetwork` |
| IPv4 network | `10.0.0.0/24` |
| NAT gateway | `10.0.0.1` |
| Guest interface | `eth0` |
| Intended guest IPv4 | `10.0.0.2/24` |
| DNS resolver | `8.8.8.8` |
| VirtualBox DHCP | Enabled |
| IPv6 in the NAT Network | Disabled in the shown configuration |

## Network layout

```text
                 Internet
                    |
             VirtualBox NAT
                    |
          NatNetwork: 10.0.0.0/24
             Gateway: 10.0.0.1
                    |
             Kali Linux VM
               interface: eth0
            intended IP: 10.0.0.2/24
              DNS: 8.8.8.8
```

## 1. Configure the VirtualBox NAT Network

1. Open **Oracle VirtualBox Manager**.
2. Open **File → Tools → Network** (the menu wording may vary by VirtualBox version), then select the **NAT Networks** tab.
3. Create or select a NAT Network with these settings:
   - **Name:** `NatNetwork`
   - **IPv4 Prefix:** `10.0.0.0/24`
   - **Enable DHCP:** checked
   - **Enable IPv6:** unchecked
4. Apply or save the settings.
5. Open the Kali VM's **Settings → Network**.
6. Under the attached adapter, select **NAT Network** and choose `NatNetwork`. Make sure the network adapter and **Cable Connected** options are enabled.

![VirtualBox NAT Network configuration](assets/virtualbox-nat-network.jpeg)

## 2. Configure IPv4 in Kali Linux

In Kali, open the network connection editor for **Wired connection 1** and select the **IPv4 Settings** tab.

Set the following values:

| Setting | Value |
|---|---|
| Method | Manual |
| Address | `10.0.0.2` |
| Netmask / prefix | `24` (`255.255.255.0`) |
| Gateway | `10.0.0.1` |
| DNS servers | `8.8.8.8` |

Save the profile. The screenshot below shows these values in the connection editor.

![Kali Linux manual IPv4 configuration](assets/kali-ipv4-manual.jpeg)

### Optional: configure the same values from the terminal

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns 8.8.8.8

sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

Bringing the connection down and back up briefly interrupts network connectivity.

> **Important:** Because DHCP is enabled on the NAT Network, make sure `10.0.0.2` is not also being assigned to another VM or device. Keep static addresses outside the DHCP allocation range, or configure an appropriate reservation/exclusion where supported.

## 3. Verify the network

Run these commands in Kali Linux:

```bash
# Show interface addresses
ip -br address

# Show the routing table
ip route

# Test outbound IP connectivity
ping -c 3 8.8.8.8

# Test DNS resolution
nslookup networkwalks.com
```

For the intended static configuration, confirm that:

- `eth0` has `10.0.0.2/24`.
- The default route uses gateway `10.0.0.1`.
- The DNS query uses `8.8.8.8` and returns an address for the domain.
- The ping test receives replies.

### Results captured in this lab

The terminal screenshot records the following successful tests:

- `ping -c 3 8.8.8.8`: **3 packets transmitted, 3 received, 0% packet loss**.
- `nslookup networkwalks.com`: DNS server `8.8.8.8` returned an address (`192.232.216.135`) at the time of the test. DNS answers can change, so this address is an example result, not a fixed value.

![Kali Linux network validation commands and output](assets/kali-network-validation.png)

## 4. Verification note: configured IP vs. active IP

There is one mismatch in the captured output that should be resolved before treating the static-IP setup as complete:

| Check | Captured value |
|---|---|
| Saved NetworkManager profile | `10.0.0.2/24`, method `manual` |
| Active `eth0` address (`ip -br a`) | `10.0.0.3/24` |
| Default route source | `10.0.0.3`, route marked `proto dhcp` |

This indicates that the saved profile shows the intended manual address, but the active interface was still using a DHCP-provided address when the terminal screenshot was taken. Internet connectivity and DNS resolution were working, but the screenshot does **not** prove that `10.0.0.2` was active.

To investigate and apply the intended profile, run:

```bash
# Check which NetworkManager connections are active
nmcli connection show --active

# Check device state
nmcli device status

# Inspect the saved profile
nmcli connection show "Wired connection 1" \
  | grep -E 'ipv4\.(method|addresses|gateway|dns)'

# Reactivate the intended profile
sudo nmcli connection up "Wired connection 1"

# Recheck the active address and route
ip -br address
ip route
```

If `eth0` still receives `10.0.0.3`, confirm that the VM is attached to `NatNetwork`, that the intended profile is active, and that no other NetworkManager profile is controlling `eth0`. Recheck the DHCP pool to avoid an address conflict before choosing a static address.

## Troubleshooting

- **No IP address:** Confirm the VirtualBox adapter is enabled, attached to `NatNetwork`, and has **Cable Connected** checked.
- **Wrong IP address:** Check active NetworkManager profiles with `nmcli connection show --active`, then reactivate the intended profile.
- **Can ping `8.8.8.8` but domain names fail:** Check the DNS setting with `nmcli connection show "Wired connection 1"` and test with `nslookup`.
- **Gateway is unreachable:** Confirm the guest address and prefix are in `10.0.0.0/24` and the gateway is `10.0.0.1`.
- **Intermittent connectivity after changing settings:** Reconnect the Kali network profile and verify the route again with `ip route`.

## Notes

- This is a basic connectivity lab, not a production network configuration.
- The DNS resolver `8.8.8.8` is a public Google Public DNS address; you can replace it with a resolver approved for your environment.
- Use this isolated lab only for systems and networks you own or are authorized to test.

## Repository contents

```text
.
├── README.md
└── assets/
    ├── virtualbox-nat-network.jpeg
    ├── kali-ipv4-manual.jpeg
    └── kali-network-validation.png
```
