# Network & file transfer

Bringing up Ethernet on the board, connecting over SSH, and moving files between host and board.

## Variables

```bash
export BOARD_IP=192.168.1.5
export HOST_IP=192.168.1.3
export HOST_IF=enx080027effb07      # host interface connected to the board (see ip -br addr)
```

## Static IP on the board

Temporary (lost on reboot):

```bash
ip link set eth0 up
ip addr add 192.168.1.5/24 dev eth0
ip -br addr show eth0
ping -c 3 192.168.1.3
```

Persistent: first check which network setup the image uses:

```bash
ps -p 1 -o comm=                                   # systemd or init
ls /etc/systemd/network/ /etc/network/interfaces 2>/dev/null
```

=== "systemd-networkd"

    ```bash
    cat > /etc/systemd/network/10-eth0.network << 'EOF'
    [Match]
    Name=eth0

    [Network]
    Address=192.168.1.5/24
    Gateway=192.168.1.3
    DNS=8.8.8.8
    EOF
    systemctl restart systemd-networkd
    ```

=== "ifupdown (/etc/network/interfaces)"

    ```bash
    cat >> /etc/network/interfaces << 'EOF'

    auto eth0
    iface eth0 inet static
        address 192.168.1.5
        netmask 255.255.255.0
        gateway 192.168.1.3
    EOF
    ifdown eth0; ifup eth0
    ```

!!! note
    With NFS root, the kernel already configured `eth0` from the `ip=` boot argument. Do not change `eth0` settings then; the rootfs would disconnect.

## SSH

The image needs an SSH server (`dropbear` or `openssh`, e.g. `IMAGE_FEATURES += "ssh-server-openssh"`). With `debug-tweaks`, root has no password.

```bash
ssh root@$BOARD_IP
```

Log in without a password, using my key:

```bash
ssh-copy-id root@$BOARD_IP
```

Host alias in `~/.ssh/config`, so `ssh rz` is enough. The last two lines skip the host key check: the board gets a new key every time it is reflashed.

```
Host rz
    HostName 192.168.1.5
    User root
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

## Copy files

```bash
scp app root@$BOARD_IP:/home/root/
scp root@$BOARD_IP:/var/log/messages .
scp -r mydir root@$BOARD_IP:/home/root/
```

Only changed files (needs `rsync` on the board):

```bash
rsync -av --progress build/ root@$BOARD_IP:/home/root/build/
```

Run a command remotely:

```bash
ssh root@$BOARD_IP "dmesg | tail -20"
```

## Mount a host directory on the board (NFS)

When the board boots from SD/eMMC but I want quick access to host files. Export the directory on the host as in [Booting with TFTP and NFS](../g2l/002nfs.md), then on the board:

```bash
mkdir -p /mnt/host
mount -t nfs -o nolock,vers=3 192.168.1.3:/nfs/share /mnt/host
```

## Internet access for the board through the host

Board connected directly to the host (no router). Share the host's internet connection with NAT.

On the host:

```bash
export WAN_IF=enp0s3       # host interface that has internet
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o $WAN_IF -j MASQUERADE
sudo iptables -A FORWARD -i $HOST_IF -o $WAN_IF -j ACCEPT
sudo iptables -A FORWARD -i $WAN_IF -o $HOST_IF -m state --state RELATED,ESTABLISHED -j ACCEPT
```

On the board:

```bash
ip route add default via 192.168.1.3
echo "nameserver 8.8.8.8" > /etc/resolv.conf
ping -c 3 google.com
```

## Check the link

```bash
ip -s link show eth0             # counters, errors
ethtool eth0                     # speed, duplex, link detected
```

Throughput (needs `iperf3` on both sides):

```bash
iperf3 -s                        # on the host
iperf3 -c 192.168.1.3 -t 10      # on the board
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `ping` fails both ways | Different subnets, cable on the other port (`eth1`), or link down: `ethtool eth0 \| grep Link` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | Board was reflashed and has a new host key. `ssh-keygen -R $BOARD_IP` |
| `Connection refused` on SSH | No SSH server in the image, or it has not started yet. Check `ps \| grep -E "sshd\|dropbear"` |
| SSH is slow to connect | Board tries a reverse DNS lookup. Add `UseDNS no` to `/etc/ssh/sshd_config` on the board |
| Board has no internet through the host | Forwarding or NAT rule missing on the host, or no default route / DNS on the board |
| Host VM cannot reach the board | VirtualBox adapter must be in *Bridged* mode on the physical interface connected to the board |
