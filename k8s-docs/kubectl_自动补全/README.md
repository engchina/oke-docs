[返回OKE中文文档集](../../README.md)

[返回kubernetes中文文档集](../README.md)

# kubectl_自动补全

执行下面命令设置`kubect`和`k`(kubect的alias)实现自动补全，

```
# 在你的 bash shell 中永久地添加自动补全
echo "source <(kubectl completion bash)" >> ~/.bashrc 
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

实现的效果是，当我们输入`kubectl`或者`k`的命令后，按下`tab`键，会自动提示命令的补全，提高输入的效率，

示例1，

![image-20221023083841793](images/image-20221023083841793.png)

示例2，

![image-20221023083950210](images/image-20221023083950210.png)

示例3，

![image-20221023084042840](images/image-20221023084042840.png)

完结！

[返回kubernetes中文文档集](../README.md)

[返回OKE中文文档集](../../README.md)
