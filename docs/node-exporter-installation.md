# Node Exporter installation

[Node Exporter](https://github.com/prometheus/node_exporter) is a simple prometheus exporter for host metrics deployed using the [node-exporter](../playbooks/node-exporter.yml) ansible playbook.

It's installation is reserved to 2 groups:

- **iaas**: Incus nodes
- **hpc**: High performance computing nodes

**frontplane** nodes don't require this playbook to be applied as [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md) already provides **node-exporter** pods.

1. Install the requirements:

```bash
ansible-galaxy install -r requirements.yml
```

2. Install node-exporter on the nodes that requires it:

```bash
ansible-playbook --limit hpc,iaas -i inventories/production/hosts playbooks/node-exporter.yml
```