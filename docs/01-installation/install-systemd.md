# Grafana systemd 설치

단일 VM이나 베어메탈에서 `grafana-server`를 systemd 서비스로 관리할 때 사용한다.

## 대상

- `grafana-server.service`

## 준비물

- Grafana 설치 패키지 또는 바이너리
- 설정 파일: `/etc/grafana/grafana.ini`
- provisioning 디렉터리: `/etc/grafana/provisioning/`
- 데이터 디렉터리: `/var/lib/grafana`

## RPM 설치

RHEL 계열에서는 Grafana RPM을 직접 설치하거나 Grafana repository를 등록해서 설치한다. 운영 환경에서는 버전을 고정한다.

### RPM 파일 직접 설치

```bash
GRAFANA_VERSION="<VERSION>"
curl -LO "https://dl.grafana.com/oss/release/grafana-${GRAFANA_VERSION}-1.x86_64.rpm"
sudo dnf install -y "./grafana-${GRAFANA_VERSION}-1.x86_64.rpm"
```

### repository 등록 후 설치

```bash
sudo tee /etc/yum.repos.d/grafana.repo >/dev/null <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

sudo dnf install -y grafana
rpm -qa | grep grafana
```

RPM 설치 후 기본 unit은 보통 `grafana-server.service`로 제공된다.

```bash
systemctl cat grafana-server
```

## tarball 설치

패키지 저장소를 쓰지 않는 환경에서는 tarball을 `/opt/grafana` 아래에 배치하고 직접 systemd unit을 만든다.

```bash
GRAFANA_VERSION="<VERSION>"
curl -LO "https://dl.grafana.com/oss/release/grafana-${GRAFANA_VERSION}.linux-amd64.tar.gz"
tar -xzf "grafana-${GRAFANA_VERSION}.linux-amd64.tar.gz"
sudo mkdir -p /opt/grafana /etc/grafana /var/lib/grafana
sudo cp -r "grafana-v${GRAFANA_VERSION}.linux-amd64"/* /opt/grafana/
sudo cp /opt/grafana/conf/defaults.ini /etc/grafana/grafana.ini
```

tarball 방식은 unit 파일과 사용자 생성을 직접 관리해야 한다.

## 절차

1. RPM 또는 tarball 방식 중 하나로 설치한다.
2. `grafana.ini`와 provisioning 파일을 배치한다.
3. `systemctl daemon-reload` 후 `enable --now grafana-server`를 실행한다.
4. `journalctl -u grafana-server -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status grafana-server
curl http://localhost:3000/api/health
```

## 운영 포인트

- 데이터소스와 대시보드는 provisioning으로 관리한다.
- 관리자 비밀번호는 환경 변수나 별도 secret 파일로 주입한다.
- 플러그인 설치가 필요하면 패키지 레벨 또는 시작 스크립트에서 반영한다.
