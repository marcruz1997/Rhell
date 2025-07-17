
# ✅ DNS Server Configuration with BIND on RHEL 10 for OpenShift

This guide provides a complete step-by-step for setting up a local DNS server using BIND on RHEL 10. The configuration is intended for lab environments with OpenShift, allowing internal name resolution like `api.ocp.lab.local` and `*.apps.ocp.lab.local`.

---

## 🔧 Configuration Steps

### 1. Install required packages

```bash
sudo dnf install bind bind-utils -y
```

---

### 2. Edit the main BIND configuration file

```bash
sudo vim /etc/named.conf
```

Modify or add the following directives:

```conf
listen-on port 53 { any; };
allow-query     { any; };
recursion yes;
```

Add DNS zones at the end of the file:

```conf
zone "ocp.lab.local" IN {
    type master;
    file "/var/named/ocp.lab.local.db";
    allow-update { none; };
};

zone "10.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/192.168.10.rev";
    allow-update { none; };
};
```

---

### 3. Create zone files

#### Forward zone (ocp.lab.local):

```bash
sudo vim /var/named/ocp.lab.local.db
```

```dns
$TTL 1D
@       IN SOA  dns.ocp.lab.local. root.ocp.lab.local. (
                            2025071701 ; Serial
                            1D         ; Refresh
                            1H         ; Retry
                            1W         ; Expire
                            3H )       ; Minimum

        IN NS   dns.ocp.lab.local.

dns     IN A    192.168.10.10
api     IN A    192.168.10.100
apps    IN A    192.168.10.200
*.apps  IN A    192.168.10.200
```

#### Reverse zone (192.168.10.rev):

```bash
sudo vim /var/named/192.168.10.rev
```

```dns
$TTL 1D
@       IN SOA  dns.ocp.lab.local. root.ocp.lab.local. (
                            2025071701
                            1D
                            1H
                            1W
                            3H )
        IN NS   dns.ocp.lab.local.

10      IN PTR  dns.ocp.lab.local.
100     IN PTR  api.ocp.lab.local.
200     IN PTR  apps.ocp.lab.local.
```

---

### 4. Adjust permissions and SELinux context

```bash
sudo chown root:named /var/named/*.db
sudo restorecon -v /var/named/*.db
```

---

### 5. Start and enable the BIND service

```bash
sudo systemctl enable --now named
systemctl status named
```

---

### 6. Open DNS port in the firewall

```bash
sudo firewall-cmd --add-service=dns --permanent
sudo firewall-cmd --reload
```

---

### 7. Test DNS resolution

Forward lookup:

```bash
dig @localhost api.ocp.lab.local
```

Reverse lookup:

```bash
dig -x 192.168.10.100 @localhost
```

---

### 8. Configure clients to use this DNS

Edit `/etc/resolv.conf`:

```conf
nameserver 192.168.10.10
search ocp.lab.local
```

To make it permanent with NetworkManager:

```bash
nmcli con mod <connection-name> ipv4.dns "192.168.10.10"
nmcli con mod <connection-name> ipv4.ignore-auto-dns yes
nmcli con up <connection-name>
```

---

## 📌 Final Considerations

- The data entered in the files was created according to the test environment. Gather all necessary information before filling in the zone files.
- This local DNS server is ideal for OpenShift labs in closed environments.
- Use `*.apps.ocp.lab.local` if you haven't yet defined a domain for your application, server, cluster, etc.
- Check logs with:
- In this example, DNS configuration was performed for an OpenShift Cluster.

```bash
journalctl -u named
```

- Validate the zone files:

```bash
named-checkzone ocp.lab.local /var/named/ocp.lab.local.db
named-checkzone 10.168.192.in-addr.arpa /var/named/192.168.10.rev
```