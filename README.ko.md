# docker-compose for perfSONAR testpoint

[🇺🇸 English](README.md) · [🇰🇷 한국어](README.ko.md)

`docker compose` 로 네트워크 성능측정 인프라 [perfsonar-testpoint](https://docs.perfsonar.net/install_options.html) 를 손쉽게 구축합니다.

## 1. 요구사항

* 우분투 24.04 이상 (cgroup v2 기반 OS)
* [Docker CE](https://docs.docker.com/engine/install/ubuntu/)
    - 공식 웹사이트 문서를 참고하여 최신으로 설치
    - `docker` 20.10 이상, `docker compose` 2.16 이상 (최신이면 만족)

## 2. 설치

```sh
git clone https://<내부-git>/perfsonar-testpoint-docker.git /opt/ps5
cd /opt/ps5
```

### 2.1. 리눅스 네트워크 커널 튜닝

인터페이스 속도에 맞는 튜닝 설정을 적용합니다.

```sh
cp /opt/ps5/sysctl/tune-100g.conf /etc/sysctl.d/
# cp /opt/ps5/sysctl/tune-10g.conf  /etc/sysctl.d/
# cp /opt/ps5/sysctl/tune-1g.conf   /etc/sysctl.d/

sysctl --system
```

### 2.2. Chrony 설정

편도지연(one way latency)을 측정하기 위해 NTP 서버와 동기화 설정을 합니다.

```sh
# 1) 우분투 기본 소스 비활성화 (필수)
mv /etc/chrony/sources.d/ubuntu-ntp-pools.sources \
   /etc/chrony/sources.d/ubuntu-ntp-pools.sources.disabled

# 2) NTP 서버 지정
cp /opt/ps5/chrony/sources.d/sample.sources /etc/chrony/sources.d/my.sources
vi /etc/chrony/sources.d/my.sources

# 3) 튜닝 파라미터 설정
cp /opt/ps5/chrony/conf.d/tune.conf /etc/chrony/conf.d/

systemctl restart chrony
```

### 2.3. 노출 정보 설정

```sh
# https://<host>/ 에 보일 안내 문구 설정
vi /opt/ps5/html/index.html

# Lookup Service 등록 정보 설정
# https://stats.perfsonar.net 에서 확인 가능
vi /opt/ps5/perfsonar/lsregistrationdaemon.conf
```

### 2.4. 방화벽 설정

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

### 2.5. (선택) Lookup Service 등록 비활성화

stats.perfsonar.net 에 노드를 노출하지 않으려면, `perfsonar-lsregistrationdaemon.service` 를 비활성화 합니다.

레포에 포함된 빈 파일 `empty.service` 를 유닛 위에 덮어씌우면 됩니다. `docker-compose.yml` 에서 다음 주석을 해제합니다.

```yaml
    volumes:
      - ./empty.service:/etc/systemd/system/multi-user.target.wants/perfsonar-lsregistrationdaemon.service
```

### 2.6. (선택) 테스트 제한

외부에서 들어오는 테스트의 시간·대역폭을 제한하려면 다음 문서를 참고하여 `pscheduler/limits.conf` 를 설정합니다. - [Configuring pScheduler Limits](https://docs.perfsonar.net/config_pscheduler_limits.html)

그리고 `docker-compose.yml` 에서 다음 주석을 해제합니다.

```yaml
    volumes:
      - ./pscheduler/limits.conf:/etc/pscheduler/limits.conf:z
```

### 2.7. 실행

```sh
docker compose up -d
```

## 3. 검증

동작확인

```sh
docker compose ps
docker exec perfsonar-testpoint systemctl list-units --failed
docker exec perfsonar-testpoint pscheduler troubleshoot
docker exec perfsonar-testpoint pscheduler ping OTHER_PERFSONAR_HOST
```

버전 확인

```sh
curl -s -k https://localhost/perfsonar_host_exporter/ | grep perfsonar_bundle
# perfsonar_bundle{type="perfsonar-testpoint",version="5.2.6"} 1
```

## 4. 운영

```sh
# 중지
docker compose down

# 버전 갱신
docker compose pull && docker compose up -d
```

## 5. 활용

`docker exec -it` 명령을 사용하여 perfsonar 명령어들을 실행합니다.

```sh
docker exec -it perfsonar-testpoint bash
docker exec -it perfsonar-testpoint pscheduler monitor
docker exec -it perfsonar-testpoint iperf3 -s -i 1
docker exec -it perfsonar-testpoint iperf3 -c OTHER_PERFSONAR_HOST -i 1 -t 10 -P 4
```

## 6. 부록

### 6.1. alias 적용

`docker exec -it` 명령이 번거러운 경우 쉘에 alias를 등록하면 편리하게 이용할 수 있습니다.

`~/.zshrc` 또는 `~/.bashrc`에 다음을 등록하고 적용합니다. 적용은 쉘 로그아웃/로그인 또는 `source ~/.zshrc`을 합니다.
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

이후 다음과 같이 이용 가능합니다.
```sh
pscheduler monitor
iperf3 -s -i 1
iperf3 -c ps.my.net -i 1 -t 10 -P 4
```

### 6.2. 커널 튜닝 확인

컨테이너가 `network_mode: host` 모드로 동작하므로 호스트와 컨테이너의 커널 파라미터가 동일해야 합니다. 다음과 같이 확인합니다.

```sh
# 호스트 커널 파라미터
sysctl net.ipv4.tcp_rmem net.core.rmem_max net.ipv4.tcp_congestion_control net.core.default_qdisc

# 컨테이너 커널 파라미터
docker exec perfsonar-testpoint \
sysctl net.ipv4.tcp_rmem net.core.rmem_max net.ipv4.tcp_congestion_control net.core.default_qdisc
```

### 6.3. Chrony 확인

```sh
# Root delay 가 1ms 미만이면 정상
chronyc tracking

# '^*' 가 지정한 서버인지 확인
chronyc -n sources

# 하드웨어 타임스탬핑 동작 여부 확인
chronyc ntpdata | grep 'Total HW'
```

### 6.4. 우분투 26.04 임시 패치 (default_qdisc)

우분투 26.04 에는 `/etc/sysctl.d/` 로 지정한 `net.core.default_qdisc` 가 NIC 에 적용되지 않고 `pfifo_fast` 로 남는 문제가 있습니다 (2026-08-20 확인). 다음 임시조치가 필요합니다.

```sh
echo sch_fq > /etc/modules-load.d/fq.conf
echo sch_fq >> /etc/initramfs-tools/modules
update-initramfs -u
reboot
```

확인은 `ip link show` 가 아니라 `tc qdisc show dev <IFACE>` 로 합니다. tx 큐 개수만큼 `fq` 가 보이면 정상입니다.

원인, 검증 방법, 되지 않는 방법(GRUB 파라미터 등)은 [docs/ubuntu26-fq-workaround.md](docs/ubuntu26-fq-workaround.ko.md) 를 참고하세요.
