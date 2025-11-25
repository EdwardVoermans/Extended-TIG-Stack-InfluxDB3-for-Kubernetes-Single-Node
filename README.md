# Extended TIG Stack Deployment on single-node K3s

Deploy a complete Telegraf-InfluxDB-Grafana (TIG) stack on a single-node Kubernetes cluster using K3s. This guide walks you through setting up a production-ready monitoring stack with persistent storage and HTTPS access.

## What You'll Get

A fully functional monitoring stack running on K3s with:
- **Telegraf** for metrics collection
- **InfluxDB v3** for time-series data storage
- **Grafana** for visualization dashboards 
- **InfluxDB Explorer** for database management
- Persistent storage that survives pod restarts
- HTTPS access via Traefik ingress (with self-signed certificates)

## Alternative versions:
- Looking for the Docker version? Check: https://tinyurl.com/5n8ydsr4
- Looking for the multi-node Kubernetes version? Check: https://tinyurl.com/3889hkvr

## Prerequisites

Before you begin:
- A server or VM with K3s installed (default installation)
- Basic familiarity with Kubernetes concepts
- DNS entries or hosts file configuration (details below)

## Storage Architecture

This deployment creates three Persistent Volume Claims (PVCs) to preserve your data across application upgrades and restarts:
1. Grafana dashboards and settings
2. InfluxDB v3 database files
3. InfluxDB Explorer configuration

### Custom Storage Location

By default, K3s stores PVC data in cryptically named directories under `/var/lib/kubelet/pods/`. To keep things organized, we'll configure a custom storage location at `/data/k3s-pvc`.

**Detailed Instructions:** See the [Custom Storage Class Configuration Guide](./CUSTOM_STORAGE_CLASS.md) for complete setup steps.

**Quick Summary:**
```bash
# Create storage directory
sudo mkdir -p /data/k3s-pvc
sudo chmod 755 /data/k3s-pvc

# Apply custom storage class configuration
kubectl apply -f local-path-config.yaml

# Restart the provisioner (required!)
kubectl -n kube-system delete pod -l app=local-path-provisioner
```

After this setup, use `storageClassName: custom-local-path` in your manifests to store data in your preferred location.

### Deployment Flow

```
1. Deployment Script (k3s-deploy-tig.sh)
   ├─ Generate credentials (InfluxDB token, Grafana password)
   ├─ Generate self-signed TLS certificates
   ├─ Update manifest with generated secrets
   └─ Deploy manifest to K3S

2. Kubernetes Resources Created (in order)
   ├─ Namespace: tig-stack-k3s
   ├─ Secrets: credentials, TLS certificate, tokens
   ├─ ConfigMaps: environment vars, scripts, configs
   └─ PersistentVolumeClaims: storage provisioning

3. Application Deployment
   ├─ StatefulSet: InfluxDB (waits for PVC binding)
   ├─ Job: Init scripts (waits for InfluxDB ready)
   │   ├─ Create InfluxDB database
   │   └─ Create Grafana service account
   ├─ Deployments: Grafana, Explorer, Telegraf
   └─ Ingresses: HTTPS endpoints

4. Post-Deployment
   └─ Script creates Grafana SA token via API
       ├─ Saved to .k3s-credentials file
       └─ Stored in K3S secret
```

### Note on step 4.
The final step in the deployment consists of creating a Grafana Service Account Token.
This is done through the Grafana API once the container is started and thus requires name resolving for your Grafana container https://tig-grafana.tig-influx.test
So this will require you to conduct the following steps on your K3S cluster prior to running the deployment script:
- Determine your LoadBalancer (traefik) IP: `kubectl get svc -A | grep LoadBalancer` 
- Configure DNS or host file: https://tig-grafana.tig-influx.test with the external LoadBalancer IP-Address
- Verify 
If the DNS or host file isn't set prior to deployment the Token creation will fail. 

## DNS Configuration

Add these entries to your DNS server (like Pi-hole) or your workstation's hosts file, pointing to your K3s node's IP address:

```
<your-external-loadbalancer-ip>  tig-explorer.tig-influx.test
<your-external-loadbalancer-ip>  tig-influxdb.tig-influx.test
<your-external-loadbalancer-ip>  tig-grafana.tig-influx.test
```

For example, if your loadbalancer / traefik IP is `192.168.1.100`:
```
192.168.1.100  tig-explorer.tig-influx.test
192.168.1.100  tig-influxdb.tig-influx.test
192.168.1.100  tig-grafana.tig-influx.test
```

## Deployment

The entire stack deploys with a single command using the provided script and manifest file.

### Files Needed

1. **k3s-deploy-tig.sh** - Deployment script
2. **tig-stack-manifests.yaml** - Kubernetes resource definitions

```bash
# Clone repository
git clone https://github.com/EdwardVoermans/Extended-TIG-Stack-InfluxDB3-for-Kubernetes-Single-Node
cd Extended-TIG-Stack-InfluxDB3-for-Kubernetes-Single-Node
```
Make the script executable:
```bash
chmod +x k3s-deploy-tig.sh
```

### Run the Deployment

```bash
./k3s-deploy-tig.sh
```

Watch as the script automatically creates all necessary resources, sets up ingress routes, and starts your monitoring stack.

## Accessing the Stack

Once deployed, access your services via HTTPS:

- **Grafana**: https://tig-grafana.tig-influx.test
- **InfluxDB Explorer**: https://tig-explorer.tig-influx.test
- **InfluxDB API**: https://tig-influxdb.tig-influx.test

**Note:** The deployment uses self-signed certificates, so you'll need to accept the security warning in your browser.

## Working with the InfluxDB API

Typically, InfluxDB v3 runs internally within the Kubernetes namespace and isn't directly accessible from outside. However, this deployment includes an ingress configuration (item #26 in the manifest) that exposes the InfluxDB API on port 443 for use cases requiring direct database access.

### Making API Calls with Self-Signed Certificates

When using self-signed certificates, API calls require the `--insecure` or `-k` flag to bypass certificate verification.

**Query Example:**
```bash
curl --insecure https://tig-influxdb.tig-influx.test/api/v3/query_sql \
  --header "Authorization: Bearer YOUR_API_TOKEN" \
  --data '{"db": "local_system", "q": "select * from cpu limit 5"}'
```

**Health Check Example:**
```bash
curl -k https://tig-influxdb.tig-influx.test/health \
  --header "Authorization: Bearer YOUR_API_TOKEN"
```

### Retrieving Your API Tokens 
The deployment script creates an API token and stores it securely. You can retrieve it in several ways:
**From the credentials file:**
```bash
cat .k3s-credentials
```

**From Kubernetes secrets (recommended):**
```
InfluxDB3 Admin Token
  Retrieve with: kubectl -n tig-stack-k3s-dev get secret tig-credentials -o jsonpath='{.data.influxdb-token}' | base64 -d

Grafana Service Account Token
  Retrieve with: kubectl get secret -n tig-stack-k3s-dev grafana-sa-token -o jsonpath='{.data.token}' | base64 -d

Grafana Admin Password 
  Retrieve with: kubectl get secret -n tig-stack-k3s-dev tig-credentials -o jsonpath='{.data.grafana-admin-password}' | base64 -d
```

## Upgrading Applications

Thanks to persistent storage, you can safely upgrade any component of the stack without losing:
- Historical metrics data
- Grafana dashboards and data sources
- InfluxDB Explorer settings
- User configurations

Simply update the image versions in the manifest and reapply the configuration.

## Troubleshooting

**PVCs not binding:**
- Verify the local-path-provisioner was restarted after applying the storage class
- Check that `/data/k3s-pvc` exists and has proper permissions

**Cannot access services via HTTPS:**
- Confirm DNS entries or hosts file configuration
- Verify Traefik is running: `kubectl -n kube-system get pods | grep traefik`
- Check ingress resources: `kubectl -n tig-stack-k3s-dev get ingress`

**API calls failing with certificate errors:**
- Use `curl --insecure` or `curl -k` for self-signed certificates
- Alternatively, add the certificate to your system's trust store

## Authors and Credits
- **Author**: Edward Voermans (edward@voermans.com)
- **Credits**: Based on work by Suyash Joshi (sjoshi@influxdata.com)
- **GitHub**: [TIG-Stack-using-InfluxDB-3](https://github.com/InfluxCommunity/TIG-Stack-using-InfluxDB-3)

## Support
Issues and pull requests are welcome! Please feel free to suggest improvements or report bugs.
For issues and questions:
- Check the troubleshooting section (tbd)
- Contact: edward@voermans.com

## License
This project builds upon MIT license.
