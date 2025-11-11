# Incus cluster installation

Here will be all the steps needed to install the [Incus](https://linuxcontainers.org/incus/) cluster running at [φorge](https://phorge.fr).

This setup comes with many features such as:

- Software Defined Networks(SDN) with [OVN](https://www.ovn.org/en/)
- Redundant Storage with [MicroCeph](https://github.com/canonical/microceph)
- Monitoring & Logging with [Prometheus](https://prometheus.io/) & [Loki](https://grafana.com/docs/loki/latest/)
- Live Migration
- Instances High Availability
- Cluster Groups
- LoadBalancer & Forwards with **BGP**
- OIDC & Authorization with [Dex](https://dexidp.io/) & [OpenFGA](https://openfga.dev/)

> Every other configurations outside of **Incus**, **MicroCeph** and **OVN** can be found in other [φorge repositories](https://github.com/orgs/phorge-fr/repositories).

## Hardware setup

We have 2 group of 3 nodes:

- Anvil: 16Gb RAM, 4c/4t, 300GB HDD, 128GB NvME, 2x 1G NIC
- Hammer: 8Gb RAM, 4c/4t, 128GB NvME, 2x 1G NIC

> Might change in the future

## Informations about nodes

In this documentation you'll often see `[All nodes]` or `[Remaining nodes]` etc. This indicates you wher to run a command. Here is a quick list of the meanings:

- [All nodes] = all nodes
- [Bootstrap node] = a node in your setup(can be any one of them, I recommend the 1st machine you deployed)
- [Remaining nodes] = all nodes that are not the bootstrap node
- [Hammer/Anvil nodes] = all nodes in a group
- ["name" node] = specific node

### Install prerequisites

1. Update & install dependencies

[All nodes]
```bash
apt update
apt install -y curl wget nano snapd
```

2. Run hostname-randomizer.sh

[All nodes]
```bash
curl https://gist.github.com/l-nmch/14e883ddaad86fd7e7930dcbb2fcef5e | sh
```

P.S. hostname-randomizer.sh changes the machine hostname to a combination of <adjective>-<animal>. It makes machine unique and easy to locate.

### Setup Incus

1. Add **Incus** repository

[All nodes]
```bash
curl -fsSL https://pkgs.zabbly.com/key.asc -o /etc/apt/keyrings/zabbly.asc
sh -c 'cat <<EOF > /etc/apt/sources.list.d/zabbly-incus-stable.sources
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc

EOF'
apt update
```

P.S. Even though **Incus** is technically already available on **Debian 13** official repositories, the package `incus-ui-canonical` is missing and is required in our setup.

2. Install **Incus** packages


[All nodes]
```bash
apt install -y incus incus-base incus-ui-canonical
```

3. Initialize Incus

[Bootstrap node]
```bash
incus admin init
```

Will prompt:

```bash
Would you like to use clustering? (yes/no) [default=no]: yes
What IP address or DNS name should be used to reach this server? [default=10.1.0.1]: 
Are you joining an existing cluster? (yes/no) [default=no]: 
What member name should be used to identify this server in the cluster? [default=clever-lynx]: 
Do you want to configure a new local storage pool? (yes/no) [default=yes]: 
Where should this storage pool store its data? [default=/var/lib/incus/storage-pools/local]: 
Do you want to configure a new remote storage pool? (yes/no) [default=no]: 
Would you like to use an existing bridge or host interface? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: 
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]: 
```

4. Add other nodes to the cluster

[Bootstrap node]
```bash
incus cluster add <other node name> # Repeat for each nodes and save the tokens
```

[Remaining nodes]
```bash
incus admin init
```

Will prompt:

```bash
Would you like to use clustering? (yes/no) [default=no]: yes
What IP address or DNS name should be used to reach this server? [default=10.1.0.x]: 
Are you joining an existing cluster? (yes/no) [default=no]: yes
Please provide join token: <token> # then enter "yes" to each following steps
```

5. Configure **Incus**

[Bootstrap node]
```bash
incus config set acme.agree_tos="true"
incus config set acme.challenge="DNS-01"
incus config set acme.domain="iaas.phorge.fr"
incus config set acme.email="<email>"
incus config set acme.provider="cloudflare"
incus config set acme.provider.environment=|
  CLOUDFLARE_EMAIL=<email>
  CLOUDFLARE_API_KEY=<api-key>
incus config set cluster.healing_threshold="4"
incus config set cluster.offline_threshold="11"
incus config set core.bgp_asn="65535"
incus config set logging.loki-frontplane.lifecycle.types="instance"
incus config set logging.loki-frontplane.target.address="http://10.0.0.11:3100"
incus config set logging.loki-frontplane.target.type="loki"
incus config set logging.loki-frontplane.types="logging,lifecycle"
incus config set oidc.claim="email"
incus config set oidc.client.id="Incus"
incus config set oidc.issuer="https=//auth.phorge.fr"
incus config set oidc.scopes="openid, offline_access, email"
incus config set openfga.api.token="<openfga-preshared-key>"
incus config set openfga.api.url="http=//10.0.0.12=8080"
incus config set openfga.store.id="<openfga-store-id>"
incus config set user.grafana_base_url="https=//monitoring.phorge.fr"
incus config set user.ui.title="Phorge Cloud IaaS"
```

P.S. Unlike other configuration options, these are not node specific and can be run on any nodes

6. Node specific configuration

[Each nodes]
```bash
incus config set core.metrics_address="10.1.0.x:8444"
incus config set core.bgp_address="10.1.0.x"
incus config set core.bgp_asn="65535"
incus config set core.bgp_routerid="10.1.0.x"
# Replace x by the last IP bit for each machine
```

### Setup microceph

:warning: Each step listed here are limited to nodes with more than 1 disk (here only *Anvil* group nodes) :warning:

1. Install **microceph**

[Each Anvil nodes]
```bash
snap install microceph
snap refresh --hold microceph # Disables auto-updates
```

2. Avoid apparmor issues

[Each Anvil nodes]
```bash
cp /var/lib/snapd/apparmor/profiles/snap.microceph.daemon /etc/apparmor.d/
cp /var/lib/snapd/apparmor/profiles/snap.microceph.osd /etc/apparmor.d/
service apparmor restart
service snapd.apparmor restart
```

3. Boostrap cluster

[Anvil Bootstrap node]
```bash
microceph cluster bootstrap
microceph.ceph config set mon mon_allow_pool_delete true
```

4. Add other nodes to the cluster

[Anvil Boostrap node]
```bash
microceph cluster add <other node name> # Repeat for each other nodes with more than 1 disk and copy the tokens
```

5. Join the cluster

[Anvil Remaining nodes]
```bash
microceph cluster join <token> # Repeate on each nodes with more than 1 disk
```

6. Add disk to the cluster

[Each Anvil nodes]
```bash
microceph disk add /dev/nvme0n1 --wipe
```

7. Verify disks

[Bootstrap node]
```bash
microceph disk list
```

Will prompt:

```
Disks configured in MicroCeph:
+-----+-------------+-----------------------------------------------------------+
| OSD |  LOCATION   |                           PATH                            |
+-----+-------------+-----------------------------------------------------------+
| 1   | clever-lynx | /dev/disk/by-id/nvme-eui.343433304b9913800025384100000001 |
+-----+-------------+-----------------------------------------------------------+
| 2   | gentle-fox  | /dev/disk/by-id/nvme-eui.343433304b9913720025384100000001 |
+-----+-------------+-----------------------------------------------------------+
| 3   | mighty-deer | /dev/disk/by-id/nvme-eui.000000000000001000080d050015d982 |
+-----+-------------+-----------------------------------------------------------+
```

8. Prepare configuration for **Incus**

[Anvil bootstrap node]
```bash
cp /var/snap/microceph/current/conf/ceph.conf /etc/ceph/
cp /var/snap/microceph/current/conf/ceph.client.admin.keyring /etc/ceph
```

Now transfer these files from the *Anvil bootstrap node* to *Remaining Anvil nodes* at the same destination path

9. Create **Incus** storage pool

[Anvil bootstrap node]
```bash
incus storage create test ceph --target <node> # Repeat for each node in the cluster
incus storage create test ceph
```


### Setup OVN

1. Install **OVN**

[Each nodes]
```bash
apt update
apt install -y ovn-central ovn-host
```

2. Setup defaults

[Each nodes]
```bash
nano /etc/default/ovn-central
```

Paste:

```bash
OVN_CTL_OPTS=" \
     --db-nb-addr=10.1.0.1 \
     --db-nb-create-insecure-remote=yes \
     --db-sb-addr=10.1.0.1 \
     --db-sb-create-insecure-remote=yes \
     --db-nb-cluster-local-addr=10.1.0.1 \
     --db-sb-cluster-local-addr=10.1.0.1 \
     --ovn-northd-nb-db=tcp:10.1.0.1:6641,tcp:10.1.0.2:6641,tcp:10.1.0.3:6641,tcp:10.1.0.4:6641,tcp:10.1.0.5:6641,tcp:10.1.0.6:6641 \
     --ovn-northd-sb-db=tcp:10.1.0.1:6642,tcp:10.1.0.2:6642,tcp:10.1.0.3:6642,tcp:10.1.0.4:6642,tcp:10.1.0.5:6642,tcp:10.1.0.6:6642"
```

Now do the same for each node but modify: `--db-nb-cluster-local-addr` & `--db-sb-cluster-local-addr` with the IP of the current node.

3. Setup `ovs-vsctl`

[Each node]
```bash
ovs-vsctl set open_vswitch . \
   external_ids:ovn-remote=tcp:10.1.0.1:6642,tcp:10.1.0.2:6642,tcp:10.1.0.3:6642,tcp:10.1.0.4:6642,tcp:10.1.0.5:6642,tcp:10.1.0.6:6642 \
   external_ids:ovn-encap-type=geneve \
   external_ids:ovn-encap-ip=10.1.0.1
```

Don't forget to modify `external_ids:ovn-encap-ip=` with the IP of the current node.

4. Create **Incus** UPLINK network

[Bootstrap node]
```bash
incus network create UPLINK --type=physical parent=<parent interface> --target=<node name> # Repeat for each node in the clyster
incus network create UPLINK --type=physical \
   bgp.peers.core-gw.address=10.1.0.254 \
   bgp.peers.core-gw.asn="65535" \
   ipv4.routes: 10.3.0.0/24 \ # Used for LoadBalancers & Forwards
   ipv4.ovn.ranges=10.2.0.10-10.2.0.20 \
   ipv4.gateway=10.2.0.254/24 \
   dns.nameservers=10.2.0.254
```

## Notes

You should now have a fully featured **Incus** cluster. This is a complicated and detailed configuration and it may vary depending on your infrastructure.