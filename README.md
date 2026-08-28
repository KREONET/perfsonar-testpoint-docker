# docker-compose for perfSONAR testpoint

[🇺🇸 English](README.md) · [🇰🇷 한국어](README.ko.md)

Deploy the network measurement infrastructure [perfsonar-testpoint](https://docs.perfsonar.net/install_options.html) easily with `docker compose`.

## 1. Prerequisites

* Ubuntu 24.04 or later (cgroup v2 based OS)
* [Docker CE](https://docs.docker.com/engine/install/ubuntu/)
    - Install the latest version by following the official documentation
    - `docker` 20.10+, `docker compose` 2.16+ (any recent release satisfies this)

## 2. Installation

```sh
git clone https://<your-git>/perfsonar-testpoint-docker.git /opt/ps5
cd /opt/ps5
```

### 2.1. Linux network kernel tuning

Apply the tuning profile that matches your interface speed.

```sh
cp /opt/ps5/sysctl/tune-100g.conf /etc/sysctl.d/
# cp /opt/ps5/sysctl/tune-10g.conf  /etc/sysctl.d/
# cp /opt/ps5/sysctl/tune-1g.conf   /etc/sysctl.d/

sysctl --system
```

### 2.2. Chrony configuration

Synchronize the clock with NTP servers so that one-way latency can be measured.

```sh
# 1) Disable the Ubuntu default sources (required)
mv /etc/chrony/sources.d/ubuntu-ntp-pools.sources \
   /etc/chrony/sources.d/ubuntu-ntp-pools.sources.disabled

# 2) Specify your NTP servers
cp /opt/ps5/chrony/sources.d/sample.sources /etc/chrony/sources.d/my.sources
vi /etc/chrony/sources.d/my.sources

# 3) Apply tuning parameters
cp /opt/ps5/chrony/conf.d/tune.conf /etc/chrony/conf.d/

systemctl restart chrony
```

### 2.3. Published information

```sh
# Landing page shown at https://<host>/
vi /opt/ps5/html/index.html

# Lookup Service registration details
# Visible at https://stats.perfsonar.net
vi /opt/ps5/perfsonar/lsregistrationdaemon.conf
```

### 2.4. Firewall

```sh
ufw allow 443/tcp         comment 'perfSONAR - pScheduler API'
ufw allow 123/udp         comment 'perfSONAR - NTP'
ufw allow 33434:33634/udp comment 'perfSONAR - traceroute'
ufw allow 861/tcp         comment 'perfSONAR - owamp control'
ufw allow 8760:9960/udp   comment 'perfSONAR - owamp test'
ufw allow 8760:9960/tcp   comment 'perfSONAR - owamp test'
ufw allow 862/tcp         comment 'perfSONAR - twamp control'
ufw allow 18760:19960/udp comment 'perfSONAR - twamp test'
ufw allow 18760:19960/tcp comment 'perfSONAR - twamp test'
ufw allow 5201/tcp        comment 'perfSONAR - iperf3, ethr'
ufw allow 5001/tcp        comment 'perfSONAR - iperf2, nuttcp control'
ufw allow 5101/tcp        comment 'perfSONAR - nuttcp data'
ufw allow 5890:5900/tcp   comment 'perfSONAR - simplestream'
```

### 2.5. (Optional) Disable Lookup Service registration

To keep the node from being published on stats.perfsonar.net, disable `perfsonar-lsregistrationdaemon.service`.

Bind-mount the empty `empty.service` file shipped in this repository over the unit. Uncomment the following line in `docker-compose.yml`.

```yaml
    volumes:
      - ./empty.service:/etc/systemd/system/multi-user.target.wants/perfsonar-lsregistrationdaemon.service
```

### 2.6. (Optional) Test limits

To restrict the duration and bandwidth of incoming tests, configure `pscheduler/limits.conf` following - [Configuring pScheduler Limits](https://docs.perfsonar.net/config_pscheduler_limits.html).

Then uncomment the following line in `docker-compose.yml`.

```yaml
    volumes:
      - ./pscheduler/limits.conf:/etc/pscheduler/limits.conf:z
```

### 2.7. Start

```sh
docker compose up -d
```

## 3. Verification

Health check

```sh
docker compose ps
docker exec perfsonar-testpoint systemctl list-units --failed
docker exec perfsonar-testpoint pscheduler troubleshoot
docker exec perfsonar-testpoint pscheduler ping OTHER_PERFSONAR_HOST
```

Version check

```sh
curl -s -k https://localhost/perfsonar_host_exporter/ | grep perfsonar_bundle
# perfsonar_bundle{type="perfsonar-testpoint",version="5.2.6"} 1
```

## 4. Operations

```sh
# Stop
docker compose down

# Update to a newer image
docker compose pull && docker compose up -d
```

## 5. Usage

Run perfSONAR commands through `docker exec -it`.

```sh
docker exec -it perfsonar-testpoint bash
docker exec -it perfsonar-testpoint pscheduler monitor
docker exec -it perfsonar-testpoint iperf3 -s -i 1
docker exec -it perfsonar-testpoint iperf3 -c OTHER_PERFSONAR_HOST -i 1 -t 10 -P 4
```

## 6. Appendix

### 6.1. Shell aliases

If typing `docker exec -it` every time is tedious, register shell aliases.

Add the following to `~/.zshrc` or `~/.bashrc`, then log out and back in, or run `source ~/.zshrc`.

```sh
alias pscheduler='docker exec -it perfsonar-testpoint pscheduler'
alias psconfig='docker exec -it perfsonar-testpoint psconfig'
alias iperf='docker exec -it perfsonar-testpoint iperf'
alias iperf3='docker exec -it perfsonar-testpoint iperf3'
alias nuttcp='docker exec -it perfsonar-testpoint nuttcp'
alias owping='docker exec -it perfsonar-testpoint owping'
alias twping='docker exec -it perfsonar-testpoint twping'
alias traceroute='docker exec -it perfsonar-testpoint traceroute'
alias traceroute6='docker exec -it perfsonar-testpoint traceroute6'
```

They can then be used directly.

```sh
pscheduler monitor
iperf3 -s -i 1
iperf3 -c ps.my.net -i 1 -t 10 -P 4
```

### 6.2. Verifying kernel tuning

The container runs with `network_mode: host`, so the host and the container must report identical kernel parameters. Verify as follows.

```sh
# Host kernel parameters
sysctl net.ipv4.tcp_rmem net.core.rmem_max net.ipv4.tcp_congestion_control net.core.default_qdisc

# Container kernel parameters
docker exec perfsonar-testpoint \
sysctl net.ipv4.tcp_rmem net.core.rmem_max net.ipv4.tcp_congestion_control net.core.default_qdisc
```

### 6.3. Verifying chrony

```sh
# Root delay below 1 ms is healthy
chronyc tracking

# Confirm '^*' points at the server you configured
chronyc -n sources

# Check whether hardware timestamping is active
chronyc ntpdata | grep 'Total HW'
```

### 6.4. Ubuntu 26.04 workaround (default_qdisc)

On Ubuntu 26.04, `net.core.default_qdisc` set through `/etc/sysctl.d/` is not applied to the NIC and it stays on `pfifo_fast` (confirmed 2026-08-20). The following workaround is required.

```sh
echo sch_fq > /etc/modules-load.d/fq.conf
echo sch_fq >> /etc/initramfs-tools/modules
update-initramfs -u
reboot
```

Verify with `tc qdisc show dev <IFACE>`, not `ip link show`. It is healthy when `fq` appears once per TX queue.

For the root cause, verification steps and approaches that do *not* work (GRUB parameters and others), see [docs/ubuntu26-fq-workaround.md](docs/ubuntu26-fq-workaround.ko.md) (Korean).
