## Block outbound IPv6 traffic to specific sites (Yandex)

To block these subnets (the **Yandex** address ranges) on an ASUS RT-N66U router, you must add 
rules to the beginning of the `FORWARD` chain (for LAN devices) and the `OUTPUT` chain (for the router itself).

**Critical Note:** The `2a02:6b8:23::/48` subnet is a subset of `2a02:6b8::/32`. Therefore, technically, 
blocking only the **`/32`** prefix is sufficient to automatically cover the second subnet.

The following instructions detail the creation of a persistent script that survives reboots.

### Step 1. Connect to the Router
Access the router via SSH (as performed previously).

### Step 2. Create the Autostart Script
In Asuswrt-Merlin 380.xx firmware, custom `ip6tables` rules must be defined in `/jffs/scripts/firewall-start`.

Execute the following command to create or overwrite the script file (copy the entire block and paste it 
into the console):

```bash
cat << 'EOF' > /jffs/scripts/firewall-start
#!/bin/sh

# List of subnets to block
# 2a02:6b8::/32 - Full Yandex range

NET1="2a02:6b8::/32"

# 1. Clean up old rules (prevents duplication on firewall restart)
# Suppress errors (2> /dev/null) if rules do not exist yet
ip6tables -D FORWARD -d $NET1 -j REJECT 2> /dev/null
ip6tables -D OUTPUT -d $NET1 -j REJECT 2> /dev/null

# 2. Block for LAN devices (LAN -> WAN)
# Use -I (Insert) to prioritize rules at the top of the chain
ip6tables -I FORWARD -d $NET1 -j REJECT

# 3. Block for the router itself (Router -> WAN)
ip6tables -I OUTPUT -d $NET1 -j REJECT

EOF
```

### Step 3. Set Permissions and Execute
Make the file executable and run it manually to apply the rules immediately without a reboot.

```bash
chmod a+rx /jffs/scripts/firewall-start
/jffs/scripts/firewall-start
```

### Step 4. Verification
Ensure the rules appear at the top of the list (Chain FORWARD):

```bash
ip6tables -nL FORWARD -v
```

The output should show the following at the beginning of the list:
1.  REJECT ... `2a02:6b8:23::/48`
2.  REJECT ... `2a02:6b8::/32`
3.  ... followed by your remaining rules.

### Why `REJECT` instead of `DROP`?
In this scenario, `REJECT` is used so that the browser immediately returns a connection error 
rather than "hanging" while attempting to load the page. However, `DROP` is generally 
more appropriate for blocking telemetry or background services.
