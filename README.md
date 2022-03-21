# Keycloak Lab

This is a lab environment showing how to deploy keycloak with clustering enabled.

## Create VMs

Create **cloud-init.yaml** and adjust it to your environment:

```yaml
#cloud-config

users:
- name: nicolas
  gecos: Nicolas MASSE
  groups: wheel
  lock_passwd: false
  ssh_authorized_keys:
  - ssh-ed25519 ABC123... nmasse@redhat.com

runcmd:
- [ "sed", "-i.post-install", "-e", "s/^%wheel\tALL=(ALL)\tALL/%wheel  ALL=(ALL)       NOPASSWD: ALL/", "/etc/sudoers" ]
- [ "sed", "-i.post-install", "-e", "s/PasswordAuthentication yes/PasswordAuthentication no/", "/etc/ssh/sshd_config" ]
- [ "systemctl", "restart", "--no-block", "sshd" ]
```

Download the Red Hat Enterprise Linux 8.5 KVM Guest Image.

```sh
sudo curl -o /var/lib/libvirt/images/sso/rhel-8.5-x86_64-kvm.qcow2 http://admin.itix.lab:8080/misc/rhel8/rhel-8.5-x86_64-kvm.qcow2
```

Deploy the Virtual Machines.

```sh
sudo mkdir -p /var/lib/libvirt/images/sso
sudo cloud-localds -f vfat /var/lib/libvirt/images/sso/cloud-init.img cloud-init.yaml

function vm () {
  name="$1"
  disk="$2"
  vcpu="$3"
  memory="$4"
  mac="$5"

  sudo virt-install --name $name --memory $memory --vcpus $vcpu --cpu host-passthrough --os-variant rhel8.4 --disk path=/var/lib/libvirt/images/sso/$name.qcow2,backing_store=/var/lib/libvirt/images/sso/rhel-8.5-x86_64-kvm.qcow2,size=$disk --network network=lab,portgroup=lab7,mac.address=$mac --console pty,target.type=virtio --serial pty --disk path=/var/lib/libvirt/images/sso/cloud-init.img,readonly=on --sysinfo system.serial=ds=nocloud --import --noautoconsole 
}

vm lb 10 1 2048 02:01:07:00:07:28
vm sso1 20 2 4096 02:01:07:00:07:29
vm sso2 20 2 4096 02:01:07:00:07:2a
vm sso3 20 2 4096 02:01:07:00:07:2b
vm db 30 2 4096 02:01:07:00:07:2c
```

Confirm all VMs are up and DNS records are ok.

```sh
ping -c1 192.168.7.40
ping -c1 192.168.7.41
ping -c1 192.168.7.42
ping -c1 192.168.7.43
ping -c1 192.168.7.44

dig +short lb.itix.lab
dig +short sso1.itix.lab
dig +short sso2.itix.lab
dig +short sso3.itix.lab
dig +short db.itix.lab
```

## Preparation

Register with RHN.

```sh
sudo subscription-manager list --available --matches '\*SSO\*' |grep -A60 "30 Day Product Trial of Red Hat JBoss Enterprise Application Platform, Self-Supported"
# Pool ID is 8a85f9a07db4828b017dc5184e5f0863
for vm in lb sso1 sso2 sso3 db; do
    ssh -o StrictHostKeyChecking=off nicolas@$vm.itix.lab uname -a
    ssh nicolas@$vm.itix.lab sudo subscription-manager register --username=rhn-support-nmasse --password=S3cr3t
    ssh nicolas@$vm.itix.lab sudo subscription-manager attach --pool=8a85f9a07db4828b017dc5184e5f0863
done
```

Download the follwing files and place them in the **files** folder.

* rh-sso-7.5.0-server-dist.zip
* rh-sso-7.5.1-patch.zip
* rhsso-1974.zip
* rhsso-2054.zip

Create a vault to store all your secrets.

```sh
ansible-vault create group_vars/all/vault.yaml
```

And add the following content:

```yaml
db_password: secret
keycloak_admin_password: secret
```

Update **inventory.yaml** to match your environment.

Deploy Keycloak.

```sh
ansible-playbook -i inventory.yaml --ask-vault-pass install.yaml
```

## Cleanup

If you need, you can remove all VMs once finished.

```sh
for vm in lb sso1 sso2 sso3 db; do
  sudo virsh destroy $vm
  sudo virsh undefine $vm
  sudo rm /var/lib/libvirt/images/sso/$vm.qcow2
done
```