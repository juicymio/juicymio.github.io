# Phoenix


## 0x00 前言
Windows+WSL2(Ubuntu22.04)下的exploit.education的phoenix虚拟机环境配置, 但不建议使用Windows配置环境, 比Linux多了不必要的麻烦.  
## 0x01 QEMU安装
官网amd的下载包只有Qcow2格式的可用了, 所以只能装一个QEMU. 去[QEMU官网](https://www.qemu.org)下载一个win版的安装包一路默认就可以了, 可以改个安装地址. 
## 0x02 虚拟机安装以及ssh
Linux下直接运行压缩包中的boot脚本即可, 但Windows不行, 我找到个[大佬博客](https://blog.lamarranet.com/index.php/exploit-education-phoenix-setup/#Installing_QEMU)提供了脚本.创建一个boot.ps1文件, 把下面的代码复制进去改下qemu的路径即可. 
```
D:\Program` Files\qemu\qemu-system-x86_64.exe `
    -kernel vmlinuz-4.9.0-8-amd64 `
    -initrd initrd.img-4.9.0-8-amd64 `
    -append "root=/dev/vda1" `
    -m 1024M `
    -netdev user,id=unet,hostfwd=tcp:127.0.0.1:2222-:22 `
    -device virtio-net,netdev=unet `
    -drive file=exploit-education-phoenix-amd64.qcow2,if=virtio,format=qcow2,index=0
```
第一行的路径是qemu的安装路径, 注意空格用\`代替. 
再用powershell运行一下这个脚本即可, 若提示禁止运行脚本则给一下脚本运行权限即可. 
下一步是ssh到Phoenix虚拟机上, 配置脚本中"hostfwd=tcp:127.0.0.1:2222-:22"指定了虚拟机监听本机的127.0.0.1:2222端口, 如果你使用普通虚拟机, 则直接ssh到这个端口即可, 可以`ssh -p2222 user@localhost` , 用户名和密码都是user, 配置环境到这里就结束了. 但如果你跟我一样使用的是WSL2, 并且没有升级到Win11所以WSL没有固定的ip地址, 那还需要再操作一下. 打开Powershell输入`Get-NetIPInterface`, 找到vEthernet(WSL), 记下前面的ifIndex, 然后在刚才的脚本前面加上两句.
```powershell
$WSL_ip = (Get-NetIPAddress -InterfaceIndex 76 -AddressFamily Ipv4).IPAddress
$WSL_ip += ":2222"
```
Get-NetIPAddress的文档: https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-netipaddress?view=windowsserver2022-ps  
把76改成你的vEthernet(WSL)的ifIndex. 再把hostfwd那里的127.0.0.1:2222替换为$WSL_ip即可. 
然后你还要在WSL2里再获取这个ip地址, 如果你给WSL2配置过代理, 那应该获取过这个ip, 如果没有, 在你的WSL2的~/.bashrc里加上这句:
```sh
host_ip=$(cat /etc/resolv.conf |grep "nameserver" |cut -f 2 -d " ")
```
然后`source ~/.bashrc`  
在WSL2里ssh的时候输入`ssh -p2222 user@$host_ip`即可. 
