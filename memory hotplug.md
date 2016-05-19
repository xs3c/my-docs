# CPU/memory hotplug

- 版本0.1

----
[toc]

## CPU hotplug


### 添加udev规则
由于目前镜像缺少qemu-ga, 所以需要创建udev规则来enable 新添加的cpu. 具体的规则如下:

在 /etc/udev/rules.d/ 目录下创建文件80­-­cpu­-hotplug.rules, 并将下列内容添加到这个文件
```
SUBSYSTEM==”cpu”, ACTION==”add”, TEST==”online”, ATTR{online}==”0”,
ATTR{online}=”1”
```
- CentOS6.x

对于咱们默认的6.5镜像, 已经默认添加这个规则了. 所以上面规则不需要.

- CentOS7.x 
CentOS7.0(ucloud 默认镜像)中, 并不默认包含该规则, 两个办法, 二选一:
	- 需要更新systemd的包到systemd-219-19.el7_2.9.x86_64版本. 
	- 参考上面自己添加udev规则
- ubuntu 12.04
镜像需要添加该规则
- ubuntu 14.04
镜像默认包含该规则

另外可以通过下面命令grep udev的默认规则, 来确定镜像上诉规则已经包含在udev的默认规则中:

```
➜  ~ grep -r -i "cpu" /lib/udev/rules.d/*                                                                                                                                                     
/lib/udev/rules.d/40-redhat.rules:# CPU hotadd request
/lib/udev/rules.d/40-redhat.rules:SUBSYSTEM=="cpu", ACTION=="add", TEST=="online", ATTR{online}=="0", ATTR{online}="1"
/lib/udev/rules.d/98-kexec.rules:SUBSYSTEM=="cpu", ACTION=="add", PROGRAM="/bin/systemctl try-restart kdump.service"
/lib/udev/rules.d/98-kexec.rules:SUBSYSTEM=="cpu", ACTION=="remove", PROGRAM="/bin/systemctl try-restart kdump.service"
```

一旦有镜像没有包含规则监控cpu的**ACTION=="add"**, 我们需要自己添加udev规则.


### xml 修改

将当前xml中vcpu element, 修改类似如下.
```
<vcpu placement='static' current='2'>32</vcpu>
```

current='2' 标识当前cpu只有两个
32表示: 当前虚拟机vcpu个数最大加到32个

## 命令添加cpu

- 添加cpu不仅要要修改运行中虚拟机的配置同时要修改虚拟机的xml文件中
`virsh setvcpus <domain id> 3 --live --config`
参考下列命令:
```
[root@localhost ~]# virsh setvcpus hotplug 4 --live --config

[root@localhost ~]# virsh vcpucount hotplug
maximum      config         4
maximum      live           4
current      config         4
current      live           4

[root@localhost ~]# virsh shutdown hotplug
Domain hotplug is being shutdown
[root@localhost ~]# virsh start hotplug
Domain hotplug started

[root@localhost ~]# virsh vcpucount hotplug
maximum      config         4
maximum      live           4
current      config         4
current      live           4
```

- 如果仅想修改在线机器的配置并不修改虚拟机的xml文件
`virsh setvcpus <domain id> 3 --live`
参考下列命令:
```
 [root@localhost ~]# virsh setvcpus hotplug 3 --live 
`[root@localhost ~]# virsh vcpucount hotplug
maximum      config         4
maximum      live           4
current      config         2
current      live           3
[root@localhost ~]# virsh shutdown hotplug
Domain hotplug is being shutdown
[root@localhost ~]# virsh list
 Id    Name                           State
----------------------------------------------------

[root@localhost ~]# virsh start hotplug
Domain hotplug started

[root@localhost ~]# virsh vcpucount hotplug
maximum      config         4
maximum      live           4
current      config         2
current      live           2

```

>注意: 
>- 这里3是vcpu的个数. 例如当前运行的虚拟机有2个vcpu, 我们需要给他添加一个vcpu时, 我们命令里面指定个数为3. 同理如果想一次添加两个vcpu 指定为4.
>- 如果我们先只指定了**--live**添加一次vcpu, 然后我们又使用**--live --config**添加一个cpu这种情况下, vcpu个数以最后一次修改为准, 并且同时修改虚拟机的xml配置文件


### 验证是否添加成功

验证是否成功分为两部分:

- 查看qemu是否成功添加cpu
`virsh vcpucount hotplug`
```
 
`[root@localhost ~]# virsh vcpucount hotplug
maximum      config         4
maximum      live           4
current      config         2
current      live           3
```

- 查看guest os 是否成功添加并且使能cpu
 登录到guest `cat /proc/cpuinfo` 查看cpu个数与qemu总共包含的live vcpu个数是否一致.
 `cat /proc/cpuinfo | grep -i "processor" | wc -l`

## Memory hotplug

### 添加udev规则

由于目前镜像缺少qemu-ga, 所以需要创建udev规则来enable 新添加的memory. 具体的规则如下:

在 /etc/udev/rules.d/ 目录下创建文件81­-­­mem-hotplug.rules, 并将下列内容添加到这个文件:

```
[root@10-10-143-95 ~]# cat /etc/udev/rules.d/81-mem-hotplug.rules 
SUBSYSTEM=="memory", ACTION=="add", TEST=="state",ATTR{state}=="offline", ATTR{state}="online"
```

- CentOS 6.x
镜像需要添加该规则
- CentOS 7.x
	两个方法, 二选一:
	- 添加该规则
	- 升级systemd的包到systemd-219-19.el7_2.9.x86_64版本
- ubuntu 12.04
镜像需要添加该规则, 目前还有问题该镜像, 正在fix
- ubuntu 14.04
镜像默认包含该规则

### xml 修改
在domain 域下面添加:`<maxMemory slots='16' unit='KiB'>16777216</maxMemory>`  其中slots表示插槽个数:目前设置16个, 16777216表示整个虚拟机最大支持的内存大小, 注意单位是KB.
另外在cpu 域下面添加
```
<numa>
      <cell id='0' cpus='0-31' memory='1048576' unit='KiB'/>
</numa>
```
其中`memory` 大小跟current memory大小一致

最终的xml 省略了无关部分, 类似下面:
```
<domain type='kvm'>
  <maxMemory slots='16' unit='KiB'>16777216</maxMemory>
  ...
  <cpu>
  ...
    <numa>
      <cell id='0' cpus='0-2' memory='1048576' unit='KiB'/>
    </numa>
   ...
  </cpu>
</domain>
```

### 动态的添加memory

- 创建一个内存设备的xml 文件
```
<memory model='dimm'>
  <target>
    <size unit='MiB'>8192</size>
    <node>0</node>
  </target>
</memory>
```
对于添加到不同的内存大小, 对应修改 `size unit` 项, 上例是添加8GB内存的例子


- 如果新加的内存需要保存, 以便下次开机启动, 虚拟机能使用到添加之后的内存大小:
`virsh attach-device hotplug memory-8g.xml --live --config`

- 如果只是给运行中虚拟机添加, 下次启动并不需要使用新添加的内存
`virsh attach-device hotplug memory-8g.xml --live `



### 验证

- 验证qemu 是否添加成功
```
virsh dommemstat <domain id>
```

- 验证guest os是否添加成功
对于guest os 里面直接通过free -m查看, 但是需要注意的是 由于guest os 本身需要占用内存来管理新添加的内存,所以free -m 查看到的结果total字段并不包含kernel reserved的内存. 所以需要 `cat /sys/devices/system/node/node0/meminfo`  





