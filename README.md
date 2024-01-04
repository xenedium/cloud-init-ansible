# cloud-init-ansible

This is a simple POC to show how to use cloud-init and QEMU to create 3 VMs and deploy a Kubernetes cluster on them with Ansible. Vagrant is not used, but it could be used to make this POC more portable.

## Requirements

This POC was tested on Arch Linux with the following packages installed:

- ansible
- cdrkit (for genisoimage)
- qemu

Make sure to enable the `kvm` kernel module.

```bash
sudo pacman -Sy ansible cdrkit qemu-full
```

QEMU needs a bridge interface for this POC. Create one with the following command (if running with NetworkManager):

```bash
nmcli connection add type bridge ifname br0 stp no
nmcli connection add type bridge-slave ifname <your-interface> master br0

# Do not use this command if you are connected to the machine via SSH as it will disconnect you and you will not be able to reconnect
nmcli connection down <your-connection> # can be obtained with nmcli connection show --active
nmcli connection up bridge-br0
nmcli connection up bridge-slave-<your-interface>
```

## Usage

To run this POC, simply run the following command:

```bash
./hack/create-vms your-cloud-img.qcow2 'ssh-rsa AAAA... user@xenearch'
```

The first argument is the path to the cloud image you want to use. The second argument is the public key you want to use to connect to the VMs later on with SSH/Ansible.
Make sure to use a cloud image that supports cloud-init.

Once the script is done, you can start all the VMs with the generated `start-vms` script under the ```hack``` directory. This script will start all the VMs. You can then start the Ansible playbook with the following command:

```bash
# Make sure to change the IP addresses in the inventory file and add the SSH key
ansible-playbook playbooks/main.yml
```

Once the playbook is done, the Kubernetes cluster should be up and running. You can connect to the master node and run `kubectl get nodes` to see the nodes.

## How it works

The `create-cluster` script will create 3 VMs with QEMU and cloud-init. The cloud-init configuration is generated from within the script and contains the public key you provided as an argument. The VMs will be created with the following names: `node-1`, `node-2` and `node-3`.

The MAC addresses of the VMs are generated based the qemu-mac-hasher.py script. This script will generate a MAC address based on the VM name. This is done so that the VMs will always have the same MAC address and thus the same IP address. You can add a DHCP Binding to your router to make sure the VMs always get the same IP address. This is needed for the Ansible inventory.
The MAC addresses can be generated with the following command:

```bash
python hack/qemu-mac-hasher.py node-1
```

The VMs need a bridge interface to connect to the internet and to be accessible in the local network.

By default the main cloud init image will be resized to `25GB`. This can be changed in the `create-cluster` script.

Once the script is done, you can start all the VMs with the generated `start-vms` script under the ```hack``` directory. You can also start the VMs individually with the following command:

```bash
cd hack/
# Start node-1
qemu-system-x86_64 -enable-kvm -m $VM_MEMORY -vnc :0 -cpu host -smp cores=$VM_CORES,threads=$VM_THREADS -nic bridge,br=br0,mac=$(python qemu-mac-hasher.py node-1) -hda node-1.qcow2
```
