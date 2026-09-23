# iStoreOS for Orange Pi R1 Plus（官方 ImageBuilder 打包）

给 **Xunlong Orange Pi R1 Plus（RK3328 / 1GB / 双千兆 / SD 卡启动）** 生成 **原生 iStoreOS** 固件。

不编译源码 —— 用的是 iStoreOS 官方发布的 ImageBuilder，`make image` 打包，几分钟出结果。

---

## 已核实的事实（2026-09-23 实拉数据）

这些不是推测，是从官方服务器实际下载/解析出来的：

| 项 | 值 | 来源 |
|---|---|---|
| ImageBuilder | `istoreos-imagebuilder-rockchip-armv8.Linux-x86_64.tar.zst` 401.8 MiB | `fw.koolcenter.com/iStoreOS/ib/rk3xxx/`，**2026-09-11 15:10** |
| sha256 | `de18ed4320d05ad873fdccaab9220f08e72f2d9cc5637191f41ffea1d039cf0b` | 官方 `sha256sums`（工作流会自动校验） |
| 内核 | **6.12.94** | IB 包索引（`kernel - 6.12.94~5fab3a97`） |
| 包管理器 | **apk** 3.0.5（**没有 opkg**） | 索引里有 `apk-mbedtls`，无 `opkg` |
| profile | `xunlong_orangepi-r1-plus` **启用状态** | `istoreos/istoreos` 仓库 `istoreos-24.10` 分支 `target/linux/rockchip/image/armv8.mk` 末尾 `TARGET_DEVICES += xunlong_orangepi-r1-plus` |
| 设备依赖包 | `kmod-usb-net-rtl8152 - 6.12.94-r1` **在索引里** | IB 包索引 |

一句话：**设备支持一直在，官方只是没给它发成品固件。**

### 与你现在的 ImmortalWrt 24.10.5 对比

| | 现在 | iStoreOS 这个 IB |
|---|---|---|
| 内核 | 6.6.122 | **6.12.94** |
| 商店 `luci-app-store` | 0.2.1-r1 | 0.2.1-r1（相同） |
| 首页 `luci-app-quickstart` | 0.12.4-r1 | **0.12.10-r1**（更新） |
| 磁盘管理 | — | `luci-app-diskman` 0.2.18_beta |
| Dockerman | 有 | 有（`luci-app-dockerman`） |
| Docker 引擎 | — | dockerd **27.3.1-r5**、docker-compose 5.1.4 |
| 包管理器 | opkg | **apk** |

**这里有个反直觉的点**：我上一轮跟你说"iStoreOS 拿到的商店版本更低"，那是基于另一个第三方源的说法。按官方 IB 的实拉索引，商店版本和你现在一样，首页反而更新，内核领先一个大版本。**这条我核实后要收回。**

### ⚠️ 唯一需要你接受的代价

官方 IB 索引里**没有** `attendedsysupgrade` / `auc` / `owut` 任何一个，官方下载站也没有 R1 Plus 的机型目录 —— 所以：

> **这台机器上没有在线一键升级。** 将来要升级，就是回来重跑一次 Actions，然后手动 `sysupgrade` 刷入。

工作流顶部预留了每月自动重建的 cron（默认注释掉），能自动帮你把新 IB 打成新固件，但**最后那一步刷机得你手动来**。这是我建议你保留选项的原因，但你既然选了 iStoreOS，咱就按这条路走。

---

## 用法

**详细到不用动脑的步骤在 👉 [`操作手册.md`](./操作手册.md)**，从建仓库、填参数、看日志、写卡到刷后配置，一步步照着点就行。

这里只留参数速查（更完整的说明见手册第 2 步）：

| 参数 | 默认 | 说明 |
|---|---|---|
| `profile` | `xunlong_orangepi-r1-plus` | LTS 版硬件（网卡 yt8531c）要选 `-lts` |
| `lan_ip` | `10.0.0.1` | **别用默认的 192.168.100.1**，否则刷完网段变了你就进不去后台 |
| `rootfs_partsize` | `4096` | rootfs 分区 MB。**换 64G 卡后建议 4096**，剩下约 58G 另建数据分区，做法见 `操作手册.md` 第 9.5 步 |
| `include_docker` | false | **1GB 内存建议先关**。你这台只有 978MB，还要跑完整网关栈 |
| `with_gui_extras` | true | 商店 + 首页 + 磁盘管理 + argon 主题 |
| `extra_packages` | 空 | 追加包，空格分隔。**包名写错构建会失败** |
| `root_password` | 空 | 建议填。在 runner 内加盐算 sha512，明文不会进日志 |

跑完 5–15 分钟，产物在 Actions 页面底部 **Artifacts**。

**想省事就两个都别管**：默认参数 Run 一次，Artifacts 里下 `*-sysupgrade.img.gz`。

---

## 关于定制的回答

**那些只是换了个壳的应用（商店 / 首页 / 磁盘管理 / 主题）→ 不需要编译，不需要 iStoreOS 源码**，官方 IB 的包索引里全都有，`with_gui_extras` 勾上就进固件了。

真正需要碰源码的只有：自己写 C/ LuCI 的插件、改内核、加不在仓库里的第三方包。你目前列的都在索引里。

---

## 刷入

**先备份。会清掉全部现有配置。**

```sh
sysupgrade -b /tmp/config-backup.tar.gz   # 在路由器上生成，然后 SCP 拉到本地
```

需要重做（刷完记得补回来）的东西：`zram 512MB + swappiness`、oom-guard 脚本与 cron、Docker 容器数据、PPPoE 宽带账号密码。

### 方式 A：sysupgrade（推荐，不用拔卡）

```sh
sysupgrade -F -n istoreos-*-sysupgrade.img.gz
```

`-F` 强制跨发行版（ImmortalWrt → iStoreOS 必加），`-n` 不保留旧配置（跨发行版升级保留配置几乎必出问题）。

```sh
sysupgrade -F /tmp/xxx.img.gz   # 保留配置
```

### 方式 B：写 SD 卡

产物里若有 `*-sdcard.img.gz`，用 Etcher / Rufus 直接写卡。
**R1 Plus 不能从 SPI flash 启动，必须 SD 卡。**

---

## 刷完之后

- 默认 `root`，密码看你有没有填 `root_password`（没填就是空密码，首次登录会被要求设置）
- **第一件事**：检查 WAN 入站（网络 → 防火墙 → WAN → 入站 → 改为 **拒绝**）
- 装软件用 **`apk add <包名>`**，不是 `opkg`（这套 iStoreOS 是 apk 的，索引里没有 opkg）
- 之前手动做的 zram / oom-guard / swappiness 需要重做
- 剩余 SD 空间：先用自带的 **磁盘管理**（diskman）看分区情况再决定怎么处理

### 关于 rootfs 扩容（这条我没在这台机器上验证过）

`ROOTFS_PARTSIZE` 对 rockchip 这种 target **不一定生效** —— 工作流同时传了 `ROOTFS_PARTSIZE` 和 `CONFIG_TARGET_ROOTFS_PARTSIZE`，哪个生效取决于 target 的 image.mk，多余的 make 变量会被忽略，不会报错。

如果刷完发现 rootfs 还是很小：
- 看 Artifacts 里镜像文件名和实际大小
- 用 iStoreOS 自带的 **磁盘管理** 看分区情况，或用命令行 `blkid` / `mount` / `df -h` 确认当前布局再决定怎么扩
- 最稳的办法其实是**构建时就把 `rootfs_partsize` 填大**（你 SD 卡 8G，填 2048 完全够），省掉事后扩容

---

## 说明

- 本仓库只调用官方 ImageBuilder，不含任何第三方代码修改
- 出的是**原生 iStoreOS**（来自 `fw.koolcenter.com` 官方 IB），不是魔改版
- **这套 workflow 我一次都没实跑过**（本机沙箱禁了 WSL/Docker，跑不了 Linux IB）。第一次可能在依赖或包名上失败，看失败步骤的日志改 `extra_packages` 即可
- 工作流里加了 profile 预检：如果 IB 里不存在该 profile，会直接失败并打印可用列表兜底
