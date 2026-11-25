# Palo Alto PA-VM: Destination NAT & Port Translation Lab

Publishing an internal IIS web site (Windows Server 2019) to the outside through a Palo Alto PA-VM firewall using **Destination NAT with port translation**.

![Topology](screenshots/00-topology.png)

## Objective

Make the internal **Portfolio** web site, which listens on `192.168.2.14:8080`, reachable from the WAN side at `http://192.168.1.25` (port 80), without exposing the internal address or the non-standard port.

## Lab Environment

| Component | Details |
|---|---|
| Firewall | Palo Alto PA-VM (PAN-OS 10.0.x) on VMware |
| Web server | Windows Server 2019 (`PDC19`) with IIS |
| Client | Windows 10 VM / external browser |
| WAN interface | `ethernet1/1` - `192.168.1.20/24` (zone `WAN`) |
| Published (public) IP | `192.168.1.25` |
| Internal web server | `192.168.2.14` (zone `LAN`) |
| IIS site / binding | `Portfolio` - `http` - `192.168.2.14:8080` |

## Implementation Steps

### 1. Publish the site in IIS
Created the `Portfolio` site and bound it to `192.168.2.14` on port `8080`.

![IIS bindings](screenshots/03-iis-site-bindings.png)

### 2. Create the NAT policies
Two NAT rules (Policies > NAT):

| # | Name | Src zone | Dst zone | Dst interface | Source | Destination | Service | Translation |
|---|---|---|---|---|---|---|---|---|
| 1 | NAT | LAN | WAN | ethernet1/1 | 192.168.2.10, 192.168.2.14 | any | any | **Source:** dynamic-ip-and-port -> ethernet1/1 `192.168.1.20/24` |
| 2 | WAN TO LAN | WAN | WAN | ethernet1/1 | any | 192.168.1.25 | service-http | **Destination:** `192.168.2.14`, port `8080` |

> In a destination NAT rule the **destination zone is the zone of the pre-NAT (public) IP**, which is `WAN` here, and not the zone of the real server.

![NAT policy](screenshots/02-nat-policy.png)

### 3. Create the Security policies
Policies > Security:

| # | Name | Src zone | Dst zone | Destination | Service | Action |
|---|---|---|---|---|---|---|
| 1 | Internet | LAN | WAN | any | any | Allow |
| 2 | Publish Server | WAN | LAN | **192.168.1.25** | service-http | Allow |
| 3 | intrazone-default | any | (intrazone) | any | any | Allow |
| 4 | interzone-default | any | any | any | any | Deny |

> Security policy is evaluated against the **pre-NAT IP** (`192.168.1.25`) but the **post-NAT zone** (`LAN`). This is the most common reason a Destination NAT lab fails.

![Security policy](screenshots/01-security-policy.png)

### 4. Commit and test
After the commit, browsing to `http://192.168.1.25` from outside loads the internal portfolio site. The `WAN TO LAN` NAT rule hit counter increments, which confirms the traffic matched the rule.

![External access](screenshots/04-external-access-result.png)

## Traffic Flow

```
Client  ->  192.168.1.25:80   (WAN zone, pre-NAT)
        ->  NAT rule "WAN TO LAN": destination translated to 192.168.2.14:8080
        ->  Security rule "Publish Server": WAN -> LAN, dst 192.168.1.25, service-http, Allow
        ->  IIS "Portfolio" site on 192.168.2.14:8080
```

## Repository Structure

```
.
├── README.md
├── docs/
│   └── Destination-NAT-Port-Translation-Lab.pdf   # full report with screenshots
├── configs/
│   ├── nat-rules.txt          # PAN-OS set commands for the NAT rules
│   ├── security-rules.txt     # PAN-OS set commands for the security rules
│   └── iis-site-binding.txt   # IIS binding + PowerShell equivalent
└── screenshots/
    ├── 00-topology.png
    ├── 01-security-policy.png
    ├── 02-nat-policy.png
    ├── 03-iis-site-bindings.png
    └── 04-external-access-result.png
```

## Tools Used

Palo Alto PA-VM, VMware, Windows Server 2019, IIS Manager, Windows 10, Microsoft Edge.

## Key Takeaways

- Destination NAT rewrites the destination IP/port; the NAT rule zone is the **pre-NAT** zone.
- Security policy uses **pre-NAT address + post-NAT zone**.
- Port translation lets the server keep a non-standard port (8080) while clients use the standard HTTP port.
- The published IP must be in the WAN subnet so the firewall can answer ARP for it.
