---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
title: 解决家庭环境宽带断线重播后，有概率无法获取 IPV6 地址的问题
excerpt: |
  最近于家庭路由开通 IPV6 后，凌晨运营商断线重新分配 IP 时，有一定概率获取不到 IPV6 地址（IPV4 正常）。
  通过大量的搜索与尝试最终解决该问题，故在这里简单记录一下，后续如有类似问题也可作参考。
author: FriesI23
date: 2024-12-26 09:00:00 +0800
category: network
tags:
  - openwrt
  - ipv6
  - pppoe
---

最近为了家庭智能设备更好的使用，便在路由上开通了 IPV6。不过自从开通后，总是在凌晨运营商断线后重新分配 IP 时，
有一定概率在 `WAN_6` 接口上获取不到 IPV6 地址，但是可以获取到 `IPV6-DP`，也可以正常通过 IPV6 访问外网。

这个很烦人，因为我在路由上有做 DDNS，而 `WAN_6` 地址丢失会导致路由上的 DDNS 脚本无法获取到正确的地址而失败。
因此在晚上搜索了一圈，终于找到了解决方案，这里记录一下，也对一些搜索时可能无用的方案做一下记录。

## 解决方法

一通搜索和尝试后，最终在[【有人遇到过重新拨号时没设置默认路由吗？】][ipv6-solution]这个 Issue 下面找到了问题原因。

主要问题在于 `WAN_6` 接口关闭晚于重新拨号建立的 `pppoe-wan` 接口，这会导致重新重新打开的 `WAN_6` 无法再获取到地址了。
解决的方法也很简单，在重新拨号时判断 `WAN_6` 是否已经关闭，如果没有关闭则等待关闭后再重新拨号。

可以根据[【这里】][ipv6-solution-script]的方案：

```shell
TARGET_FILE="/etc/ppp/ip-down.d/check_wan6_down"

mkdir /etc/ppp/ip-down.d
touch $TARGET_FILE

cat << 'EOF' > "$TARGET_FILE"
#!/bin/sh
while true; do
  if ubus -S list "network.interface.wan_6"; then
    # ubus call network.interface.wan_6 down
    echo "waiting wan_6 down..." >> /path/to/logfile
    sleep 1
  else
    break
  fi
done
EOF

chmod +x "$TARGET_FILE"

# add file to backup configuration
echo $TARGET_FILE >> /etc/sysupgrade.conf
```

### 注：DropBear 相关

IPV6 地址获取可能比 Dropbear 启动慢，如果在以前在 `WAN_6` 上有绑定 SSH 接口，则可能在地址变化后无法访问
（DropBear 监听了错误的接口，必须手动重启服务已改变监听地址。

一个简单的解决方案：将绑定的 `Interface` 设置为 **`unspecified`** 即可，不设置接口 DropBear 便会监听 `::::<your-port>`.

## 一些无效的设置（少走弯路）

包括：

- 在防火墙中添加放行规则：IPV6 IGMP
- 在防火墙中添加放行规则：456 to this device
- 使用 `WAN` 自动创建的 `WAN_6` 而不是 OpenWRT 自带的 `WAN6`

对于第一条，首先 IPV6 使用 MLD 取代了 IGMP，因此只要在防火墙中放行 MLD 即可，放行 IGMP 没有意义也没有道理。

对于第二条，DHCPv6 客户端使用 546/547 端口，开放 456 端口也没有任何道理。

对于最后一条，自动创建的 `WAN_6` 和自己配置的 `WAN6` 其实没有区别，只要正确配置两者的效果是相同的，
不过对于一般用户使用 `WAN_6` 虚拟接口确实更加简单方便，不过最终使用哪个都可行，与上面出现的问题没有关系。

<!-- refs -->

[ipv6-solution]: https://github.com/hanwckf/immortalwrt-mt798x/issues/57
[ipv6-solution-script]: https://github.com/hanwckf/immortalwrt-mt798x/issues/57#issuecomment-1586964992
