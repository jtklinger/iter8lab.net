---
title: "Replacing Dual Headscale Tailnets with a Unified Netbird Mesh"
date: 2026-03-30
draft: false
tags: ["netbird", "headscale", "tailscale", "vpn", "wireguard", "homelab"]
categories: ["Infrastructure"]
summary: "How I replaced two independent Headscale tailnets with a single Netbird mesh VPN, eliminating profile switching and simplifying network access across two domains."
---

## The Problem

I run two domains across separate VLANs — a production homelab on `lab.towerbancorp.com` (VLAN 100) and family services on `ourhomeport.com` (VLAN 150). For remote access, I had two independent Headscale instances running on GCP, each managing its own tailnet.

The problem was simple but constant: my desktop could only connect to one tailnet at a time. Working on Vaultwarden? Switch to the lab profile. Need to check Authentik? Switch to OHP. Tailscale doesn't support connecting to two control servers simultaneously, so every time I crossed domains, I had to manually switch profiles and wait for the connection to re-establish.

MCP SSH tools, DNS resolution, and service access all depended on which tailnet I was connected to. It was workable but annoying — and it completely blocked the future goal of onboarding family devices that need access to services on both networks.

## Why Netbird

I evaluated three options:

**Single Headscale with ACLs** would have been the least disruptive — merge both tailnets, use ACL policies for segregation. But it doesn't solve the fundamental problem: Tailscale's client model is one-control-server-per-device. ACLs can restrict who talks to whom, but the desktop would still be in a single mesh with a single DNS configuration.

**ZeroTier** natively supports multiple networks per device, which would have solved the switching problem directly. But SSO/OIDC is only available on their paid cloud tier — not on the self-hosted controller. For a homelab that's heading toward Authentik-based family onboarding, that was a dealbreaker.

**Netbird** uses a single mesh with groups and access policies for segregation. Every device joins one network, but policies control what each device can reach. My desktop goes in the `admin-devices` group with access to everything. Lab servers go in `lab-servers`. OHP services go in `ohp-servers`. No switching, no profiles — just policies.

The clincher was Netbird v0.62's built-in local user management. Previous versions required an external identity provider just to get started, which is what tripped me up on an earlier attempt. Now you can deploy with local users first and add SSO later.

## Architecture

The deployment is a single GCP e2-small VM (2GB RAM — Netbird's minimum) running the management server, signal server, relay, dashboard, and embedded STUN via Docker Compose.

### Groups and Policies

| Group | Members | Purpose |
|-------|---------|---------|
| admin-devices | Desktop, laptop, phone | Full access to everything |
| lab-servers | kvm01, kvm02, ns1, storage, backup, smtp | Lab infrastructure |
| ohp-servers | server01 | OurHomePort services |
| work-devices | Work laptop | Lab access only |
| family-devices | (future) | Restricted family access |

The access policies are straightforward:

- `admin-devices` ↔ `lab-servers` (bidirectional)
- `admin-devices` ↔ `ohp-servers` (bidirectional)
- `lab-servers` ↔ `lab-servers` (intra-group)
- `ohp-servers` ↔ `ohp-servers` (intra-group)
- `work-devices` → `lab-servers` (lab only, no OHP)
- Default all-to-all policy: **disabled**

No policy exists between `lab-servers` and `ohp-servers`, so they're isolated by default. I verified this by pinging server01 from kvm01 over the Netbird mesh — 100% packet loss, exactly as intended.

### Subnet Routes

Netbird peers get their own mesh IPs (100.69.x.x), but I also need to reach services on the physical VLANs. Two subnet routes handle this:

- **kvm01** routes `192.168.100.0/24` (VLAN 100) — distributed to admin and work groups
- **server01** routes `192.168.150.0/24` (VLAN 150) — distributed to admin group only

From my desktop, I can now ping `192.168.100.106` (kvm02) and `192.168.150.100` (server01) simultaneously. No switching.

## The Migration

The key principle was **no disruption during migration**. Netbird uses the `wt0` WireGuard interface while Tailscale uses `tailscale0` — they coexist without conflict. I ran both VPN systems in parallel throughout the entire process.

### Phase 1: Server Deployment

Created the GCP VM, installed Docker, ran the Netbird quickstart script, obtained a Let's Encrypt certificate. The quickstart handles most of the setup — management server, signal, relay, and dashboard all come up in a single Docker Compose stack.

### Phase 2: Pilot

Enrolled three devices first — desktop, kvm01, and server01. This was the minimum to prove cross-VLAN access worked. The API made group creation, policy setup, and setup key generation scriptable:

```bash
# Create a group
curl -X POST "$NB_API/groups" \
  -H "Authorization: Token $TOKEN" \
  -d '{"name":"lab-servers"}'

# Create an access policy
curl -X POST "$NB_API/policies" \
  -H "Authorization: Token $TOKEN" \
  -d '{"name":"Admin to Lab","enabled":true,
       "rules":[{"sources":["admin-id"],
                 "destinations":["lab-id"],
                 "bidirectional":true,
                 "protocol":"all",
                 "action":"accept"}]}'
```

Client installation on Rocky Linux is a one-liner:

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sudo sh
sudo netbird up --setup-key <KEY> \
  --management-url https://netbird.ourhomeport.com
```

### Phase 3: Full Enrollment

Once the pilot verified cross-VLAN access, I enrolled all remaining servers in parallel via SSH. 12 peers total — 8 lab servers, 1 OHP server, desktop, laptop, and phone. The work laptop got its own `work-devices` group with a separate setup key that only grants lab access.

One hiccup: kvm02 (Rocky Linux 10) had a full root disk at 100%. Cleared 9.5GB of rotated `/var/log/messages` files to make room for the install.

### Phase 4: Headscale Removal

With all devices on Netbird, I removed Tailscale from every server. Two servers (kvm01 and server01) refused the `tailscale down` command because the MCP SSH session was routed through Tailscale — it detected it would kill its own connection. The `--accept-risk=lose-ssh` flag resolved that, and the sessions seamlessly fell over to Netbird routing.

After Tailscale removal, `/etc/resolv.conf` reverted from Tailscale's MagicDNS (`100.100.100.100`) to NetworkManager defaults. Most servers picked up ns1 (192.168.100.22) correctly; a few needed their DNS reconfigured via `nmcli`.

Both Headscale GCP VMs were deleted, static IPs released, and the kvm02 HA standby (failover scripts, DB sync cron, nginx config) was cleaned up.

## DNS Simplification

The Netbird migration triggered a broader DNS cleanup. The original setup had four layers of DNS — public Google Cloud DNS, private BIND on ns1, Headscale MagicDNS with split DNS rules, and ControlD on the Ubiquiti router with its own split DNS rules.

With Headscale gone and Netbird not needing split DNS, I moved the entire `lab.towerbancorp.com` forward zone from BIND to Google Cloud DNS. The records resolve to private IPs (192.168.100.x) via public DNS — anyone can resolve them, but only on-network or Netbird-connected devices can reach them. DNS is not a security layer.

This eliminated:
- The Headscale MagicDNS layer entirely
- The ControlD split DNS rules on the router (lab queries no longer need special routing to ns1)
- The Netbird nameserver group for lab DNS (public DNS handles it)
- BIND's forward zone (ns1 now only serves reverse PTR records)

The `ourhomeport.com` DNS records were also updated from Tailscale IPs to physical IPs — `auth.ourhomeport.com` now resolves to `192.168.150.100` instead of `100.64.0.2`.

## Results

| Before | After |
|--------|-------|
| 2 Headscale instances on 2 GCP VMs | 1 Netbird instance on 1 GCP VM |
| Manual profile switching between tailnets | Single mesh, both VLANs accessible simultaneously |
| MagicDNS + split DNS + BIND forward zone | Public DNS only, BIND reverse-only |
| Auth-key device enrollment | Setup keys with auto-group assignment (SSO planned) |
| ~$14/mo (2x e2-micro) | ~$13/mo (1x e2-small) |

The connection quality varies by peer — server01 connects P2P at ~1ms latency, while kvm01 is relayed at ~80ms. The relay is through the Netbird server in GCP, which adds a round trip. Investigating P2P connectivity for kvm01 is on the list.

## What's Next

- **Authentik SSO integration** — Netbird supports OIDC, and Authentik is already running. This will enable family device enrollment via "Sign in with Authentik" instead of setup keys.
- **Family device onboarding** — The `family-devices` group is created but empty. Once SSO is working, onboarding becomes: install Netbird, sign in, connect.
- **kvm01 P2P investigation** — Currently relayed. Getting this to P2P would drop latency from 80ms to ~1ms.
