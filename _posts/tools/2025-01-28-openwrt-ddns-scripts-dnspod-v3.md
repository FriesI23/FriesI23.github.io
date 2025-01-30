---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
title: 「王婆卖瓜」在 OpenWRT 中使用 DNSPod 动态解析域名
excerpt: |
  使用 ddns-scripts_tencent_cloud 为 OpenWRT 提供对 DNSPod 的 DDns 支持，
  脚本使用最新的 v3 API，支持 IPV4 与 IPV6 地址，同时提供对 ddns-scripts LUCI 界面中的各种配置项的兼容。
author: FriesI23
date: 2025-01-30 08:00:00 +0800
category: tools
tags:
  - openwrt
  - ddns-scripts
  - dnspod
  - tencent-cloud
  - ddns
---

一直使用 DNSPod 提供的域名服务，前阵子准备将自己服务器的更新脚本移动到主路由上（OpenWRT）。
本着不重复造轮子的原则，在网上搜索了一波，发现了一些可用的 Package。不过看起来多多少少都有点问题，
这里在下面先简要列举一下：

- [`tencentcloud-openwrt-plugin-ddns`][tencentcloud-openwrt-plugin-ddns]: 官方插件，不过似乎很久没有更新了，
  很多 Issue 也没有得到回复，看起来不是很可靠。
- [`ddns-scripts_dnspod`][ddns-scripts_dnspod]: 基于 `ddns-scripts` 的脚本，试了下确实可行，不过有几个问题：
  1. 代码内部实现比较简陋，没有遵守 `ddns-scripts` 中自定义脚本相对标准的写法（参考 `dynamic_dns_updater.sh` 脚本）。
  2. 使用 v2 版本的 API，由于 DNSPod 已被腾讯收购，而整个服务已经整合到腾讯云中，现在最新的 API 为 v3 版本，
     v2 已经标记为**废弃**，因此使用这版 API 存在一定的风险。
- [`qcloud-ddns-docker`][qcloud-ddns-docker]: 使用 `tccli`, 且要部署 Docker，因此不想考虑。

基于上面各种方案的问题，因此决定自己为 `ddns-scripts` 写一个基于 v3 API 的脚本，因此有了如下项目：

## FriesI23/ddns-scripts_tencent_cloud

项目地址：[`ddns-scripts_tencent_cloud`][ddns-scripts_tencent_cloud]

该脚本基于 `ddns-scripts` 项目，并且基于腾讯云 v3 API 进行编写，提供以下支持：

- 同时支持 IPv4 / IPv6
- 支持更新解析值 / 为新域名增加解析 (新增需保证查询主机名为被解析域名).

具体使用方法可以查看 [`README.md`][ddns-scripts_tencent_cloud/readme]，
当然最简单的方法就是在 Releases 界面找到最新的 `*.ipk*` 上传到 OpenWRT 进行安装。

## openwrt/package/ddns-scripts

该项目已同步提交到 [`openwrt/package/ddns-scripts`][ddns-scripts/update_dnspod_cn_v3.sh] 中，
如果你的 `ddns-scripts` 的 Build Number `>=58`，则可以通过在搜索 `ddns-scripts-dnspod-v3` 直接进行安装.

截止 2025/01/30 的相关提交 PR：

- [Add script (#25619)](https://github.com/openwrt/packages/pull/25619)
- [Fix some bugs (#25748)](https://github.com/openwrt/packages/pull/25748)

<!-- refs -->

[tencentcloud-openwrt-plugin-ddns]: https://github.com/Tencent-Cloud-Plugins/tencentcloud-openwrt-plugin-ddns
[ddns-scripts_dnspod]: https://github.com/nixonli/ddns-scripts_dnspod
[qcloud-ddns-docker]: https://github.com/AllanChain/qcloud-ddns-docker
[ddns-scripts_tencent_cloud]: https://github.com/FriesI23/ddns-scripts_tencent_cloud
[ddns-scripts_tencent_cloud/readme]: https://github.com/FriesI23/ddns-scripts_tencent_cloud/blob/master/README.md
[ddns-scripts/update_dnspod_cn_v3.sh]: https://github.com/openwrt/packages/blob/master/net/ddns-scripts/files/usr/lib/ddns/update_dnspod_cn_v3.sh
