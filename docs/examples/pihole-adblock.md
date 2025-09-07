# PiHole Adblocker Example with Slicer

PiHole is a common adblocker for home networks. This example shows how to run PiHole in a Slicer VM.

You can use a regular PC like an N100 or a Raspberry Pi.

## Userdata

The automated installation of PiHole users a custom userdata script.

Save this as `setup-pihole.sh`:

```bash
#!/usr/bin/env bash
set -euxo pipefail

# ---------- Tunables ----------
UPSTREAMS=("1.1.1.1" "9.9.9.9")   # change if you like
ENABLE_DNSSEC="true"              # "true" or "false"
ADMIN_PASS=""                     # leave empty to auto-generate
# -------------------------------

retry() { i=0; while true; do if "$@"; then return 0; fi; i=$((i+1)); [ "$i" -ge 30 ] && return 1; sleep 2; done; }
wait_default_route() { for _ in $(seq 1 60); do /usr/sbin/ip -4 route show default | /usr/bin/grep -q . && return 0; sleep 1; done; return 1; }

# 1) Network ready before any fetches
wait_default_route

# 2) Bring up systemd-resolved (no stub on :53 to avoid conflict)
#    and point resolv.conf at resolved's generated file
/usr/bin/mkdir -p /etc/systemd || :
/usr/bin/tee /etc/systemd/resolved.conf >/dev/null <<EOF
[Resolve]
DNS=${UPSTREAMS[*]}
FallbackDNS=1.0.0.1 149.112.112.112
DNSStubListener=no
EOF
/bin/systemctl unmask systemd-resolved || :
/bin/systemctl enable --now systemd-resolved || :
if [ -f /run/systemd/resolve/resolv.conf ]; then
  chattr -i /etc/resolv.conf || :
  chattr -i /etc/systemd/resolved.conf || :

  /bin/ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
else
  # fallback if resolved isn't providing it for some reason
  /usr/bin/tee /etc/resolv.conf >/dev/null <<EOF
$(for s in "${UPSTREAMS[@]}"; do echo "nameserver $s"; done)
options timeout:2 attempts:2
EOF
fi

# 3) Speed up apt (prefer IPv4)
#    NOTE: do this BEFORE update to avoid AAAA timeouts
/usr/bin/tee /etc/apt/apt.conf.d/99force-ipv4 >/dev/null <<'EOF'
Acquire::ForceIPv4 "true";
EOF

# 4) Base tools
retry /usr/bin/apt-get update
DEBIAN_FRONTEND=noninteractive retry /usr/bin/apt-get install -y \
  curl ca-certificates iproute2 procps e2fsprogs openssl dnsutils || :

# 5) Detect primary interface + IP/CIDR
IFACE="$(/usr/sbin/ip -o -4 route show to default | /usr/bin/awk '{print $5}' | /usr/bin/head -n1)"
IP_CIDR="$(/usr/sbin/ip -o -4 addr show dev "${IFACE}" | /usr/bin/awk '{print $4}' | /usr/bin/head -n1)"
VM_IP="$(echo "$IP_CIDR" | /usr/bin/cut -d/ -f1)"

# 6) Preseed Pi-hole v6 config (TOML + setupVars)
/usr/bin/mkdir -p /etc/pihole
# pihole.toml
/usr/bin/tee /etc/pihole/pihole.toml >/dev/null <<EOF
[dns]
interface = "0.0.0.0"
upstreams = [$(printf '"%s",' "${UPSTREAMS[@]}" | sed 's/,$//')]
dnssec = ${ENABLE_DNSSEC}
EOF

# setupVars.conf (some installers still consult it)
/usr/bin/tee /etc/pihole/setupVars.conf >/dev/null <<EOF
PIHOLE_INTERFACE=${IFACE}
IPV4_ADDRESS=${IP_CIDR}
IPV6_ADDRESS=
DNSMASQ_LISTENING=single
PIHOLE_DNS_1=${UPSTREAMS[0]}
PIHOLE_DNS_2=${UPSTREAMS[1]:-${UPSTREAMS[0]}}
DNSSEC=${ENABLE_DNSSEC}
QUERY_LOGGING=true
INSTALL_WEB_SERVER=true
INSTALL_WEB_INTERFACE=true
LIGHTTPD_ENABLED=true
BLOCKING_ENABLED=true
WEBPASSWORD=
EOF

# 7) Unattended install
retry /usr/bin/curl -fsSL https://install.pi-hole.net -o /tmp/pihole-install.sh
PIHOLE_SKIP_OS_CHECK=true /bin/bash /tmp/pihole-install.sh --unattended || :

# 8) CLI path + admin password
PIHOLE_BIN="/usr/local/bin/pihole"; [ -x "$PIHOLE_BIN" ] || PIHOLE_BIN="/usr/bin/pihole"
if [ -z "${ADMIN_PASS}" ]; then ADMIN_PASS="$(/usr/bin/openssl rand -base64 18)"; fi
"${PIHOLE_BIN}" setpassword  "${ADMIN_PASS}" || :
"${PIHOLE_BIN}" -g || :

# 9) Ensure service enabled & started
/bin/systemctl enable pihole-FTL || :
/bin/systemctl restart pihole-FTL || :

# 10) Output & quick hints
/usr/bin/printf "Pi-hole admin: http://%s/admin\n" "${VM_IP}"
/usr/bin/printf "Pi-hole admin password: %s\n" "${ADMIN_PASS}" > /root/pihole-admin-pass.txt
/usr/bin/printf "Saved admin password to /root/pihole-admin-pass.txt\n"
echo "Test locally: dig +short openfaas.com @127.0.0.1 || true"
```

## VM configuration file

Next, set up a modest VM configuration file - use the example given in the [walkthrough](/getting-started/walkthrough).

Then under `host_groups`, add the following, and update the hostgroup name to avoid clashes.

```yaml
  host_groups:
  - name: pihole
    userdata_file: ./setup-pihole.sh
```

## Accessing PiHole

Watch the logs from the serial console to see when the install is complete.

```bash
arkade get fstail
sudo fstail /var/log/slicer/
```

Next, the PiHole admin interface will be available on port 80 of the VM's IP address.

This is typically `http://192.168.137.2/admin` and the admin password will have been printed to the logs found above.

Failing that, you can log into the VM using the SOS console or regular SSH and read the file at `/root/pihole-admin-pass.txt`.

Example DNS query:

Add nslookup and dig to your Slicer system if not already present

```bash
sudo apt update && \
    sudo apt install -qy dnsutils
```

Perform a lookup using PiHole as the DNS server:

```bash
# Should work fine
nslookup openfaas.com 192.168.137.2

# Should be blocked
nslookup doubleclick.net 192.168.137.2
```

To use PiHole from another machine, you have a couple of options:

1. Log into your router and add a static route for the VM's IP address via the slicer host. Then set the DNS server under the router's DHCP settings to the PiHole VM's IP address.
2. Enable DNAT via iptables from the Slicer host to the PiHole VM, then go into your home router and set the DNS server under the router's DHCP settings to the Slicer host's IP address.

Example DNAT rule for iptables:

```bash
# First, clear any existing DNS DNAT rules to avoid conflicts
# Optional step..
# sudo iptables -t nat -F PREROUTING

# Set your PiHole VM IP and detect the network interface
export PIHOLE_IP=192.168.136.2  # Update this to match your actual PiHole VM IP
export SLICER_IFACE=$(ip -o -4 route show to default | awk '{print $5}' | head -n1)

echo "Slicer host interface: $SLICER_IFACE"
echo "PiHole VM IP: $PIHOLE_IP"

# Add DNAT rules for DNS traffic coming from the external interface
sudo iptables -t nat -A PREROUTING -i $SLICER_IFACE -p udp --dport 53 -j DNAT --to-destination $PIHOLE_IP:53
sudo iptables -t nat -A PREROUTING -i $SLICER_IFACE -p tcp --dport 53 -j DNAT --to-destination $PIHOLE_IP:53

# Add MASQUERADE rule so return traffic works properly
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

If you'd also like a rule for the admin dashboard, so it's also accessible from the host IP:

```bash
# Set your PiHole VM IP and detect the network interface
export PIHOLE_IP=192.168.136.2  # Update this to match your actual PiHole VM IP
export SLICER_IFACE=$(ip -o -4 route show to default | awk '{print $5}' | head -n1)

sudo iptables -t nat -A PREROUTING -i $SLICER_IFACE -p tcp --dport 80 -j DNAT --to-destination $PIHOLE_IP:80
```
