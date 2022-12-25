[返回OKE中文文档集](../README.md)

# OKE节点Cloud-Init多种设置示例

```
#!/bin/bash
curl --fail -H "Authorization: Bearer Oracle" -L0 http://169.254.169.254/opc/v2/instance/metadata/oke_init_script | base64 --decode >/var/run/oke-init.sh

sudo dd iflag=direct if=/dev/sda of=/dev/null count=1
echo "1" | sudo tee /sys/class/block/sda/device/rescan
echo "y" | sudo /usr/libexec/oci-growfs

cat <<EOF | sudo tee /etc/security/limits.d/20-nproc.conf
*          soft    nproc     65535
*          hard    nproc     65535
root       soft    nproc     unlimited
root       hard    nproc     unlimited
EOF

cat <<EOF | sudo tee /etc/security/limits.d/20-nofile.conf
*          soft    nofile     65535
*          hard    nofile     65535
root       soft    nofile     65535
root       hard    nofile     65535
EOF

cat <<EOF | sudo tee -a /etc/profile
ulimit -SHn 65535
EOF
source /etc/profile

bash /var/run/oke-init.sh --kubelet-extra-args "--register-with-taints=env=cluster-autoscaler:NoSchedule,env1=network-pending:NoSchedule"

sleep 180s
reboot
```



[返回OKE中文文档集](../README.md)
