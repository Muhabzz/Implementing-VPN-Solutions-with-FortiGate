Here’s a clean **`README.md`** version of your instructions, formatted properly for Markdown:


# Configure Linux Machine for LAN Access

Follow these steps to set a static IP on your Linux machine so it can access your LAN:

---

## 1. Open network interfaces file

Open the terminal and edit the network configuration file:

```bash
sudo nano /etc/network/interfaces
````

---

## 2. Add static IP configuration

Add the following lines at the end of the file:

```bash
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

> **Note:**
>
> * `eth0` is your network interface. Replace it if your interface has a different name (check with `ip a`).
> * `address` is the static IP you want to assign.
> * `gateway` should be your FortiGate LAN IP.

---

## 3. Restart networking

Apply the changes by restarting the network service:

```bash
sudo systemctl restart networking
```

---

## 4. Verify configuration

Check that the static IP is applied:

```bash
ip addr show eth0
ping 192.168.1.1
```

You should be able to reach your FortiGate LAN interface and access resources in your network.

```

