---
title: Deploy and destroy
layout: default
nav_order: 5
---

# Deploy and destroy
{: .no_toc }

Manual AWS CLI on your laptop, then commands on the instance over an SSM session. No installer scripts.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

{: .cost }
> This creates a `t3.xlarge`, 150 GB gp3, and optionally an Elastic IP. [Destroy](#destroy) or stop the instance when finished.

Context: [Architecture]({{ site.baseurl }}/architecture/), [SeisComP]({{ site.baseurl }}/seiscomp/), [Desktop]({{ site.baseurl }}/desktop/).

## Prerequisites

| Requirement            | Check                                                        |
| ---------------------- | ------------------------------------------------------------ |
| AWS CLI v2             | `aws --version`                                              |
| Profile `sandbox`      | `aws sts get-caller-identity --profile sandbox`              |
| Session Manager plugin | `session-manager-plugin --version`                           |
| Region                 | always `ap-southeast-2` (this profile has no default region) |

```bash
export AWS_PROFILE=sandbox AWS_DEFAULT_REGION=ap-southeast-2 AWS_PAGER=""
aws sts get-caller-identity --query Account --output text
```

{: .tip }
> Prefer an interactive SSM session (`aws ssm start-session`) and paste the on-box blocks. `AWS-RunShellScript` uses `dash`; see [Troubleshooting]({{ site.baseurl }}/troubleshooting/).

## 1. VPC

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.81.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=seiscomp-lab},{Key=Project,Value=seiscomp-lab}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames

PUB_SUBNET=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.81.1.0/24 \
  --availability-zone ap-southeast-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=seiscomp-lab-public-2a},{Key=Project,Value=seiscomp-lab}]' \
  --query Subnet.SubnetId --output text)
aws ec2 modify-subnet-attribute --subnet-id "$PUB_SUBNET" --map-public-ip-on-launch

PRIV_SUBNET=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.81.11.0/24 \
  --availability-zone ap-southeast-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=seiscomp-lab-private-2a},{Key=Project,Value=seiscomp-lab}]' \
  --query Subnet.SubnetId --output text)

IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=seiscomp-lab-igw},{Key=Project,Value=seiscomp-lab}]' \
  --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

PUB_RT=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=seiscomp-lab-public},{Key=Project,Value=seiscomp-lab}]' \
  --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id "$PUB_RT" --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
aws ec2 associate-route-table --route-table-id "$PUB_RT" --subnet-id "$PUB_SUBNET"

PRIV_RT=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=seiscomp-lab-private},{Key=Project,Value=seiscomp-lab}]' \
  --query RouteTable.RouteTableId --output text)
aws ec2 associate-route-table --route-table-id "$PRIV_RT" --subnet-id "$PRIV_SUBNET"

SG_ID=$(aws ec2 create-security-group --vpc-id "$VPC_ID" \
  --group-name seiscomp-lab-aio \
  --description "SeisComP lab: outbound; RDP from operator IP" \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=seiscomp-lab-aio},{Key=Project,Value=seiscomp-lab}]' \
  --query GroupId --output text)

MYIP=$(curl -sS https://checkip.amazonaws.com | tr -d '[:space:]')
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 3389 --cidr "${MYIP}/32"
```

IAM instance profile (wait a few seconds after attaching the role before `run-instances`):

```bash
aws iam create-role --role-name seiscomp-lab-ssm \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name seiscomp-lab-ssm \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam create-instance-profile --instance-profile-name seiscomp-lab-ssm
aws iam add-role-to-instance-profile --instance-profile-name seiscomp-lab-ssm --role-name seiscomp-lab-ssm
sleep 10
```

## 2. Launch Ubuntu 24.04

```bash
AMI=$(aws ssm get-parameters \
  --names /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query 'Parameters[0].Value' --output text)

IID=$(aws ec2 run-instances \
  --image-id "$AMI" \
  --instance-type t3.xlarge \
  --subnet-id "$PUB_SUBNET" \
  --security-group-ids "$SG_ID" \
  --iam-instance-profile Name=seiscomp-lab-ssm \
  --associate-public-ip-address \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":150,"VolumeType":"gp3","Encrypted":true,"DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=seiscomp-lab-aio},{Key=Project,Value=seiscomp-lab}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids "$IID"
```

Wait until SSM is online (`PingStatus=Online`):

```bash
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=$IID"
```

Elastic IP (optional but convenient for Windows RDP):

```bash
ALLOC=$(aws ec2 allocate-address --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=seiscomp-lab},{Key=Project,Value=seiscomp-lab}]' \
  --query AllocationId --output text)
aws ec2 associate-address --instance-id "$IID" --allocation-id "$ALLOC"
aws ec2 describe-addresses --allocation-ids "$ALLOC" --query 'Addresses[0].PublicIp' --output text
```

{: .warning }
> `--associate-public-ip-address` is required. This subnet has no NAT. Without a public IP there is no outbound path (apt, gsm, GEOFON).

Open a shell:

```bash
aws ssm start-session --target "$IID"
```

The rest of this page is **on the instance**. Become root with `sudo -i` unless a block says `sysop`.

## 3. OS bootstrap

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y python3 python3-venv gnupg wget curl ca-certificates unzip chrony
id sysop || useradd -m -s /bin/bash -G sudo sysop
echo 'sysop ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/sysop
chmod 440 /etc/sudoers.d/sysop
timedatectl set-timezone UTC
systemctl enable --now chrony
sudo -u sysop bash -lc 'mkdir -p ~/install && cd ~/install && wget -q https://data.gempa.de/packages/Public/gsm/gempa-gsm.tar.gz && tar xzf gempa-gsm.tar.gz'
```

## 4. gsm (public repo)

```bash
sudo -u sysop bash -lc '
cd /home/sysop/install/gsm
./gsm setup -y -r 7 --os ubuntu --osversion 24.04 --arch x86_64 \
  --installpath /home/sysop/seiscomp --datadir /home/sysop/data
./gsm update
./gsm install -y seiscomp world-minimal
'
```

{: .warning }
> `gsm install` must have `-y` or it prompts and hangs.

## 5. Distro deps and MariaDB

`seiscomp install-deps` calls `apt install` without `-y`. Source the dep scripts and force `apt-get install -y`:

```bash
export DEBIAN_FRONTEND=noninteractive
apt() {
  if [ "${1:-}" = "install" ]; then
    shift
    /usr/bin/apt-get install -y "$@"
  else
    /usr/bin/apt-get "$@"
  fi
}
. /home/sysop/seiscomp/share/deps/ubuntu/24.04/install-base.sh
. /home/sysop/seiscomp/share/deps/ubuntu/24.04/install-gui.sh
. /home/sysop/seiscomp/share/deps/ubuntu/24.04/install-mariadb-server.sh
. /home/sysop/seiscomp/share/deps/ubuntu/24.04/install-fdsnws.sh

systemctl enable --now mariadb
cat >/etc/mysql/mariadb.conf.d/99-seiscomp.cnf <<'EOF'
[mysqld]
innodb_buffer_pool_size = 512M
innodb_flush_log_at_trx_commit = 2
character-set-server = utf8mb4
collation-server = utf8mb4_bin
bind-address = 127.0.0.1
EOF
systemctl restart mariadb

mysql <<'SQL'
CREATE DATABASE IF NOT EXISTS seiscomp CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER IF NOT EXISTS 'sysop'@'localhost' IDENTIFIED BY 'sysop';
GRANT ALL PRIVILEGES ON seiscomp.* TO 'sysop'@'localhost';
FLUSH PRIVILEGES;
SQL
mysql seiscomp < /home/sysop/seiscomp/share/db/mysql.sql
```

## 6. Config, inventory, bindings

```bash
install -d -o sysop -g sysop /home/sysop/seiscomp/etc /home/sysop/.seiscomp /home/sysop/data

cat >/home/sysop/seiscomp/etc/global.cfg <<'EOF'
agencyID = LEARN
datacenterID = LEARN
organization = "SeisComP learning lab"
recordstream = slink://localhost:18000
connection.server = localhost/production
EOF

cat >/home/sysop/seiscomp/etc/scmaster.cfg <<'EOF'
interface.bind = 127.0.0.1:18180
EOF

cat >/home/sysop/seiscomp/etc/slarchive.cfg <<'EOF'
keep = 7
EOF

chown sysop:sysop /home/sysop/seiscomp/etc/global.cfg \
  /home/sysop/seiscomp/etc/scmaster.cfg \
  /home/sysop/seiscomp/etc/slarchive.cfg

if ! grep -q SEISCOMP_ROOT /home/sysop/.bashrc; then
  sudo -u sysop /home/sysop/seiscomp/bin/seiscomp print env >> /home/sysop/.bashrc
fi

wget -q "http://geofon.gfz.de/fdsnws/station/1/query?net=GE&sta=WLF,STU,MORC,RGN&cha=BH%3F&level=response" \
  -O /tmp/ge-lab.xml
sudo -u sysop /home/sysop/seiscomp/bin/seiscomp exec import_inv fdsnxml /tmp/ge-lab.xml
```

Bindings:

```bash
install -d -o sysop -g sysop \
  /home/sysop/seiscomp/etc/key \
  /home/sysop/seiscomp/etc/key/global \
  /home/sysop/seiscomp/etc/key/seedlink \
  /home/sysop/seiscomp/etc/key/scautopick \
  /home/sysop/seiscomp/etc/key/slarchive

cat >/home/sysop/seiscomp/etc/key/global/profile_BH <<'EOF'
detecStream = BH
detecLocid = ""
EOF

cat >/home/sysop/seiscomp/etc/key/scautopick/profile_default <<'EOF'
EOF

cat >/home/sysop/seiscomp/etc/key/seedlink/profile_geofon <<'EOF'
sources = chain
sources.chain.address = geofon.gfz.de
sources.chain.port = 18000
sources.chain.selectors = BH?.D
EOF

cat >/home/sysop/seiscomp/etc/key/slarchive/profile_week <<'EOF'
keep = 7
selectors = BH?.D
EOF

for sta in WLF STU MORC RGN; do
  cat >/home/sysop/seiscomp/etc/key/station_GE_${sta} <<'EOF'
global:BH
scautopick:default
seedlink:geofon
slarchive:week
EOF
done
chown -R sysop:sysop /home/sysop/seiscomp/etc/key
```

Enable, start, systemd:

```bash
sudo -u sysop bash -lc '
export PATH=/home/sysop/seiscomp/bin:$PATH
export SEISCOMP_ROOT=/home/sysop/seiscomp
seiscomp enable scmaster seedlink slarchive scautopick scautoloc scamp scmag scevent
seiscomp update-config
seiscomp start
'

cat >/etc/systemd/system/seiscomp.service <<'EOF'
[Unit]
Description=SeisComP learning lab
After=network-online.target mariadb.service
Wants=network-online.target
Requires=mariadb.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=sysop
Group=sysop
Environment=HOME=/home/sysop
ExecStart=/home/sysop/seiscomp/bin/seiscomp start
ExecStop=/home/sysop/seiscomp/bin/seiscomp stop
TimeoutStartSec=120

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable seiscomp.service
```

FDSNWS, QC, event log:

```bash
cat >/home/sysop/seiscomp/etc/fdsnws.cfg <<'EOF'
listenAddress = 127.0.0.1
port = 8080
recordstream = sdsarchive://@ROOTDIR@/var/lib/archive
EOF
chown sysop:sysop /home/sysop/seiscomp/etc/fdsnws.cfg

sudo -u sysop bash -lc '
export SEISCOMP_ROOT=/home/sysop/seiscomp
seiscomp enable fdsnws scqc scevtlog
seiscomp update-config fdsnws scqc scevtlog
seiscomp start fdsnws scqc scevtlog
'
```

Helicorder streams:

```bash
cat >/home/sysop/seiscomp/etc/scheli.cfg <<'EOF'
heli.streams = GE.WLF..BHZ,GE.STU..BHZ,GE.MORC..BHZ,GE.RGN..BHZ
EOF
chown sysop:sysop /home/sysop/seiscomp/etc/scheli.cfg
```

## 7. Desktop (XFCE + xrdp)

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y \
  xfce4 xfce4-terminal xfce4-notifyd xfce4-panel xfce4-session \
  xfce4-settings xfce4-appfinder desktop-base dbus-x11 \
  xrdp xorgxrdp \
  fonts-dejavu-core libnotify-bin \
  mesa-utils libgl1-mesa-dri \
  policykit-1

systemctl disable --now lightdm 2>/dev/null || true

# Globals listen on 3389. Do not change [Xorg] port= (must stay -1).
python3 - <<'PY'
from pathlib import Path
p = Path("/etc/xrdp/xrdp.ini")
text = p.read_text()
out = []
section = ""
for line in text.splitlines(True):
    if line.startswith("["):
        section = line.strip()
    if line.startswith("port=") and section == "[Globals]":
        line = "port=3389\n"
    out.append(line)
p.write_text("".join(out))
PY

sed -i 's/^ListenAddress=.*/ListenAddress=0.0.0.0/' /etc/xrdp/sesman.ini
sed -i 's/^allowed_users=.*/allowed_users=anybody/' /etc/X11/Xwrapper.config

cat >/etc/xrdp/startwm.sh <<'EOF'
#!/bin/sh
if test -r /etc/profile; then . /etc/profile; fi
if test -r "$HOME/.profile"; then . "$HOME/.profile"; fi
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x /etc/xrdp/startwm.sh

adduser xrdp ssl-cert || true
systemctl enable --now xrdp
systemctl restart xrdp-sesman xrdp
```

Password into SSM (from your **laptop**, after generating on the box):

On the instance:

```bash
PASS=$(openssl rand -base64 18 | tr -d '/+=' | cut -c1-16)
echo "sysop:${PASS}" | chpasswd
echo "$PASS" > /root/seiscomp-rdp-password
chmod 600 /root/seiscomp-rdp-password
echo "$PASS"
```

On the laptop:

```bash
aws ssm put-parameter --name /seiscomp-lab/sysop-rdp-password \
  --type SecureString --value 'PASTE_THE_PASSWORD' --overwrite
```

Launchers and toasts:

```bash
install -d -o sysop -g sysop \
  /home/sysop/Desktop \
  /home/sysop/.local/share/applications \
  /home/sysop/.config/autostart \
  /home/sysop/bin

cat >/etc/profile.d/seiscomp.sh <<'EOF'
export SEISCOMP_ROOT=/home/sysop/seiscomp
export PATH="$SEISCOMP_ROOT/bin:$PATH"
export LD_LIBRARY_PATH="$SEISCOMP_ROOT/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export PYTHONPATH="$SEISCOMP_ROOT/lib/python${PYTHONPATH:+:$PYTHONPATH}"
export QT_QPA_PLATFORM=xcb
EOF

cat >/home/sysop/bin/sc-launch <<'EOF'
#!/bin/bash
set -e
export SEISCOMP_ROOT=/home/sysop/seiscomp
export PATH="$SEISCOMP_ROOT/bin:$PATH"
export LD_LIBRARY_PATH="$SEISCOMP_ROOT/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export PYTHONPATH="$SEISCOMP_ROOT/lib/python${PYTHONPATH:+:$PYTHONPATH}"
export QT_QPA_PLATFORM=xcb
cd "$HOME"
exec "$SEISCOMP_ROOT/bin/$1" "${@:2}"
EOF
chmod 755 /home/sysop/bin/sc-launch

cat >/home/sysop/bin/sc-toast-event <<'EOF'
#!/bin/bash
set +e
uid=$(id -u)
export HOME="${HOME:-/home/sysop}"
npid=$(pgrep -u "$uid" -f 'xfce4/notifyd/xfce4-notifyd' | head -1)
[ -z "$npid" ] && exit 0
export DBUS_SESSION_BUS_ADDRESS=$(tr '\0' '\n' < "/proc/$npid/environ" | sed -n 's/^DBUS_SESSION_BUS_ADDRESS=//p' | head -1)
export DISPLAY=$(tr '\0' '\n' < "/proc/$npid/environ" | sed -n 's/^DISPLAY=//p' | head -1)
body="$*"
[ -n "$body" ] || body="New SeisComP event"
gdbus call --session --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications \
  --method org.freedesktop.Notifications.Notify \
  SeisComP 0 dialog-information "SeisComP" "$body" '[]' '{}' 8000 >/dev/null
logger -t scalert "toast: $body"
EOF
chmod 755 /home/sysop/bin/sc-toast-event

cat >/home/sysop/seiscomp/etc/scalert.cfg <<'EOF'
scripts.event = /home/sysop/bin/sc-toast-event
EOF
chown sysop:sysop /home/sysop/seiscomp/etc/scalert.cfg

write_desktop() {
  local id="$1" name="$2" comment="$3" bin="$4"
  local file="/home/sysop/.local/share/applications/${id}.desktop"
  cat >"$file" <<EOF
[Desktop Entry]
Type=Application
Version=1.0
Name=${name}
Comment=${comment}
Exec=/home/sysop/bin/sc-launch ${bin}
Icon=applications-science
Terminal=false
Categories=Science;Geoscience;SeisComP;
StartupNotify=true
EOF
  chmod 755 "$file"
  cp "$file" "/home/sysop/Desktop/${id}.desktop"
}

write_desktop scconfig "scconfig" "Configure modules, stations and bindings" scconfig
write_desktop scmv "scmv" "Map view" scmv
write_desktop scmvx "scmvx" "Map view (scmvx)" scmvx
write_desktop scrttv "scrttv" "Real-time waveform viewer" scrttv
write_desktop scolv "scolv" "Event analysis" scolv
write_desktop scesv "scesv" "Event summary" scesv
write_desktop scheli "scheli" "Helicorder" scheli
write_desktop scqcv "scqcv" "Quality control" scqcv
write_desktop scmm "scmm" "Messaging-system monitor" scmm

cat >/home/sysop/.xsession <<'EOF'
#!/bin/bash
. /etc/profile
. "$HOME/.profile"
exec startxfce4
EOF
chmod 755 /home/sysop/.xsession
chown -R sysop:sysop /home/sysop/Desktop /home/sysop/.local /home/sysop/.config /home/sysop/bin /home/sysop/.xsession

sudo -u sysop /home/sysop/seiscomp/bin/seiscomp enable scalert
sudo -u sysop /home/sysop/seiscomp/bin/seiscomp restart scalert
```

Confirm `[Xorg]` still has `port=-1`:

```bash
awk '/^\[Xorg\]/,/^\[/{print}' /etc/xrdp/xrdp.ini | head
ss -lntp | grep 3389
```

## 8. Prove

On the instance as `sysop`:

```bash
sudo -iu sysop
seiscomp check
slinktool -Q localhost
curl -sS 'http://127.0.0.1:8080/fdsnws/station/1/query?net=GE&level=station&format=text'
```

Expect live BH for GE.MORC, GE.RGN, GE.STU, GE.WLF. More checks: [Prove]({{ site.baseurl }}/prove/). RDP: [Desktop]({{ site.baseurl }}/desktop/).

## Destroy

From the laptop. Stop first if you might come back; terminate and delete when done.

```bash
export AWS_PROFILE=sandbox AWS_DEFAULT_REGION=ap-southeast-2 AWS_PAGER=""

# Stop (keeps disk; EIP bills if left associated)
aws ec2 stop-instances --instance-ids "$IID"

# Or terminate
aws ec2 terminate-instances --instance-ids "$IID"
aws ec2 wait instance-terminated --instance-ids "$IID"

aws ec2 disassociate-address --association-id "$ASSOC"   # if you noted it
aws ec2 release-address --allocation-id "$ALLOC"

aws ec2 delete-security-group --group-id "$SG_ID"
aws ec2 detach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID"
# delete subnets, route tables, then:
aws ec2 delete-vpc --vpc-id "$VPC_ID"

aws iam remove-role-from-instance-profile --instance-profile-name seiscomp-lab-ssm --role-name seiscomp-lab-ssm
aws iam delete-instance-profile --instance-profile-name seiscomp-lab-ssm
aws iam detach-role-policy --role-name seiscomp-lab-ssm \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name seiscomp-lab-ssm

aws ssm delete-parameter --name /seiscomp-lab/sysop-rdp-password
```

Delete unused volumes and network interfaces if terminate left anything behind.
