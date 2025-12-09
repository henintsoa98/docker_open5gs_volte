# DOCKER INSTALLATION
:q
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $(whoami)
echo "Do relogin session"
```
# ADJUSTEMENT
(Do apt search linux-image-generic to find latest version that replace 6.8, step to activate mode performance)
```
docker pull ubuntu:focal
sudo apt install linux-image-generic-6.8 linux-tools-common linux-tools-generic-6.8
echo "Need to reboot into 6.8 kernel"
```

# BUILDING IMAGES DOCKER
```bash
cd
# Build docker images for open5gs EPC/5GC components
git clone https://github.com/lfarizav/docker_open5gs
tar cJf docker_open5gs.tar.xz docker_open5gs
cd docker_open5gs/base
## as =>AS
docker build --no-cache --force-rm -t docker_open5gs .
# Build docker images for kamailio IMS components
cd ../ims_base
docker build --no-cache --force-rm -t docker_kamailio .
# Build docker images for srsRAN_4G eNB + srsUE (4G+5G)
cd ../srslte
docker build --no-cache --force-rm -t docker_srslte .

cd ..
set -a
source .env
sudo ufw disable
sudo sysctl -w net.ipv4.ip_forward=1
sudo cpupower frequency-set -g performance

# For 4G deployment only
docker compose -f docker-compose.yaml build
```

# PARAMETER OF MCC/MNC AND IP
```bash
DOCKER_HOST_IP=`ip addr show docker0 | grep "inet " | awk '{print $2}' | sed "s#/[0-9]*##"`
MCC=999
MNC=70
PRB=15
sed -i "s/DOCKER_HOST_IP=[.0-9]*/DOCKER_HOST_IP=$DOCKER_HOST_IP/" .env
sed -i "s/SGWU_ADVERTISE_IP=[.0-9]*/SGWU_ADVERTISE_IP=$DOCKER_HOST_IP/" .env
sed -i "s/MCC=[0-9]*/MCC=$MCC/" .env
sed -i "s/MNC=[0-9]*/MNC=$MNC/" .env
sed -i "s/n_prb = [0-9]*/n_prb = $PRB/" srslte/enb.conf
```

# NETWORK DEPLOYMENT
**OPEN TERMINAL 1 AND 2, RUN COMMAND AT SAME TIME, diff time < 5s, FIRST START WILL FAIL SOMETIMES BECAUSE OF INITIALIZATION, SO Ctrl-C, AND RUN AGAIN**
## terminal 1 : DEPLOY
```bash
set -a
sudo ufw disable
sudo sysctl -w net.ipv4.ip_forward=1
sudo cpupower frequency-set -g performance
source .env

docker compose -f docker-compose.yaml up
```
## terminal 2 : SRSENB
```bash
docker attach srsenb
```

# PROVISIONNING SIM IN HSS USING SCRIPT
**NB : each sim has it own key, so adjust to your value or search for programming your sim** \
open5gs-dbctl add {imsi key opc}: adds a user to the database with default values
## terminal 3 : UE SCRIPT
```bash
cd
git clone --depth 1 https://github.com/henintsoa98/keyosmocom
git checkout bdfcea03e712c600bd88192c9f7a8ab4c96f10ff
cd keyosmocom/docker_open5gs_build
bash setup.bash
#in wayland : (in X11 session, just copy the output of : cat OSMOHLR)
cat OSMOHLR | wl-copy
```

## terminal 4 : OSMOHLR SCRIPT
```bash
docker exec -ti osmohlr bash
telnet localhost 4258
# now in telnet session
enable
## PASTE 
```

# PROVISIONNING SIM IN HSS MANUALLY
## terminal 3 : UE MANUALLY
```bash
cd
cd docker_open5gs
source .env
xdg-open http://$DOCKER_HOST_IP:8080/docs/
```
## APN > PUT > TRY IT OUT
Paste and EXECUTE for this two parameter separately
```json
{
  "apn": "internet",
  "arp_priority": 0,
  "nidd_mechanism": 0,
  "ip_version": 0,
  "arp_preemption_capability": true,
  "nidd_rds": 0,
  "pgw_address": "string",
  "arp_preemption_vulnerability": true,
  "nidd_preferred_data_mode": 0,
  "sgw_address": "string",
  "charging_rule_list": "string",
  "charging_characteristics": "stri",
  "nbiot": true,
  "apn_ambr_dl": 0,
  "nidd_scef_id": "string",
  "apn_id": 1,
  "apn_ambr_ul": 0,
  "nidd_scef_realm": "string",
  "qci": 0
}
```
and
```json
{
  "apn": "ims",
  "arp_priority": 0,
  "nidd_mechanism": 0,
  "ip_version": 0,
  "arp_preemption_capability": true,
  "nidd_rds": 0,
  "pgw_address": "string",
  "arp_preemption_vulnerability": true,
  "nidd_preferred_data_mode": 0,
  "sgw_address": "string",
  "charging_rule_list": "string",
  "charging_characteristics": "stri",
  "nbiot": true,
  "apn_ambr_dl": 0,
  "nidd_scef_id": "string",
  "apn_id": 2,
  "apn_ambr_ul": 0,
  "nidd_scef_realm": "string",
  "qci": 0
}
```

## AUC > PUT > TRY IT OUT
Paste, change value of $ICCID, $IMSI, $KI, $OPC, $ADM1, and EXECUTE for each user
```json
{
  "sqn": 0,
  "pin1": "string",
  "misc1": "string",
  "iccid": "$iccid",
  "pin2": "string",
  "misc2": "string",
  "imsi": "$imsi",
  "puk1": "string",
  "misc3": "string",
  "batch_name": "string",
  "puk2": "string",
  "misc4": "string",
  "sim_vendor": "string",
  "kid": "string",
  "ki": "$ki",
  "esim": true,
  "psk": "string",
  "opc": "$opc",
  "lpa": "string",
  "des": "string",
  "amf": "8000",
  "adm1": "$adm1"
}
```
Take note for value of **auc_id** in response body under server response, for next step

## SUBSCRIBER > PUT > TRY IT OUT
Paste, change the value of $AUC_ID, $MSISDN, $IMSI, and EXECUTE for each user
```json
{
  "enabled": true,
  "roaming_enabled": true,
  "auc_id": ${auc_id},
  "roaming_rule_list": null,
  "default_apn": 1,
  "subscribed_rau_tau_timer": 300,
  "apn_list": "1,2",
  "serving_mme": null,
  "msisdn": "$msisdn",
  "serving_mme_timestamp": null,
  "ue_ambr_dl": 0,
  "serving_mme_realm": null,
  "imsi": "$imsi",
  "ue_ambr_ul": 0,
  "serving_mme_peer": null,
  "nam": 0
}
```

## IMS_SUBSCRIBER > PUT > TRY IT OUT
Paste, change value of $MNC, $MCC, $MSISDN, $IMSI, and EXECUTE for each user \
NB: here, if mcc/mnc = 999/70 => MNC = 070 , MCC = 999
```json
{
  "pcscf": null,
  "scscf": "sip:scscf.ims.mnc${MNC}.mcc${MCC}.3gppnetwork.org:6060",
  "pcscf_realm": null,
  "pcscf_active_session": null,
  "scscf_realm": "ims.mnc${MNC}.mcc${MCC}.3gppnetwork.org",
  "pcscf_timestamp": null,
  "scscf_peer": "scscf.ims.mnc${MNC}.mcc${MCC}.3gppnetwork.org;hss.ims.mnc${MNC}.mcc${MCC}.3gppnetwork.org",
  "msisdn": "$msisdn",
  "pcscf_peer": null,
  "sh_template_path": null,
  "msisdn_list": "[$msisdn]",
  "xcap_profile": null,
  "imsi": "$imsi",
  "sh_profile": "string",
  "ifc_path": "default_ifc.xml"
}
```

## terminal 4 : OSMOHLR MANUALLY
```bash
docker exec -ti osmohlr bash
telnet localhost 4258
# now in telnet session
enable
## set each user as
subscriber imsi 999700000087870 create
subscriber imsi 999700000087870 update msisdn 00000001
```

# PROVISIONNING SIM IN OPEN5GS
If from script, just edit to match to this config bellow, 
```bash
cd
cd docker_open5gs
source .env
xdg-open http://$DOCKER_HOST_IP:9999
```
LOGIN
|username|admin|
| ---    | --- |
|password|1423 |


Please also register MSISDN. At that time, set the APN setting information as follows. in Kbps
| APN      | Type | QCI | ARP | Capability | Vulnerablility | MBR DL/UL   | GBR DL/UL |
| ---      | ---  | --- | --- | ---        | ---            | ---         | ---       |
| internet | IPv4 | 9   | 8   | Disabled   | Disabled       | 1Gbps/1Gbps |           |
| ims      | IPv4 | 5   | 1   | Disabled   | Disabled       | 3850/1530   |           |
|          |      | 1   | 2   | Enabled    | Enabled        | 128/128     | 128/128   |
|          |      | 2   | 4   | Enabled    | Enabled        | 812/812     | 812/812   |

- Tap + button
- add IMSI
- add Subscriber Key K
- add Operator Key OPC
- .
- Go to Slice configuration, Session configuration
- APN : internet
- type : IPV4
- QCI : 9
- ARP : 8
- capability : disabled
- vulnerability : disabled
- AMBR DL : 1Gbps
- AMBR UL : 1Gbps
.
- Go to PCC Rules
- Tap + button on center
- APN : ims
- type : IPV4
- QCI : 5
- ARP : 1
- capability : disabled
- vulnerability : disabled
- AMBR DL : 3850Kbps
- AMBR UL : 1530Kbps
- .
- Go to PCC Rules bellow
- Tap + button on left
- QCI : 1
- ARP : 2
- capability : enabled
- vulnerability : enabled
- MBR DL : 128Kbps
- MBR UL : 128Kbps
- GBR DL : 128Kbps
- GBR UL : 128Kbps
- .
- Go to PCC Rules bellow
- Tap + button on left
- QCI : 2
- ARP : 4
- capability : enabled
- vulnerability : enabled
- MBR DL : 812Kbps
- MBR UL : 812Kbps
- GBR DL : 812Kbps
- GBR UL : 812Kbps

# NOW CONNECT THE USER
