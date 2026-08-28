# 우분투 26.04 `default_qdisc` 미적용 워크어라운드

2026-08-20 확인 / Ubuntu 26.04 LTS / kernel 7.0.0-27-generic

mlx5(100G VM) 2대, ixgbe(10G 물리) 1대에서 재현 및 해결 검증.

## 증상

`/etc/sysctl.d/` 로 `net.core.default_qdisc = fq` 를 지정하고 `sysctl --system` 을 해도 NIC 의 실제 qdisc 는 `pfifo_fast` 로 남습니다.

```
# sysctl -n net.core.default_qdisc
fq

# tc qdisc show dev ens16
qdisc mq 0: root
qdisc pfifo_fast 0: parent :8 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
qdisc pfifo_fast 0: parent :7 ...
... (tx 큐 개수만큼)
```

### `ip link show` 로는 안 보입니다

```
# ip link show ens16
2: ens16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP ...
```

`ip link` 의 `qdisc` 필드는 **루트 qdisc 하나만** 표시합니다. `mq` 는 패킷을 큐잉하지 않는 디스패처일 뿐이고, 실제 동작을 결정하는 것은 그 아래 TX 큐별 leaf qdisc 입니다. 정상/비정상 모두 `ip link` 에는 똑같이 `mq` 로 나옵니다.

**반드시 `tc qdisc show` 로 확인해야 합니다.**

## 원인

qdisc 는 **NIC 등록 시점**에 결정됩니다. 그때 `sch_fq` 모듈이 없으면 커널 빌트인인 `pfifo_fast` 로 폴백하고, 이후 모듈이 로드되거나 sysctl 이 적용돼도 **이미 붙은 qdisc 에는 소급 적용되지 않습니다.**

우분투 26.04 는 initrd 에서도 systemd 를 돌리므로 `systemd-sysctl` 이 두 번 실행되는데, 실제 루트에서의 실행은 NIC 등록보다 늦습니다.

```
[1.7s] initrd   : systemd-modules-load -> systemd-sysctl
[2.5s] kernel   : mlx5_core ens16: renamed from eth0     <- 여기서 qdisc 결정
[4.3s] 실제 루트 : systemd-modules-load -> systemd-sysctl  <- 너무 늦음
```

`sch_fq` 는 모듈입니다 (`CONFIG_NET_SCH_FQ=m`). 기본 상태에서는 부팅 후 6초쯤에야 로드됩니다.

## 해결

`update-initramfs` 는 `/etc/modules-load.d/` 와 `/etc/sysctl.d/` 를 **initrd 안으로 복사**합니다. 따라서 initrd 단계(NIC 등록 이전)에서 모듈 로드와 sysctl 적용을 모두 끝낼 수 있습니다.

```sh
echo sch_fq > /etc/modules-load.d/fq.conf      # initrd 안에서 모듈을 로드시킴
echo sch_fq >> /etc/initramfs-tools/modules    # initrd 에 모듈 파일(.ko)을 포함시킴
update-initramfs -u                            # 위 설정과 sysctl.d 를 initrd 에 반영
reboot
```

두 파일은 역할이 다르므로 **둘 다 필요합니다.**

전제: `/etc/sysctl.d/tune-*.conf` 에 `net.core.default_qdisc = fq` 가 살아 있어야 합니다(주석 처리되어 있으면 안 됩니다).

### 재부팅 전 확인

```sh
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'sch_fq\.ko|sysctl\.d/tune-|modules-load\.d/fq\.conf'
```

세 줄이 모두 나와야 합니다.

```
etc/modules-load.d/fq.conf
etc/sysctl.d/tune-100g.conf
usr/lib/modules/7.0.0-27-generic/kernel/net/sched/sch_fq.ko.zst
```

### 재부팅 후 확인

```sh
tc qdisc show dev <IFACE> | awk '$1=="qdisc"{c[$2]++} END{for(k in c) print k, c[k]}'
```

```
mq 1
fq 8      <- tx 큐 개수만큼 fq 면 정상
```

부팅 순서도 확인할 수 있습니다.

```sh
journalctl -b -o short-monotonic | grep -E 'Finished systemd-(modules-load|sysctl)'
journalctl -b -k -o short-monotonic | grep -E 'renamed from eth0|Network Connection'
```

## 검증 결과

| 호스트 | NIC | 드라이버 | tx 큐 | 결과 |
|---|---|---|---|---|
| ps-daej | ens16 100G | mlx5 | 8 | `mq 1` + `fq 8` |
| ps-daej2 | ens16 100G | mlx5 | 8 | `mq 1` + `fq 8` |
| ps-ulsn | enp1s0f0 10G | ixgbe | 4 | `mq 1` + `fq 4` |

ps-ulsn 은 물리 서버라 부팅이 느려 initrd 단계가 30초대에 실행되지만, 순서는 동일합니다.

```
[30.27s] initrd  : systemd-modules-load
[30.97s] initrd  : systemd-sysctl
[33.73s] kernel  : ixgbe Intel(R) 10 Gigabit Network Connection
```

## 되지 않는 방법들

### GRUB 커널 파라미터

```
GRUB_CMDLINE_LINUX="default_qdisc=fq"
```

커널이 인식하지 못하고 무시합니다.

```
Unknown kernel command line parameters "default_qdisc=fq", will be passed to user space.
```

커널 cmdline 으로 sysctl 을 설정하려면 `sysctl.` 접두사가 필요합니다(`sysctl.net.core.default_qdisc=fq`, Linux 5.8+). 다만 그 경우에도 `sch_fq` 가 initrd 에 있어야 하므로 결국 위 방법이 필요합니다.

### `/etc/modules-load.d/` 만 추가

`systemd-modules-load.service` 는 실제 루트에서 실행되므로 NIC 등록보다 늦습니다. `update-initramfs -u` 를 함께 해야 initrd 로 복사되어 의미가 생깁니다.

### `/etc/initramfs-tools/modules` 만 추가

`update-initramfs -u` 를 실행하지 않으면 initrd 에 반영되지 않습니다. 파일에 `sch_fq` 가 적혀 있어도 initrd 안에는 없는 상태가 됩니다.

## 런타임 즉시 적용

재부팅 없이 적용하려면 `mq` 를 유지한 채 자식에만 걸어야 합니다.

```sh
IF=ens16
n=$(ls -d /sys/class/net/$IF/queues/tx-* | wc -l)
tc qdisc replace dev $IF root handle 1: mq
for i in $(seq 1 $n); do tc qdisc replace dev $IF parent 1:$i fq; done
```

`parent :N` 형식은 동작하지 않습니다. 커널이 자동 생성한 `mq` 의 핸들이 `0:` 이라 자식을 지정할 수 없어 `Error: Failed to find specified qdisc` 가 납니다. 그래서 `root handle 1: mq` 로 루트를 다시 만든 뒤 `parent 1:N` 을 씁니다.

> **`tc qdisc replace dev $IF root fq` 는 쓰지 마세요.**
> `mq` 루트가 단일 `fq` 로 교체되어 하드웨어 TX 큐 전부가 큐디스크 하나·락 하나로
> 직렬화됩니다. 100G 에서는 `pfifo_fast` 보다 나쁩니다.
> 이 상태가 되면 `tc qdisc show` 에 `mq` 없이 `qdisc fq ... root` 만 보입니다.
