---
name: ocp-infra
description: OCP UPI 설치 인프라 구성 스킬. Bastion 서버 DNS(bind)/HAProxy/httpd 설정, Pull Secret 미러링, install-config.yaml 생성, Ignition 파일 생성, RHCOS 노드별 coreos-installer+nmcli 명령어 자동 생성. "Bastion 설정", "DNS 구성", "HAProxy", "ignition 생성", "RHCOS 설치", "노드 설치 명령어", "OCP UPI 설치" 관련 요청 시 반드시 이 스킬을 사용할 것.
---

# OCP Infra Skill

실제 PoC 검증된 OCP UPI 설치 절차 기반. Bastion(10.1.10.174) 중심으로 전체 인프라를 구성한다.

## 클러스터 정보

```
Bastion: ens33=10.1.10.51(외부) / ens35=192.168.0.1(내부, NAT GW)
         DNS + HAProxy + Registry + httpd + NAT Gateway
Bootstrap: 192.168.0.10  설치 완료 후 HAProxy 설정에서 제거
Master 1:  192.168.0.11
Master 2:  192.168.0.12
Master 3:  192.168.0.13
Worker 1:  192.168.0.14
Worker 2:  192.168.0.15
GPU Worker: 192.168.0.16 (5월 추가 — 현재 DNS/HAProxy 주석 처리)

도메인: cpf.com / 클러스터: ocp / OCP: 4.21 (stable-4.21.6)
내부 서브넷: 192.168.0.0/24 / 게이트웨이: 192.168.0.1
IP 설정 파일: nodes-config.env (배포 전 source하여 사용)
```

## 이중 네트워크 핵심 포인트

- **외부 접근**: 개발 PC → 10.1.10.51 (Bastion ens33)
- **내부 통신**: 모든 OCP 노드 → 192.168.0.1 (Bastion ens35)
- **api.ocp.cpf.com** → 10.1.10.51 (외부용)
- **api-int.ocp.cpf.com** → 192.168.0.1 (OCP 노드 내부 통신용 — 필수)
- **NAT**: 내부 노드의 외부 인터넷 접근은 Bastion MASQUERADE 경유

## 제약 사항

- Bastion에는 Claude Console 미설치 (Vanilla 유지)
- 네트워크 설정은 nmcli만 사용 (nmtui 사용 금지)
- 재부팅은 `sudo reboot` (stop/start 불필요)
- 엔지니어가 노드에서 직접 실행하는 명령어를 노드별로 명확히 분리 제공
- 작업 전 반드시 `source nodes-config.env` 실행

## 설치 절차

### 0. nodes-config.env 로드

```bash
source ~/nodes-config.env
echo "Bastion 외부: $BASTION_EXT_IP / 내부: $BASTION_INT_IP"
```

### 0-1. Bastion 이중 NIC + NAT 설정

```bash
# ens35 내부 NIC 설정
nmcli con mod ens35 ipv4.addresses ${BASTION_INT_IP}/24 \
  ipv4.method manual connection.autoconnect yes
nmcli con up ens35

# IP 포워딩 + NAT
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf && sysctl -p
iptables -t nat -A POSTROUTING -s ${INTERNAL_SUBNET} -o ens33 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```

### 1. Bastion 사전 준비 (SSH 키 + TLS 인증서)

SSH 키 생성:
```bash
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
```

TLS 인증서 (SAN 필수 포함):
- `api.ocp.cpf.com`, `api-int.ocp.cpf.com`, `*.apps.ocp.cpf.com`
- IP: `10.1.10.174` (내부), `192.168.0.1` (외부)

```bash
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout root.key -out root.crt -config req-minimal.cnf -extensions 'v3_req'
```

### 2. DNS (bind)

필수 레코드:
- `api.ocp` → **10.1.10.51** (외부 접근)
- `api-int.ocp` → **192.168.0.1** (OCP 노드 내부 통신 — 반드시 내부 IP)
- `*.apps.ocp` → **10.1.10.51** (외부 접근)
- bootstrap/master1-3/worker1-2 → 192.168.0.10~15
- Reverse PTR: 192.168.0.x 서브넷 기준
- GPU Worker(192.168.0.16): 주석 처리 → 5월 활성화

검증:
```bash
sudo named-checkconf /etc/named.conf
sudo named-checkzone cpf.com /var/named/forward.zone
```

### 3. HAProxy

4개 프론트엔드 (외부 10.1.10.51 + 내부 192.168.0.1 동시 바인딩):
- `6443` (OCP API): `bind 10.1.10.51:6443` + `bind 192.168.0.1:6443`
- `22623` (Machine Config): 동일
- `80` (HTTP Ingress): `bind 10.1.10.51:80`
- `443` (HTTPS Ingress): `bind 10.1.10.51:443`

백엔드 서버: 모두 192.168.0.x (내부 IP)

SELinux 포트 허용 필수:
```bash
sudo setsebool -P haproxy_connect_any 1
sudo semanage port -a -t http_port_t -p tcp 6443
sudo semanage port -a -t http_port_t -p tcp 22623
```

> 설치 완료 후 Bootstrap(192.168.0.10) 항목을 HAProxy에서 제거한다.

### 4. Pull Secret 및 이미지 미러링

```bash
cat ~/pull-secret.txt | jq > ~/.docker/pull-secret.json
# Nexus auth와 병합 후 Mirror Registry로 미러링
```

> 상세 절차: `references/pull-secret-mirror.md`

### 5. install-config.yaml 및 Ignition 생성

```bash
./openshift-install create manifests --dir=./install-dir
./openshift-install create ignition-configs --dir=./install-dir
# bootstrap.ign → httpd로 호스팅
cp install-dir/bootstrap.ign /var/www/html/
```

### 6. 노드별 RHCOS 설치 (엔지니어 직접 실행)

각 노드마다 다음 2단계 명령어를 생성한다:
1. `nmcli` 명령어 (네트워크 설정)
2. `coreos-installer` 명령어 (RHCOS 설치 + ignition URL)
3. `sudo reboot`

노드별 명령어 예시 (Master 1 — 192.168.0.11):
```bash
# 1. 네트워크 설정 (게이트웨이/DNS = Bastion 내부 IP)
nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.0.11/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns 192.168.0.1 \
  ipv4.method manual
nmcli con up "Wired connection 1"

# 2. RHCOS 설치 (httpd URL = Bastion 내부 IP)
sudo coreos-installer install /dev/sda \
  --ignition-url http://192.168.0.1:8080/master.ign \
  --insecure-ignition \
  --copy-network

# 3. 재부팅
sudo reboot
```

## 설치 완료 검증

```bash
./openshift-install --dir=./install-dir wait-for bootstrap-complete
./openshift-install --dir=./install-dir wait-for install-complete
oc get nodes
oc get clusteroperators
```

> 상세 트러블슈팅: `references/install-troubleshooting.md`
