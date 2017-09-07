### Prerequisites

- Redhat Enterprise Enterprise Linux (RHEL) 7.x
  - Default security profile
  - Minimal Install
- VMware vCenter 5.5+ or higher
  - Administrator user required
- Linux or OSX machine to run Tectonic installer
  - With network access to vCenter server
- Tectonic installer 1.7.1
- Pre-allocated IP addresses for cluster and pre-created DNS records
- Optional: Load balancer to direct console and api traffic (F5, Netscaler, HAProxy)

## DNS and IP address allocation

Installation of Tectonic requires using static allocation of IP Addresses for VMware installation.

Prior to the start of setup create required DNS records. Below is a sample table of 2 master nodes, 3 etcd nodes and 2 worker nodes.

| Record | Type | Value |
|------|-------------|:-----:|
|tectonic.scottsumner.net | A | 192.168.1.91 |
|api.scottsumner.net | A | 192.168.1.92 |
|master-01.scottsumner.net | A | 192.168.1.110 |
|master-02.scottsumner.net | A | 192.168.1.111 |
|etcd-01.scottsumner.net | A | 192.168.1.115 |
|etcd-02.scottsumner.net | A | 192.168.1.116 |
|etcd-03.scottsumner.net | A | 192.168.1.117 |
|worker-01.scottsumner.net | A | 192.168.1.112 |
|worker-02.scottsumner.net | A | 192.168.1.113 |

In this example the virtual IP for console traffic will be 192.168.1.91, DNS resolvable at tectonic.scottsumner.net and the virtual IP for API traffic will be 192.168.92, DNS resolvable at api.scottsumner.net

## Generate SSH key for Tectonic installation

During the Tectonic installation you will need to provide SSH keys that will be used for accessing all the nodes within cluster during and after the installation. You can use an exist SSH key or generate a new one. This key needs to exist on the machine where the Tectonic installer will be executed.

1. On machine where Tectonic installer will be executed create ssh key by executing `ssh-keygen -t rsa -C "your_email@example.com"`
2. Confirm you have a id_rsa and id_rsa.pub file within ~/.ssh folder

## Creation of RHEL 7.x VM template

Prior to installation of Tectonic we need to create a RHEL 7.x VM template that will be used for the worker nodes. You will need to initial create worker nodes running Container Linux then you swap them out with RHEL worker nodes.

Download RHEL media
1. Login to Redhat Customer Portal at https://access.redhat.com
1. Click on Downloads link
1. Under Infrastructure Management click on Red Hat Enterprise Linux
1. Download Red Hat Enterprise Linux 7.4 Binary DVD
1. Upload rhel-server-7.4-x86_64-dvd.iso to whatever Datastore you keep ISOs

Create RHEL Virtual Machine
1. Within vCenter Create New Virtual Machine
1. At Select a creation type select Create a new virtual Machine
1. Name the virtual machine rhel74_minimal
1. Select the compute resource that will be used for install Tectonic Cluster
1. Select a datastore to store virtual machine
1. Select compatibility: ESXi 6.5 and later
1. Guest OS Family: Linux / Guest OS Version: Red Hat Enterprise Linux 7 (64-bit)
1. Set vCPU to 2
1. Set Memory to 8192
1. Set New Hard disk to 30 GB
1. Select Network that will also be used during Tectonic Cluster Install
1. New CD/DVD Drive change to Datastore ISO File and select rhel-server-7.4-x86_64-dvd.iso
1. Enable Connect at Power On
1. Click finish button

Install RHEL on Virtual Machine
1. Power on new VM
1. Open console to new VM
1. Select Install Red Hat Enterprise Linux 7.4
1. Select English / English (United States) and click Continue
1. Set Data & Time to the correct timezone
1. Ensure Software Selection is set to Minimal Install
1. Under System / Installation Destination select 30 GiB VMware Virtual disk
1. Click Begin Installation
1. Set Rood Password
1. Click reboot on installation completion

Prepare RHEL VM for use as Tectonic Worker nodes
1. Login to RHEL as reboot
1. Execute `nmcli dev show` and note GENERAL.DEVICE (e.g ens192)
1. Execute `nmtui`
1. Select Edit a connection
1. Select device (e.g. ens192) and click edit
1. For template will use Automatic
1. Ensure DNS servers point to DNS that can resolve pre-created DNS records
1. Ensure Automatically connect is enabled
1. Click OK and Back
1. Select Activate a connection
1. Set system hostname change it to worker-01
1. Reboot so hostname change takes effect
1. Enable RHEL subscription by executing `subscription-manager register --username <username> —password <password> --auto-attach`
1. Enabled the rhel-7-server-extras-rpm by executing `subscription-manager repos --enable=rhel-7-server-extras-rpms`
1. Update RHEL by executing `yum update -y`
1. Create kubernetes directory by excuting `mkdir /etc/kubernetes`
1. Install net-tools (optional) by executing `yum install net-tools -y`
1. Create core UNIX user used by Tectonic by executing `useradd -d /home/core core`
1. Create temporary password for core by executing `passwd core`
1. Enable sudo for core user by executing `usermod -aG wheel core`
1. On machine where Tectonic installer will be executed need to setup ssh key for Core user by executing `ssh-copy-id -i ~/.ssh/id_rsa.pub core@worker-01`
1. Login to RHEL VM as core
1. Download tectonic-release RPM by executing `curl -LJO http://yum.prod.coreos.systems/repo/tectonic-rhel/el7/x86_64/Packages/tectonic-release-7-2.el7.noarch.rpm`
1. Verify the signinature by executing `rpm -qip tectonic-release-1.6.2-4.el7.noarch.rpm`
1. Install the tectonic-worker RPM by executing `sudo yum localinstall tectonic-release-1.6.2-4.el7.noarch.rpm -y`
1. Fix tectonic.repo typo by executing `sudo sed -i 's|\.el7|el7|g' /etc/yum.repos.d/tectonic.repo`

Disable password login and Sudo password (optional)
1. Login to RHEL VM as root
1. Edit /etc/ssh/sshd_config
    1. ChallengeResponseAuthentication no
    1. PasswordAuthentication no
    1. UsePAM no
1. Restart ssh service `systemctl restart sshd.service`
1. Need to temporarily give write access to sudo config by executing `chmod +w /etc/sudoers`
1. Edit /etc/sudoers comment out `#%wheel  ALL=(ALL)       ALL` and uncomment `%wheel        ALL=(ALL)       NOPASSWD: ALL`

Convert RHEL VM into template
1. Shutdown RHEL VM by executing `sudo shutdown -h now`
1. Within vCenter convert to template the RHEL VM rhel74_minimal

## Install Tectonic Cluster on VMware

Refer to instruction on installing Tectonic on VMware here: https://github.com/coreos/tectonic-installer/blob/master/Documentation/install/vmware/vmware-terraform.md

## Replace CL Worker with RHEL Worker

The assumption is that kubectl is already installed on machine where Tectonic Installer was execute.  If not please install kubectl using these instructions: https://kubernetes.io/docs/tasks/tools/install-kubectl/

1. On Tectonic Installer machine copy generated/auth/kubeconfig to ~/.kube by executing `cp generated/auth/kubeconfig ~/.kube/config`
1. Execute `kubectl get nodes` to verify cluster is running
1. Execute `kubectl cluster-info` to verify cluster information
1. Execute `kubectl drain worker-01 --ignore-daemonsets --delete-local-data` to drain services on node we will replace with RHEL worker nodes
1. Execute `kubectl delete node worker-01` to remove container linux worker node
1. Within vCenter power of worker-01 VM
1. Within vCenter remove worker-01 VM from inventory
1. Within vCenter create New VM from rhel74_mimimal template
1. Select Folder used during Tectonic install (e.g. Tectonic171)
1. Name VM worker-01 (same name as old CL VM)
1. Select same ESXi host used previously during Tectonic install
1. Select same datastore used previously during Tectonic install
1. Power on worker-01 VM and open console and login as root
1. Use `nmtui` to edit connection / IPv4 / Manual switch ip address to the original IP of worker-01 (e.g. 192.168.1.112/24) and add correct Gateway (e.g. 192.168.1.1)
1. Set system hostname to worker-01 if its not already set.
1. Run `ip a` or `ipconfig -a` to confirm IP address is set correctly
1. Run `ping 192.168.1.1` to verify you can reach Gateway
1. On Tectonic Installer machine copy generated/auth/kubeconfig to new worker-01 node `scp generated/auth/kubeconfig core@worker-01:/tmp/kubeconfig`
1. On new worker-01 node move kubeconfig into /etc/kubernetes folder `sudo cp /tmp/kubeconfig /etc/kubernetes/kubeconfig`
1. To install tectonic-release execute `sudo yum -y install tectonic-worker --nogpgcheck`
1. Edit kubelet.env by executing `sudo vi /etc/kubernetes/kubelet.env` and change `KUBELET_IMAGE_TAG=v1.7.1_coreos.0`
1. Install Firewalld by executing `sudo yum -y install firewalld`
1. Start Firewalld by executing `sudo systemctl start firewalld`
1. Modify Firewalld zone by executing `sudo firewall-cmd  --zone=trusted --add-interface ens192` (ens192 might not be your interface)
1. Modify Firewalld default zone by executing `sudo firewall-cmd --set-default-zone=trusted`
1. Verify Firewalld settings by executing `sudo firewall-cmd --list-all`
1. Setup cluster DNS IP by executing:
   - cat > tectonic-worker << EOF
KUBERNETES_DNS_SERVICE_IP=10.3.0.10
EOF
sudo mv tectonic-worker /etc/sysconfig/
sudo sed -i 's|KUBERNETES_DNS_SERVICE_IP=|KUBERNETES_DNS_SERVICE_IP=10.3.0.10|g' \
     /etc/kubernetes/kubesettings-local.env
1. If the previous command doesn't work just create new file under /etc/kubernetes/kubesettings-local.env with the follow:
   KUBERNETES_DNS_SERVICE_IP=10.3.0.10
1. Also create /etc/sysconfig/tectonic-worker with the following:
   KUBERNETES_DNS_SERVICE_IP=10.3.0.10
   CLUSTER_DOMAIN=<base domain of Tectonic cluster>
1. Setup Flannel configuration by executing:
   - cat > 10-flannel.conf << EOF
{
    "name": "cbr0",
    "type": "flannel",
    "delegate": {
        "isDefaultGateway": true
        }
}
EOF
1. Create cni plugin net.d directory by executing `sudo mkdir -p /etc/kubernetes/cni/net.d/`
1. Copy new flannel config by executing `sudo mv 10-flannel.conf /etc/kubernetes/cni/net.d/`
1. Set SELinux to Permissive Mode by executing `sudo setenforce 0`
1. To ensure SELinux is disable on reboot modify /etc/sysconfig
1. Enable kubelet service by executing `sudo systemctl enable kubelet`
1. Start kubelet service by executing `sudo systemctl start kubelet`
1. Execute `sudo journalctl -f -u kubelet` to monitoring kubelet logs during startup
