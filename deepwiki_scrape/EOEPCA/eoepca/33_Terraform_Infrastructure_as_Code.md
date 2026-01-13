# Terraform Infrastructure as Code

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [bin/install-kubeseal.sh](bin/install-kubeseal.sh)
- [bin/install-rke.sh](bin/install-rke.sh)
- [creodias/.gitignore](creodias/.gitignore)
- [creodias/.terraform/modules/modules.json](creodias/.terraform/modules/modules.json)
- [creodias/README.md](creodias/README.md)
- [creodias/deployCREODIAS.sh](creodias/deployCREODIAS.sh)
- [creodias/eoepca.tf](creodias/eoepca.tf)
- [creodias/eoepca.tfvars](creodias/eoepca.tfvars)
- [creodias/modules/compute/main.tf](creodias/modules/compute/main.tf)
- [creodias/modules/compute/nfs-setup.sh](creodias/modules/compute/nfs-setup.sh)
- [creodias/modules/compute/nfs.tf](creodias/modules/compute/nfs.tf)
- [creodias/modules/compute/outputs.tf](creodias/modules/compute/outputs.tf)
- [creodias/modules/compute/variables.tf](creodias/modules/compute/variables.tf)
- [creodias/modules/loadbalancer/main.tf](creodias/modules/loadbalancer/main.tf)
- [creodias/terraform.tfstate](creodias/terraform.tfstate)
- [creodias/terraform.tfstate.backup](creodias/terraform.tfstate.backup)
- [creodias/variables.tf](creodias/variables.tf)
- [kubernetes/cluster.7z](kubernetes/cluster.7z)
- [kubernetes/create-cluster-config.sh](kubernetes/create-cluster-config.sh)

</details>



## Overview

This document describes the Terraform modules and configurations used to provision the EOEPCA infrastructure on OpenStack (CREODIAS/Cloudferro platform). The Terraform code creates all virtual machines, networking, security groups, storage volumes, and load balancers required for a Kubernetes cluster deployment.

For information about setting up the Kubernetes cluster after infrastructure provisioning, see [Kubernetes Cluster Setup](#8.1). For details about the network architecture and security groups, see [Network Architecture](#8.3).

## Terraform Module Structure

The Terraform configuration is organized into modular components under [creodias/]():

```mermaid
graph TB
    subgraph "Root Module"
        Main["eoepca.tf<br/>Root Configuration"]
        Vars["variables.tf<br/>Variable Definitions"]
        TVars["eoepca.tfvars<br/>Deployment Values"]
    end
    
    subgraph "Module: network"
        NetMod["modules/network/<br/>Network Infrastructure"]
    end
    
    subgraph "Module: ips"
        IpsMod["modules/ips/<br/>Floating IP Allocation"]
    end
    
    subgraph "Module: compute"
        CompMod["modules/compute/main.tf<br/>VM Instances & Security"]
        NfsMod["modules/compute/nfs.tf<br/>NFS Server"]
        CompVars["modules/compute/variables.tf"]
        CompOut["modules/compute/outputs.tf"]
    end
    
    subgraph "Module: loadbalancer"
        LbMod["modules/loadbalancer/main.tf<br/>Load Balancer"]
    end
    
    Main -->|uses| NetMod
    Main -->|uses| IpsMod
    Main -->|uses| CompMod
    Main -->|uses| LbMod
    
    IpsMod -->|depends on| NetMod
    CompMod -->|depends on| IpsMod
    LbMod -->|depends on| CompMod
    LbMod -->|depends on| NetMod
    
    TVars -->|provides values| Vars
    Vars -->|defines| Main
```

**Sources:** [creodias/eoepca.tf:1-146](), [creodias/.terraform/modules/modules.json:1]()

## Infrastructure Components

### Virtual Machine Types

The Terraform configuration provisions multiple categories of VMs with specific roles:

| VM Type | Resource Name | Count Variable | Purpose |
|---------|---------------|----------------|---------|
| Bastion | `openstack_compute_instance_v2.bastion` | `number_of_bastions` | SSH gateway to cluster |
| K8s Master | `openstack_compute_instance_v2.k8s_master_no_floating_ip` | `number_of_k8s_masters_no_floating_ip` | Control plane nodes |
| K8s Workers | `openstack_compute_instance_v2.k8s_node_no_floating_ip` | `number_of_k8s_nodes_no_floating_ip` | Worker nodes |
| ETCD | `openstack_compute_instance_v2.etcd` | `number_of_etcd` | Dedicated etcd nodes |
| NFS Server | `openstack_compute_instance_v2.eoepca_nfs` | 1 | Shared storage |
| GlusterFS | `openstack_compute_instance_v2.glusterfs_node_no_floating_ip` | `number_of_gfs_nodes_no_floating_ip` | Distributed storage (optional) |

**Sources:** [creodias/modules/compute/main.tf:116-437](), [creodias/modules/compute/nfs.tf:1-70]()

### Deployment Topology Diagram

```mermaid
graph TB
    subgraph "External Network"
        Internet["Internet<br/>(external3)"]
    end
    
    subgraph "OpenStack CREODIAS"
        subgraph "Floating IPs"
            BastionFIP["bastion_fips<br/>185.52.192.185"]
            LBFIP["loadbalancer_fips<br/>185.52.192.231"]
        end
        
        subgraph "Private Network: 192.168.123.0/24"
            Router["openstack_networking_router_v2.k8s"]
            
            subgraph "Security Groups"
                SGBastion["develop-bastion<br/>SSH access"]
                SGMaster["develop-k8s-master<br/>API:6443"]
                SGK8s["develop-k8s<br/>Internal cluster"]
                SGWorker["develop-k8s-worker<br/>NodePort 30000-32767"]
                SGLB["develop-lb<br/>HTTP/HTTPS"]
            end
            
            subgraph "Compute Instances"
                Bastion["bastion-1<br/>192.168.123.24<br/>flavor: eo1.xsmall"]
                Master["k8s-master-nf-1<br/>192.168.123.15<br/>flavor: eo2.large"]
                Worker1["k8s-node-nf-1<br/>192.168.123.16<br/>flavor: eo2.xlarge"]
                WorkerN["k8s-node-nf-N<br/>...5 more workers<br/>flavor: eo2.xlarge"]
                NFS["nfs<br/>192.168.123.14<br/>flavor: eo2.large"]
            end
            
            subgraph "Load Balancer"
                LB["openstack_lb_loadbalancer_v2.k8s"]
                HTTPSPool["lb_pool_v2.https<br/>:443→:31443"]
                HTTPPool["lb_pool_v2.http<br/>:80→:31080"]
            end
            
            subgraph "Storage Volumes"
                NFSVol["nfs-expansion<br/>1024GB SSD<br/>/dev/sdb"]
            end
        end
        
        subgraph "eodata Network"
            EOData["eodata<br/>10.111.0.0/16<br/>CloudFerro Data"]
        end
    end
    
    Internet -->|SSH:22| BastionFIP
    Internet -->|HTTP/HTTPS| LBFIP
    
    BastionFIP --> Bastion
    LBFIP --> LB
    
    Router -->|routes| Internet
    
    LB --> HTTPSPool
    LB --> HTTPPool
    HTTPSPool -->|31443| Worker1
    HTTPSPool -->|31443| WorkerN
    HTTPPool -->|31080| Worker1
    HTTPPool -->|31080| WorkerN
    
    Bastion -.->|SSH gateway| Master
    Bastion -.->|SSH gateway| Worker1
    Bastion -.->|SSH gateway| NFS
    
    NFSVol -->|attached| NFS
    Worker1 -.->|mount NFS| NFS
    WorkerN -.->|mount NFS| NFS
    
    Worker1 -.->|secondary network| EOData
    WorkerN -.->|secondary network| EOData
```

**Sources:** [creodias/terraform.tfstate:96-1100](), [creodias/eoepca.tfvars:1-57](), [creodias/README.md:76-97]()

## Module: Network

The network module creates the private network infrastructure for the cluster:

```mermaid
graph LR
    ExtNet["external_net<br/>31d7e67a-b30a-43f4-8b06-1667c70ba90d"]
    
    Router["openstack_networking_router_v2.k8s<br/>router_id"]
    Network["openstack_networking_network_v2<br/>network_name: develop"]
    Subnet["openstack_networking_subnet_v2<br/>subnet_cidr: 192.168.123.0/24<br/>dns_nameservers: 8.8.8.8, 8.8.4.4"]
    
    ExtNet -->|external_gateway| Router
    Router -->|routes to| Subnet
    Subnet -->|part of| Network
```

**Configuration:**
- Network name is set via `cluster_name` variable
- Subnet CIDR configurable in [eoepca.tfvars:10]()
- DNS nameservers configurable in [eoepca.tfvars:11]()
- Router provides NAT to external network

**Sources:** [creodias/eoepca.tf:6-16](), [creodias/eoepca.tfvars:6-11]()

## Module: IPs

The IPs module allocates floating IPs from the external pool for public-facing resources:

```mermaid
graph TB
    Pool["floatingip_pool<br/>external3"]
    
    subgraph "Floating IP Resources"
        BastionFIP["openstack_networking_floatingip_v2.bastion<br/>count: number_of_bastions"]
        MasterFIP["openstack_networking_floatingip_v2.k8s_master<br/>count: number_of_k8s_masters"]
        NodeFIP["openstack_networking_floatingip_v2.k8s_node<br/>count: number_of_k8s_nodes"]
    end
    
    subgraph "Associations"
        BastionAssoc["openstack_compute_floatingip_associate_v2.bastion"]
        MasterAssoc["openstack_compute_floatingip_associate_v2.k8s_master"]
        NodeAssoc["openstack_compute_floatingip_associate_v2.k8s_node"]
    end
    
    Pool -->|allocates from| BastionFIP
    Pool -->|allocates from| MasterFIP
    Pool -->|allocates from| NodeFIP
    
    BastionFIP -->|associates to| BastionAssoc
    MasterFIP -->|associates to| MasterAssoc
    NodeFIP -->|associates to| NodeAssoc
```

**Typical Deployment:**
- Bastion: 1 floating IP (`number_of_bastions = 1`)
- Masters: 0 floating IPs (uses `number_of_k8s_masters_no_floating_ip` instead)
- Workers: 0 floating IPs (uses `number_of_k8s_nodes_no_floating_ip` instead)
- Load Balancer: 1 floating IP (created in loadbalancer module)

**Sources:** [creodias/eoepca.tf:18-29](), [creodias/eoepca.tfvars:18-33]()

## Module: Compute

The compute module creates all VM instances and security infrastructure.

### Security Groups

Security groups control network access to VMs:

```mermaid
graph TB
    subgraph "Security Group Hierarchy"
        SGBastion["openstack_networking_secgroup_v2.bastion<br/>SSH from bastion_allowed_remote_ips"]
        SGMaster["openstack_networking_secgroup_v2.k8s_master<br/>TCP:6443 from master_allowed_remote_ips"]
        SGK8s["openstack_networking_secgroup_v2.k8s<br/>All traffic within group<br/>SSH from k8s_allowed_remote_ips<br/>Egress to k8s_allowed_egress_ips"]
        SGWorker["openstack_networking_secgroup_v2.worker<br/>TCP:30000-32767 from 0.0.0.0/0"]
    end
    
    subgraph "Applied To VMs"
        Bastion["bastion<br/>security_groups:<br/>- bastion<br/>- k8s"]
        Master["k8s-master<br/>security_groups:<br/>- k8s-master<br/>- k8s"]
        Worker["k8s-node<br/>security_groups:<br/>- k8s-worker<br/>- k8s"]
    end
    
    SGBastion -->|applied to| Bastion
    SGMaster -->|applied to| Master
    SGK8s -->|applied to| Bastion
    SGK8s -->|applied to| Master
    SGK8s -->|applied to| Worker
    SGWorker -->|applied to| Worker
```

**Security Group Resources:**
- `openstack_networking_secgroup_v2.bastion` - [creodias/modules/compute/main.tf:31-36]()
- `openstack_networking_secgroup_v2.k8s_master` - [creodias/modules/compute/main.tf:14-18]()
- `openstack_networking_secgroup_v2.k8s` - [creodias/modules/compute/main.tf:49-53]()
- `openstack_networking_secgroup_v2.worker` - [creodias/modules/compute/main.tf:81-85]()

**Sources:** [creodias/modules/compute/main.tf:14-96]()

### Compute Instance Creation

VM instances are created with metadata for Kubespray configuration:

```mermaid
graph TB
    subgraph "Instance Creation Flow"
        Keypair["openstack_compute_keypair_v2.k8s<br/>public_key from public_key_path"]
        
        subgraph "Master Instances"
            MasterNoFIP["openstack_compute_instance_v2.k8s_master_no_floating_ip<br/>metadata.kubespray_groups:<br/>'etcd,kube-master,k8s-cluster,vault,no-floating'"]
        end
        
        subgraph "Worker Instances"
            WorkerNoFIP["openstack_compute_instance_v2.k8s_node_no_floating_ip<br/>metadata.kubespray_groups:<br/>'kube-node,k8s-cluster,no-floating'"]
        end
        
        subgraph "Provisioning"
            RKEScript["provisioner: file<br/>rke-node-setup.sh"]
            RKEExec["provisioner: remote-exec<br/>Execute setup script"]
        end
    end
    
    Keypair -->|key_pair| MasterNoFIP
    Keypair -->|key_pair| WorkerNoFIP
    
    MasterNoFIP -->|connection via bastion| RKEScript
    RKEScript --> RKEExec
```

**Key Instance Attributes:**
- `image_name` - OS image (default: "Ubuntu 18.04 LTS")
- `flavor_id` - VM size (e.g., "eo2.large" for masters, "eo2.xlarge" for workers)
- `key_pair` - SSH key for access
- `network.name` - Attached to private network
- `security_groups` - List of security groups
- `metadata.kubespray_groups` - Used by RKE for node role assignment

**Sources:** [creodias/modules/compute/main.tf:384-437](), [creodias/eoepca.tfvars:39-47]()

### NFS Server Provisioning

The NFS server provides shared persistent storage for the cluster:

```mermaid
graph TB
    subgraph "NFS Resources"
        NFSInstance["openstack_compute_instance_v2.eoepca_nfs<br/>name: develop-nfs<br/>flavor: eo2.large<br/>ip: 192.168.123.14"]
        
        NFSVolume["openstack_blockstorage_volume_v2.nfs_expansion<br/>size: nfs_disk_size (1024GB)<br/>volume_type: SSD"]
        
        VolumeAttach["openstack_compute_volume_attach_v2.volume_attachment_nfs<br/>device: /dev/sdb"]
        
        NFSSetup["provisioner: remote-exec<br/>nfs-setup.sh"]
    end
    
    subgraph "NFS Setup Script Actions"
        InstallNFS["apt-get install<br/>nfs-kernel-server"]
        CreateRoot["mkdir /data"]
        Partition["fdisk /dev/sdb<br/>mkfs -t ext4 /dev/sdb1"]
        Mount["mount /dev/sdb1 /data<br/>add to /etc/fstab"]
        CreateExports["mkdir /data/userman<br/>mkdir /data/proc<br/>mkdir /data/resman<br/>mkdir /data/dynamic"]
        ExportConfig["update /etc/exports<br/>restart nfs-kernel-server"]
    end
    
    NFSVolume -->|attached as| VolumeAttach
    VolumeAttach -->|/dev/sdb on| NFSInstance
    NFSInstance -->|executes| NFSSetup
    
    NFSSetup --> InstallNFS
    InstallNFS --> CreateRoot
    CreateRoot --> Partition
    Partition --> Mount
    Mount --> CreateExports
    CreateExports --> ExportConfig
```

**NFS Export Directories:**
- `/data/userman` - User management data
- `/data/proc` - Processing data
- `/data/resman` - Resource management data
- `/data/dynamic` - Dynamic workspace data

**Export Configuration:**
- All exports use `*(rw,no_root_squash,no_subtree_check)`
- Accessible from all cluster nodes

**Sources:** [creodias/modules/compute/nfs.tf:1-70](), [creodias/modules/compute/nfs-setup.sh:1-44](), [creodias/eoepca.tfvars:56]()

## Module: Load Balancer

The load balancer module creates an OpenStack Octavia load balancer to route external traffic:

```mermaid
graph TB
    subgraph "Load Balancer Components"
        LB["openstack_lb_loadbalancer_v2.k8s<br/>name: develop-lb<br/>vip_network_id: network_id"]
        
        subgraph "Listeners"
            HTTPSListener["openstack_lb_listener_v2.https<br/>protocol: HTTPS<br/>protocol_port: 443"]
            HTTPListener["openstack_lb_listener_v2.http<br/>protocol: HTTP<br/>protocol_port: 80"]
        end
        
        subgraph "Pools"
            HTTPSPool["openstack_lb_pool_v2.https<br/>lb_method: ROUND_ROBIN<br/>persistence: SOURCE_IP"]
            HTTPPool["openstack_lb_pool_v2.http<br/>lb_method: ROUND_ROBIN<br/>persistence: SOURCE_IP"]
        end
        
        subgraph "Members"
            HTTPSMembers["openstack_lb_members_v2.https<br/>k8s_node_ips → port 31443"]
            HTTPMembers["openstack_lb_members_v2.http<br/>k8s_node_ips → port 31080"]
        end
        
        FIP["openstack_networking_floatingip_v2.loadbalancer<br/>pool: floatingip_pool"]
    end
    
    LB --> HTTPSListener
    LB --> HTTPListener
    
    HTTPSListener --> HTTPSPool
    HTTPListener --> HTTPPool
    
    HTTPSPool --> HTTPSMembers
    HTTPPool --> HTTPMembers
    
    FIP -->|associates to| LB
```

**Load Balancer Forwarding:**
- External HTTPS (443) → NodePort 31443 on all worker nodes
- External HTTP (80) → NodePort 31080 on all worker nodes
- Connects to `k8s_node_ips`, `k8s_node_hm_ips`, and `k8s_node_ws_ips`

**Security:**
- Load balancer uses security groups `develop-lb` and `develop-k8s`
- Allows ingress on ports 80 and 443
- Egress to all destinations (IPv4 and IPv6)

**Sources:** [creodias/modules/loadbalancer/main.tf:1-184](), [creodias/eoepca.tf:83-93]()

## Deployment Configuration

### Variable Configuration File

The [eoepca.tfvars]() file contains deployment-specific values:

| Variable | Purpose | Default Value |
|----------|---------|---------------|
| `cluster_name` | Deployment identifier | `"develop"` |
| `external_net` | OpenStack external network UUID | `"31d7e67a-b30a-43f4-8b06-1667c70ba90d"` |
| `subnet_cidr` | Internal network CIDR | `"192.168.123.0/24"` |
| `number_of_bastions` | Bastion host count | `1` |
| `number_of_k8s_masters_no_floating_ip` | Master count | `1` |
| `number_of_k8s_nodes_no_floating_ip` | Worker count | `6` |
| `flavor_bastion` | Bastion VM flavor | `"14"` (eo1.xsmall) |
| `flavor_k8s_master` | Master VM flavor | `"20"` (eo2.large) |
| `flavor_k8s_node` | Worker VM flavor | `"21"` (eo2.xlarge) |
| `image` | OS image name | `"Ubuntu 18.04 LTS"` |
| `nfs_disk_size` | NFS volume size in GB | `1024` |

**Sources:** [creodias/eoepca.tfvars:1-57](), [creodias/variables.tf:1-233]()

### OpenStack Authentication

Terraform uses the OpenStack client for authentication. Configuration is loaded from `clouds.yaml`:

**Example clouds.yaml:**
```yaml
clouds:
  eoepca:
    auth:
      auth_url: https://cf2.cloudferro.com:5000/v3
      username: "user@example.com"
      project_name: "eoepca"
      project_id: d86660d4a1a443579c71096771a8104c
      user_domain_name: "cloud_xxxxx"
      password: "password"
    region_name: "RegionOne"
    interface: "public"
    identity_api_version: 3
```

**Location:** Must be placed in one of:
- `./clouds.yaml` (current directory)
- `$HOME/.config/openstack/clouds.yaml`
- `/etc/openstack/clouds.yaml`

**Sources:** [creodias/README.md:29-53]()

## Deployment Process

### Deployment Script Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as deployCREODIAS.sh
    participant TF as terraform
    participant OS as OpenStack API
    
    User->>Script: ./deployCREODIAS.sh [apply|destroy]
    Script->>Script: Check OS_CLOUD env var
    Script->>TF: terraform init
    
    Note over Script,TF: Phase 1: Keypair
    Script->>TF: terraform apply<br/>-target=module.compute.openstack_compute_keypair_v2.k8s
    TF->>OS: Create keypair: kubernetes-{cluster_name}
    OS-->>TF: Keypair created
    
    Script->>OS: Wait for keypair ready<br/>openstack keypair show
    OS-->>Script: Keypair confirmed
    
    Note over Script,TF: Phase 2: Full Deployment
    Script->>TF: terraform apply<br/>-var-file=eoepca.tfvars
    TF->>OS: Create network resources
    TF->>OS: Create compute instances
    TF->>OS: Create load balancer
    TF->>OS: Attach volumes
    OS-->>TF: Infrastructure provisioned
    TF-->>Script: Outputs (IPs, IDs)
    Script-->>User: Deployment complete
```

**Deployment Steps:**
1. Set `OS_CLOUD` environment variable to match `clouds.yaml` project
2. Run [deployCREODIAS.sh]():
   ```bash
   cd creodias
   export OS_CLOUD=eoepca
   ./deployCREODIAS.sh apply
   ```
3. Terraform creates keypair first, then remaining resources
4. Script waits for keypair availability before proceeding
5. Full deployment completes with all VMs, network, and storage

**Sources:** [creodias/deployCREODIAS.sh:1-52](), [creodias/README.md:58-68]()

## Terraform Outputs

The Terraform configuration produces outputs used by subsequent deployment steps:

```mermaid
graph LR
    subgraph "Terraform Outputs"
        BastionFIPs["bastion_fips<br/>List of bastion floating IPs"]
        LBFIPs["loadbalancer_fips<br/>List of LB floating IPs"]
        MasterIPs["k8s_master_ips<br/>Private IPs of masters"]
        NodeIPs["k8s_node_ips<br/>Private IPs of workers"]
        NFSIP["nfs_ip_address<br/>Private IP of NFS"]
        SubnetCIDR["subnet_cidr<br/>Internal network CIDR"]
    end
    
    subgraph "Consumed By"
        RKEConfig["create-cluster-config.sh<br/>Generates cluster.yml"]
        BastionVPN["bastion-vpn.sh<br/>Establishes sshuttle VPN"]
        Kubectl["kubectl<br/>Cluster management"]
    end
    
    BastionFIPs -->|used by| RKEConfig
    MasterIPs -->|used by| RKEConfig
    NodeIPs -->|used by| RKEConfig
    BastionFIPs -->|used by| BastionVPN
    SubnetCIDR -->|used by| BastionVPN
    BastionFIPs -->|SSH gateway| Kubectl
```

**Output Commands:**
```bash
# View all outputs as JSON
terraform output -json

# View specific output
terraform output bastion_fips
```

**Output Usage Examples:**

From [create-cluster-config.sh:21-45]():
```bash
# Extract master nodes from terraform state
master_nodes=$(terraform output -state=../creodias/terraform.tfstate -json | \
  jq -r '.k8s_master_ips.value[]')

# Extract worker nodes from terraform state  
worker_nodes=$(terraform output -state=../creodias/terraform.tfstate -json | \
  jq -r '.k8s_node_ips.value[]')

# Extract bastion IP from terraform state
bastion=$(terraform output -state=../creodias/terraform.tfstate -json | \
  jq -r '.bastion_fips.value[]')
```

**Sources:** [creodias/eoepca.tf:95-146](), [creodias/modules/compute/outputs.tf:1-20](), [kubernetes/create-cluster-config.sh:20-80]()

## Provider Configuration

The Terraform configuration uses the OpenStack provider with Octavia support:

```hcl
provider "openstack" {
  version     = "~> 1.17"
  use_octavia = true
}
```

**Provider Features:**
- Version constraint: `~> 1.17` (1.17.x)
- `use_octavia = true` - Uses Octavia for load balancer resources
- Authentication via OpenStack client (`clouds.yaml`)

**Sources:** [creodias/eoepca.tf:1-4]()

## Instance Provisioning with Remote Execution

Master and worker nodes execute provisioning scripts via SSH through the bastion host:

```mermaid
graph LR
    subgraph "Provisioning Connection"
        Local["Local Machine<br/>Terraform"]
        Bastion["Bastion Host<br/>bastion_fips[0]"]
        Target["Target Instance<br/>Master/Worker"]
    end
    
    subgraph "Provisioning Steps"
        FileProvisioner["provisioner 'file'<br/>rke-node-setup.sh → /tmp/"]
        RemoteExec["provisioner 'remote-exec'<br/>chmod +x<br/>execute setup script"]
    end
    
    Local -->|SSH via| Bastion
    Bastion -->|SSH to| Target
    Target -->|upload| FileProvisioner
    FileProvisioner -->|execute| RemoteExec
```

**Connection Configuration:**
```hcl
connection {
  type         = "ssh"
  user         = "${var.ssh_user}"
  private_key  = "${chomp(file(trimsuffix(var.public_key_path, ".pub")))}"
  host         = "${self.access_ip_v4}"
  bastion_host = var.bastion_fips[0]
}
```

**Provisioning Actions:**
- Uploads `rke-node-setup.sh` to `/tmp/`
- Makes script executable
- Executes script to configure Docker and user permissions for RKE

**Sources:** [creodias/modules/compute/main.tf:418-436]()

## State Management

Terraform state is stored in local state files:

| File | Purpose |
|------|---------|
| `terraform.tfstate` | Current infrastructure state |
| `terraform.tfstate.backup` | Previous state backup |

**State Contents:**
- All resource IDs and attributes
- Resource dependencies
- Output values
- Provider configurations

**State Format:** JSON with versioning (format version 4, Terraform 0.12.29)

**Important:** State files contain sensitive data and should not be committed to version control.

**Sources:** [creodias/terraform.tfstate:1-10](), [creodias/terraform.tfstate.backup:1-10]()

## Next Steps

After Terraform provisioning completes:

1. **Create Kubernetes Cluster Configuration:**
   ```bash
   cd kubernetes
   ./create-cluster-config.sh develop cluster.yml
   ```
   This generates `cluster.yml` for RKE using Terraform outputs.

2. **Establish Bastion VPN:**
   ```bash
   ../bin/bastion-vpn.sh
   ```
   Creates sshuttle VPN for cluster access.

3. **Deploy Kubernetes:** See [Kubernetes Cluster Setup](#8.1) for RKE deployment steps.

**Sources:** [creodias/README.md:126-128](), [kubernetes/create-cluster-config.sh:1-117]()