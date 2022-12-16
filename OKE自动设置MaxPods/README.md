[返回OKE中文文档集](../README.md)

# OKE自动设置MaxPods

OKE可以在创建Pool时自定义设置MaxPods。(此方法并非Oracle官方支持，如果发生问题，无法等到Oracle Support的支持)
这种情况，在创建Pool时，在"show advanced options"的"Initialization script"里面输入下面脚本内容。

示例中，添加，`--max-pods=30`。请修改为想使用的数量。

```
#!/bin/bash
curl --fail -H "Authorization: Bearer Oracle" -L0 http://169.254.169.254/opc/v2/instance/metadata/oke_init_script | base64 --decode >/var/run/oke-init.sh
bash /var/run/oke-init.sh --kubelet-extra-args '--max-pods=30'
```



ToDo: 如果设置太多，会报如下错误，有待解决，

```
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create pod network sandbox k8s_nginx-deployment-5dc684ffb4-zh2qs_default_e46515c3-905f-46cf-addf-637cf283d72e_0(c8c7f4290ca433f6dec82fd99532a8e22cabf4ea7224fc9e65e30e899caa2c47): error adding pod default_nginx-deployment-5dc684ffb4-zh2qs to CNI network "cbr0": plugin type="flannel" failed (add): failed to allocate for range 0: no IP addresses available in range set: 10.244.2.1-10.244.2.126
```



[返回OKE中文文档集](../README.md)