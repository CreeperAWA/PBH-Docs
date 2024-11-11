# 进度检查器

进度检查器（ProgressCheatBlocker）（有时也称为 PCB 或者启发式检测算法）是由 PeerBanHelper 创建的基于下载进度的一种启发式的反吸血检测算法。  

## 概述

传统的反吸血通常依靠检查 PeerID 或者 ClientName 来屏蔽。这对于*迅雷*或者*QQ旋风*这种如实报告自己 PeerID 的 Peer 效果不错。但如果恶意吸血者冒充了 qBittorrent 或 Transmission 这种正常客户端，那么传统的反吸血手段就会完全失效。

PeerBanHelper 会持续追踪所有活动种子上的活跃 Peers，实时追踪它们的进度情况。在出现下面的情况时，PeerBanHelper 就会封禁它们：

* Peer 不汇报自己的进度，一直保持为 0%
* Peer 不如实汇报自己的进度，汇报的进度与实际下载了的进度不符
* Peer 断开重连后进度归零，或者进度变少超过一定值（进度回退）
* Peer 在下载了 100% 种子体积大小后，没有断开连接而是仍然持续下载（超量下载）

对于进度检查器来说，其进度判定不是基于单个 IP 地址，而是基于 “IP 组”。同一个组里的 IP 地址都被视为同一个 Peer，不论 IP、端口、PeerID、ClientName 是否相同。  
* 对于 IPv4，这个组的默认大小是 `/32`，也就是 1 个 IPv4 地址
* 对于 IPv6，这个组的默认大小是 `/60`，通常是路由器得到的 IPv6 前缀大小

PBH 持续追踪 Peer 的下载进度，并计算其上传量和上传增量。当 Peer 的统计数据由于某种原因重置了(例如更换端口和PeerID)，PBH 会通过之前记录的数据为 Peer 修正进度以避免进度欺骗。  

## 配置文件

```yaml
  # 进度作弊检查器：Progress Cheat Blocker
  # 注：有时这会错误的封禁部分启用“超级做种”的客户端。但在大多数情况下，此模块能够有效阻止循环下载的流量消耗器，建议启用。
  # Note: Sometimes it may incorrect ban some clients if they enabled "Super Seeding", but in most cases, it can accurately detect the cheat/bad peers.
  progress-cheat-blocker:
    enabled: true
    # Torrent 小于此值不进行检查（单位：字节），对等体可能来不及同步正确的下载进度
    # Skip the check if torrent smaller than this value, unit: bytes, peer may have to no chance to sync the progress
    minimum-size: 50000000
    # 最大差值，单位百分比（1.0 = 100% 0.5=50%）; Max difference, float percentage (1.0=100%, 0.5=50%)
    # PeerBanHelper 根据 BT 客户端记录的向此对等体实际上传的字节数，计算该对等体的最小下载进度
    # PeerBanHelper will use BT client recorded data to check the actual uploaded bytes, and calculate minimal progress that this peer should have
    # 并与对等体汇报给 BT 客户端下载进度进行比较
    # and compare with peer reported progress
    # 如果对等体汇报的总体下载进度远远低于我们上传给此对等体的数据量的比例，我们应考虑客户端正在汇报假进度
    # If peer reported progress is smaller than our calculated progress too much, we will consider it's cheating
    # 默认值为：10%
    # Default allowed percentage is 10%
    # 即：假设我们上传了 50% 的数据量给对方，对方汇报自己的下载进度只有 39%，差值大于 10%，进行封禁
    # It will run like: if we uploaded 50% of data at least to peer, but peer reporting it only have 39%, difference ge 10%, we will ban it
    # 对于自动识别迅雷、QQ旋风的变种非常有效，能够在不更新规则的情况下自动封禁报假进度的吸血客户端
    # It works well on detecting new various and cheat clients.
    maximum-difference: 0.1
    # 进度倒退检测 - Progress rewind detection
    # 默认：最多允许倒退 7% 的进度 - Default: Up to 7% rewind is allowed
    # (考虑到有时文件片段在传输时可能因损坏而未通过校验被丢弃，我们允许客户端出现合理的进度倒退)
    # (Sometimes the pisces may break during transfer, client may drop those pisces, we allow client have rewind in reasonable range)
    # 设置为 -1 以禁用此检测
    # Set to -1 for disabling
    rewind-maximum-difference: 0.07
    # 过量下载：禁止那些在同一个种子的累计下载量超过种子本身大小的客户端
    # Excessive download - Block peers that download even more bytes on a single torrent than the torrent itself
    # 此模块对 Transmission 不起效
    # Not working with Transmission
    block-excessive-clients: true
    # 过量下载计算阈值
    # Calculation threshold
    # 计算方式是： 是否过量下载 = 上传总大小 > (种子总大小 * excessive-threshold)
    # IsExcessive = uploaded > (torrent_size * excessive-threshold)
    excessive-threshold: 1.5
    # IPV4 前缀长度，前缀相同的 IP 都被视为同一个用户
    # IPV4 prefix length, same prefix will trick as a same user
    # 32 = 单个 IP 地址，IPV4 资源宝贵，通常 ISP 不会分配多个 IP 地址
    # 32 = Single IP address, ISP usually only allocate single IPV4 for one user
    ipv4-prefix-length: 32
    # IPV6 前缀长度，前缀相同的 IP 都被视为同一个用户
    # IPV6 prefix length, same prefix will trick as a same user
    # 64 = 常见的 ISP 为单个接入用户分配的前缀长度
    # 64 = The common prefix length that ISP allocate for one user
    ipv6-prefix-length: 60
    ban-duration: 2592000000
    # 启用持久化记录
    # Enable persist recording
    # 启用此功能可能增加磁盘 I/O 并可能影响性能
    # May increase disk I/O and impact the performance
    enable-persist: true
    # 持久化数据存储时长
    # Persist duration
    # 延长此值可缓解针对 PeerBanHelper 的 “缓慢失忆攻击”，但会增加磁盘 I/O 并影响性能
    # Increase this value can alleviate "Slow forgetting attack", This helps stop bad peers from taking advantage of this weakness to reset their data records.
    # 缩短此值可提高性能但吸血者者可能利用这一点进行 “缓慢失忆攻击”
    # Decrease this value may lead to "Slow forgetting attack"
    # 单位：ms 默认值：1209600000 （14 天）
    # Time unit: ms, default: 1209600000 (14 days)
    persist-duration: 1209600000
```

## 缺陷

* 如果对方使用 “超级做种”，那么这可能会导致意外封禁。
* 如果没有关闭下载器的 “允许来自同一 IP 地址的多个连接”，则可能导致统计数据出错，成倍增加快速增长。这个选项默认是关闭的。
