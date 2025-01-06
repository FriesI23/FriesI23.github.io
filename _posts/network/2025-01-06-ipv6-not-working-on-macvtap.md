---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
title: 记录 KVM 在使用 macvtap 提供网络后，IPV6 地址无法 Ping 通的问题
excerpt: >-
  在使用 KVM (libvirt) 配置 macvtap 网络时，由于默认多播通信的限制，可能导致客户端中的 IPv6 无法正常工作。
  问题旨在记录并解决该问题，以恢复虚拟机中 IPv6 的正常功能。
author: FriesI23
date: 2025-01-06 09:00:00 +0800
category: network
tags:
  - ipv6
  - libvirt
  - macvtap
  - multicast
---

最近准备将家里面的 `Home Assistant` IPV6 GUA 地址配置到域名，设置时候突然发现 HA 的 IPV6 地址无法除了本机外都无法 Ping 通。
最后发现竟然是 KVM 中 macvtap 的默认配置导致的问题，故在这里简单记录一下解决方法。

下面的方案也适用于所有使用 KVM 部署虚拟机并由 macvtap 提供 host 网络接口的情况。

## IPV6 中的邻居发现（ND)

IPV6 与 IPV4 的 ARP 不同，`ND` 由组播（multicast) 而不是广播（broadcast）。
这里的技术细节就不在本文赘述了，我们只需要知道 IPV6 与 IPV4 路由发现的机制并不相同。

## 排查过程

发现 HA 中的 IPV6 地址无效后，首先怀疑是网络适配器配置错误，但是检查 Interface 相关配置后并没有发现异常。
因此对同一宿主机上的其他不同主机也进行的检查，发现：

- KVM 下使用 host 的客户端都存在 IPV6 无法 ping 通，甚至无法访问的情况
  （由于网络栈都是用了双栈协议，在不特意需求 IPV6 的前提下，失效并不会对网络访问产生影响，故一直没有发现）。
- 其他使用 Docker 或者 LXC 托管的服务都正常。

将问题缩小后 KVM Guest 后，由于 IPV6 地址创建正常（个人网络使用 SLAAC 由 RA ISP 提供 PD 委托），因此怀疑是路由表有问题。
使用 `ip -6 route show` 查看后，果然对应的 Interface 上没有对应路由条目，因此可以初步判断问题出在邻居发现（ND）上。

尝试在 Guest 中进行抓包，发现所有虚拟机内都没有收到任何 NS 报文。

## 解决问题（修复 Multicast）

最终在 ["How to configure macvtap to let it pass multicast packet correctly?"](https://superuser.com/a/1033768) 找到了解决方案。
下面引用一段原文解释上述问题为什么会产生：

> libvirt defaults to trustGuestRxFilters=no. This is documented to ignore the guest "interface mac address and receive filters".
> This is regrettably obscure; the commit message is slightly clearer: "interface's mac address and unicast/multicast filters".
>
> -- ["IPv4/IPv6 multicasts are not forwarded via macvtap/macvlan bridge to VM (between VMs)"][redhat-bug-1035253-s1]

我们可以从上面看到，`libvirt` 默认会对 `macvtap/macvlan` 的多播进行过滤，这同样会过滤掉 IPV6 中邻居发现相关的报文，而 IPV6 本身依赖多播。
因此限制多播便会导致 IPV6 功能不完整。

解决方法也很简单，在虚拟机对应的 macvtap 接口上启用多播。下面会以一个虚拟机的 XML 片段进行简单举例：

```xml
<!-- Add `trustGuestRxFilters="yes"` on Interfaces you want enable multicast-->
<domain type='kvm' id='7'>
  <name>vm</name>
  <uuid>xxxx-xxxx-xxx</uuid>
  <!-- ... -->
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <!-- ... -->
    <interface type='network' trustGuestRxFilters='yes'>
      <mac address='6a:ca:1a:ff:3d:65'/>
      <source network='host'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </interface>
    <!-- ... -->
  </devices>
  <!-- ... -->
</domain>
```

关闭虚拟机后，XML 添加对应属性，然后重启虚拟机，检查路由表正常，且能正常收到 `NS` 等报文，也能 Ping 通地址，问题解决。

## 参考资料

- ["Neighbor Discovery Protocol"](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol)
- ["How to configure macvtap to let it pass multicast packet correctly?"](https://superuser.com/q/944678)
- ["IPv4/IPv6 multicasts are not forwarded via macvtap/macvlan bridge to VM (between VMs)"](https://bugzilla.redhat.com/show_bug.cgi?id=1035253)

<!-- refs -->

[redhat-bug-1035253-s1]: https://bugzilla.redhat.com/show_bug.cgi?id=1035253#c15
