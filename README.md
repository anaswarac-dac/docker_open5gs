# docker_open5gs
Quite contrary to the name of the repository, this repository contains docker files to deploy an Over-The-Air (OTA) or RF simulated 4G/5G network using following projects:
- Core Network (4G/5G) - open5gs - https://github.com/open5gs/open5gs
- IMS (Only 4G supported i.e. VoLTE) - kamailio
- IMS HSS - https://github.com/nickvsnetworking/pyhss
- Osmocom HLR - https://github.com/osmocom/osmo-hlr
- Osmocom MSC - https://github.com/osmocom/osmo-msc
- srsRAN (4G/5G) - https://github.com/srsran/srsRAN
- UERANSIM (5G) - https://github.com/aligungr/UERANSIM

## Tested Setup

Docker host machine

- Ubuntu 20.04 or 22.04

Over-The-Air setups: 

- srsRAN (eNB/gNB) using Ettus USRP B210
- srsRAN eNB using LimeSDR Mini v1.3
- srsRAN eNB using LimeSDR-USB

RF simulated setups:

 - srsRAN (gNB + UE) simulation over ZMQ
 - UERANSIM (gNB + UE) simulator

## Building docker images

* Mandatory requirements:
	* [docker-ce](https://docs.docker.com/install/linux/docker-ce/ubuntu) - Version 22.0.5 or above
	* [docker compose](https://docs.docker.com/compose) - Version 2.14 or above


#### Clone repository and build base docker image of open5gs, kamailio, srsRAN_4G, srsRAN_Project, ueransim

```
# Build docker images for open5gs EPC/5GC components
git clone https://github.com/anaswarac-dac/docker_open5gs.git
cd docker_open5gs
git checkout ims-sdcore
cd ims_base
sudo docker build --no-cache --force-rm -t docker_kamailio .
```
### Multihost setup configuration

#### Kamailio deployment

###### On the host running the 5GC

Edit only the following parameters in **.env** as per your setup
```
cd ..
nano .env
```

```
MCC
MNC
DOCKER_HOST_IP --> This is the IP address of the host running 5GC
UPF_ADVERTISE_IP --> Change this to value of DOCKER_HOST_IP
UE_IPV4_INTERNET --> Change this to your desired (Not conflicted) UE network ip range for internet APN
UE_IPV4_IMS --> Change this to your desired (Not conflicted) UE network ip range for ims APN
SDCORE_NRF_IP=10.42.132.162 --> SDCore NRF Service IP
SDCORE_NRF_PORT=29510
SDCORE_PCF_IP=10.42.193.238 --> SDCore PCF Service IP
SDCORE_PCF_PORT=29507
```
## Network Deployment

###### Kamailio deployment

```
set -a
source .env
set +a
sudo ufw disable
sudo sysctl -w net.ipv4.ip_forward=1

# For Kamailio deployment only
sudo docker compose -f vonr-deploy.yaml build

# Kamailio
sudo docker compose -f vonr-deploy.yaml up
```

## Provisioning of SIM information

### Provisioning of SIM information in pyHSS is as follows:

1. Goto http://<DOCKER_HOST_IP>:8080/docs/
2. Select **apn** -> **Create new APN** -> Press on **Try it out**. Then, in payload section use the below JSON and then press **Execute**

```
{
  "qci": 9,
  "arp_priority": 8,
  "apn": "internet",
  "apn_ambr_dl": 200,
  "apn_ambr_ul": 100,
  "arp_preemption_capability": true,
  "arp_preemption_vulnerability": true,
  "nbiot": false
}
```

Take note of **apn_id** specified in **Response body** under **Server response** for **internet** APN

Repeat creation step for following payload

```
{
  "apn_ambr_ul": 12,
  "qci": 5,
  "apn": "ims",
  "arp_priority": 1,
  "arp_preemption_capability": true,
  "arp_preemption_vulnerability": true,
  "nbiot": false,
  "apn_ambr_dl": 24
}
```

```
{
  "apn_ambr_ul": 2,
  "qci": 1,
  "apn": "ims",
  "arp_priority": 1,
  "arp_preemption_capability": true,
  "arp_preemption_vulnerability": true,
  "nbiot": false,
  "apn_ambr_dl": 4
}
```

Take note of **apn_id** specified in **Response body** under **Server response** for **ims** APN

**Execute this step of APN creation only once**

3. Next, select **auc** -> **Create new AUC** -> Press on **Try it out**. Then, in payload section use the below example JSON to fill in ki, opc and amf for your SIM and then press **Execute**

```
{
  "opc": "C42449363BBAD02B66D16BC975D77CC1",
  "amf": "8000",
  "imsi": "001010000000001",
  "ki": "fec86ba6eb707ed08905757b1bb44b8f",
  "sqn": 0
}
```

```
{
  "opc": "C42449363BBAD02B66D16BC975D77CC1",
  "amf": "8000",
  "imsi": "001010000000002",
  "ki": "fec86ba6eb707ed08905757b1bb44b8f",
  "sqn": 0
}
```

Take note of **auc_id** specified in **Response body** under **Server response**

**Replace imsi, ki, opc and amf as per your programmed SIM**

4. Next, select **subscriber** -> **Create new SUBSCRIBER** -> Press on **Try it out**. Then, in payload section use the below example JSON to fill in imsi, auc_id and apn_list for your SIM and then press **Execute**

```
{
  "imsi": "001010000000001",
  "enabled": true,
  "auc_id": 1,
  "default_apn": 1,
  "apn_list": "1,2",
  "msisdn": "9000000001",
  "ue_ambr_dl": 0,
  "ue_ambr_ul": 0
}
```

```
{
  "imsi": "001010000000002",
  "enabled": true,
  "auc_id": 2,
  "default_apn": 1,
  "apn_list": "1,2",
  "msisdn": "9000000002",
  "ue_ambr_dl": 0,
  "ue_ambr_ul": 0
}
```

- **auc_id** is the ID of the **AUC** created in the previous steps
- **default_apn** is the ID of the **internet** APN created in the previous steps
- **apn_list** is the comma separated list of APN IDs allowed for the UE i.e. APN ID for **internet** and **ims** APN created in the previous steps

**Replace imsi and msisdn as per your programmed SIM**

5. Finally, select **ims_subscriber** -> **Create new IMS SUBSCRIBER** -> Press on **Try it out**. Then, in payload section use the below example JSON to fill in imsi, msisdn, msisdn_list, scscf_peer, scscf_realm and scscf for your SIM/deployment and then press **Execute**

```
{
    "imsi": "001010000000001",
    "msisdn": "9000000001",
    "sh_profile": "string",
    "scscf_peer": "scscf.ims.mnc001.mcc001.3gppnetwork.org",
    "msisdn_list": "[9000000001]",
    "ifc_path": "default_ifc.xml",
    "scscf": "sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060",
    "scscf_realm": "ims.mnc001.mcc001.3gppnetwork.org"
}
```

```
{
    "imsi": "001010000000002",
    "msisdn": "9000000002",
    "sh_profile": "string",
    "scscf_peer": "scscf.ims.mnc001.mcc001.3gppnetwork.org",
    "msisdn_list": "[9000000002]",
    "ifc_path": "default_ifc.xml",
    "scscf": "sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060",
    "scscf_realm": "ims.mnc001.mcc001.3gppnetwork.org"
}
```

**Replace imsi, msisdn and msisdn_list as per your programmed SIM**

**Replace scscf_peer, scscf and scscf_realm as per your deployment**

## Not supported
- IPv6 usage in Docker
