# fnos 环境双 Realtek 交换机识别与通信问题排查

## 背景

Aigo S60（RK3588）板载 **两组 Realtek 交换机芯片**，通过 RGMII 分别接到 RK3588 的两个 GMAC：

| 交换机 | GMAC (CPU 口) | 芯片 | 用户端口 | SMI GPIO |
|--------|---------------|------|----------|----------|
| switch0 | gmac1 (`fe1c0000`, `end1`) | RTL8367S | eth1~eth5 | gpio4 PA6/PA7, reset PC0 |
| switch1 | gmac0 (`fe1b0000`, `end0`) | RTL8367RB-VB | eth6~eth10 | gpio4 PA4/PA5, reset PB7 |

约束条件：**fnos 内核（`6.18.18.c1145-trim #1145`）和它自带的内核模块均不可修改**，唯一能改的是设备树 `rk3588-aigo-s60.dtb`。fnos 使用的是 **upstream `rtl8365mb.ko` / `tag_rtl8_4.ko`** 模块（不是 Rockchip BSP 的私有 `rtl_api` 驱动），这一点是后续排查的关键前提。

## 现象

把仓库编译的 `rk3588-aigo-s60.dtb` 部署到 fnos 后，出现两个问题：

1. **只识别一组交换机**：`ip a` 只看到 `eth1@end1`~`eth5@end1`（switch0），`end0` 下没有任何 `eth6~eth10`（switch1）。`dmesg` 只有 `switch0: found an RTL8367S switch`，完全没有 switch1 的 probe 日志。

2. **在让 switch1 识别后**，`end0` 持续刷报错：
   ```
   net end0: could not find user for port 4
   ```
   eth6~eth10 注册了但无法正常收包（LOWERLAYERDOWN 或虽 UP 但无实际数据）。

## 问题一：switch1 完全不识别

### 排查过程

`dmesg` 显示只有 switch0 被 probe，switch1 毫无线索。但反编译机器上实际加载的 DTB（`/sys/firmware/fdt`）确认 `switch1` 节点确实存在：

```
switch1 {
    compatible = "realtek,rtl8367rb";
    ...
};
```

节点在，却没有 driver 匹配 → 问题出在 `compatible` 与 driver 的 `of_match_table`。

检查 fnos 机器上的模块：

```bash
$ modinfo /usr/lib/modules/6.18.18.c1145-trim/kernel/drivers/net/dsa/realtek/rtl8365mb.ko
alias:          of:N*T*Crealtek,rtl8365mbC*
alias:          of:N*T*Crealtek,rtl8365mb
```

fnos 自带的 upstream `rtl8365mb.ko` 的 `of_match_table` **只有 `realtek,rtl8365mb`**，不包含 `realtek,rtl8367rb`。

对照仓库里（我们修改过的）`drivers/net/dsa/realtek/rtl8365mb_main.c`：

```c
static const struct of_device_id rtl8365mb_of_match[] = {
    { .compatible = "realtek,rtl8365mb",  .data = &rtl8365mb_variant, },
    /* 我们额外添加的，fnos 模块里没有这两行 */
    { .compatible = "realtek,rtl8367rb",     .data = &rtl8365mb_variant, },
    { .compatible = "realtek,rtl8367rb-vb",  .data = &rtl8365mb_variant, },
    { /* sentinel */ },
};
```

仓库源码里为 `rtl8367rb` / `rtl8367rb-vb` 加了匹配项，但 **fnos 的模块是 upstream 原版**，没有这些条目。于是 switch1 的 `compatible = "realtek,rtl8367rb"` 没有任何 driver 认领 → 不 probe。

### 根因

> **switch1 的 `compatible = "realtek,rtl8367rb"` 不在 fnos upstream `rtl8365mb.ko` 的 `of_match_table` 中**，导致 platform driver 不匹配、switch1 节点不被 probe。

### 解决思路

`rtl8365mb` 驱动内部并不依赖 `compatible` 区分芯片型号——它在 `rtl8365mb_detect()` 阶段通过 SMI 读取芯片 ID 寄存器自动识别实际型号。这一点从模块内建的字符串即可验证：

```bash
$ strings rtl8365mb.ko | grep -iE "RTL8367"
RTL8367S
RTL8367RB-VB
```

而 switch0 本身就是用 `realtek,rtl8365mb` 这个 compatible、被检测为 `RTL8367S` 的（`dmesg`: `switch0: found an RTL8367S switch`），证明该驱动对 RTL8367 家族完全兼容。

因此修复非常直接：**把 switch1 的 `compatible` 改成 `realtek,rtl8365mb`**，让 fnos 的 stock 模块能匹配；芯片的真实型号仍由驱动在运行时自动检测。

## 问题二：`could not find user for port N` 刷屏

### 现象

switch1 识别后，`end0`（switch1 的 CPU 口）每收到一个包就报：

```
net end0: could not find user for port 4
```

`port` 值有时是 4，有时是其他值。eth6~eth10 全部 `LOWERLAYERDOWN`，无法通信。

### 排查过程

报错来自 `tag_rtl8_4.c` 的 `rtl8_4_read_tag()`（DSA Realtek RTL8_4 标签解析）。仓库里我们改过的版本：

```c
/* 仓库版本（带多 switch 支持） */
struct dsa_port *cpu_dp = dev->dsa_ptr;
skb->dev = dsa_conduit_find_user(dev, cpu_dp->ds->index, port);
```

但 **fnos 机器上跑的是 upstream 原版 `tag_rtl8_4.ko`**，对应源码（`git show 0db103e91:net/dsa/tag_rtl8_4.c`）：

```c
/* upstream 原版 —— 硬编码 device index = 0 */
skb->dev = dsa_conduit_find_user(dev, 0, port);
if (!skb->dev) {
    dev_warn_ratelimited(&dev->dev,
                         "could not find user for port %d\n", port);
    return -ENOENT;
}
```

`dsa_conduit_find_user(dev, 0, port)` 的语义（`net/dsa/tag.h`）：

```c
static inline struct net_device *dsa_conduit_find_user(struct net_device *dev,
                                                       int device, int port)
{
    struct dsa_port *cpu_dp = dev->dsa_ptr;
    struct dsa_switch_tree *dst = cpu_dp->dst;
    struct dsa_port *dp;

    list_for_each_entry(dp, &dst->ports, list)
        if (dp->ds->index == device && dp->index == port &&
            dp->type == DSA_PORT_TYPE_USER)
            return dp->user;
    return NULL;
}
```

它在 CPU 口所属 DSA tree 的端口列表里找 **`ds->index == device`（这里是 0）且 `dp->index == port` 的 USER 端口**。

而 `ds->index` 由 DTS 的 `dsa,member` 属性决定（`net/dsa/dsa.c`）：

```c
ds->index = m[1];              // dsa,member 第二个值
ds->dst  = dsa_tree_touch(m[0]); // 第一个值是 tree index
```

原 DTS 配置：

| 交换机 | `dsa,member` | tree | `ds->index` |
|--------|--------------|------|-------------|
| switch0 | （无，默认 `{0,0}`） | 0 | 0 |
| switch1 | `<1 1>` | 1 | **1** |

于是 switch1 的 `ds->index = 1`，但 upstream `tag_rtl8_4` 硬编码用 `device=0` 查找 → 永远找不到 `ds->index==1` 的端口 → 每个入向包都报 `could not find user for port N`。

> 为什么 switch0 不报错？因为 switch0 的 `ds->index=0`，正好匹配硬编码的 `device=0`。

### 根因

> **upstream `tag_rtl8_4.ko` 硬编码 `dsa_conduit_find_user(dev, 0, port)`（device index 固定 0）**，而 switch1 原本的 `dsa,member = <1 1>` 使其 `ds->index = 1`，RX 路径无法解析 user port。

### 解决思路

fnos 的 `tag_rtl8_4.ko` 不可改，硬编码的 `device=0` 是死规定。那么唯一的办法是**让 switch1 的 `ds->index` 也等于 0**。

`dsa,member = <tree_index switch_index>`。两个 switch 各自挂在自己的 GMAC 上、互不级联，本就是两个独立的 DSA tree。只要让 switch1 在它自己的 tree 里是**唯一的、index 为 0** 的 switch 即可：

| 交换机 | 修改后 `dsa,member` | tree | `ds->index` |
|--------|---------------------|------|-------------|
| switch0 | （无，默认 `{0,0}`） | 0 | 0 |
| switch1 | `<1 0>` | 1 | **0** |

这样 `dmesg` 仍是 `DSA: tree 0 setup` + `DSA: tree 1 setup`（两个独立 tree，互不干扰），而两个 switch 的 `ds->index` 都是 0 → 硬编码的 `device=0` 对两者都能匹配。

## DTS 修改

两处修改都在 `arch/arm64/boot/dts/rockchip/rk3588-aigo-s60.dts`：

### 修改 1：switch1 的 compatible

```diff
 switch1: switch1 {
-    compatible = "realtek,rtl8367rb";
+    /* fnos 的 upstream rtl8365mb.ko of_match_table 只列出
+     * "realtek,rtl8365mb"，不认 "realtek,rtl8367rb"。改用
+     * rtl8365mb 让 stock 模块能匹配；驱动 rtl8365mb_detect()
+     * 仍会通过芯片 ID 自动识别为 RTL8367RB-VB。 */
+    compatible = "realtek,rtl8365mb";
     mdc-gpios = <&gpio4 RK_PA4 GPIO_ACTIVE_HIGH>;
     mdio-gpios = <&gpio4 RK_PA5 GPIO_ACTIVE_HIGH>;
     reset-gpios = <&gpio4 RK_PB7 GPIO_ACTIVE_LOW>;
```

### 修改 2：switch1 的 dsa,member

```diff
     reset-gpios = <&gpio4 RK_PB7 GPIO_ACTIVE_LOW>;
-    dsa,member = <1 1>;
+    /* 每个 switch 是各自 DSA tree 的唯一成员，ds->index 均为 0。
+     * 这是必须的，因为 fnos 的 upstream tag_rtl8_4.ko 硬编码
+     * dsa_conduit_find_user(dev, 0, port)。若 ds->index=1，
+     * RX 路径无法解析 user port，会刷
+     * "could not find user for port N"。 */
+    dsa,member = <1 0>;
```

### 编译与部署

```bash
# 编译 DTB
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    rockchip/rk3588-aigo-s60.dtb

# 备份并推送
sshpass -p root scp arch/arm64/boot/dts/rockchip/rk3588-aigo-s60.dtb \
    root@<IP>:/boot/dtb/rockchip/rk3588-aigo-s60.dtb

# 重启生效
ssh root@<IP> 'reboot'
```

## 验证结果

重启后 `dmesg` 与 `ip -br a` 确认：

```
[    3.314178] rtl8365mb-smi switch0: found an RTL8367S switch
[    3.517148] DSA: tree 0 setup
[    3.653817] rtl8365mb-smi switch1: found an RTL8367RB-VB switch
[    3.842714] DSA: tree 1 setup
...
[    6.537544] rk_gmac-dwmac fe1b0000.ethernet end0: Link is Up - 1Gbps/Full
```

```
$ ip -br a
end1             UP             fe80::bb73:3d3d:c4ba:53a2/64
end0             UP             fe80::f7a0:34be:d38b:ab84/64
eth1@end1        LOWERLAYERDOWN
eth2@end1        LOWERLAYERDOWN
eth3@end1        LOWERLAYERDOWN
eth4@end1        LOWERLAYERDOWN
eth5@end1        UP             192.168.33.41/24 ...
eth6@end0        LOWERLAYERDOWN
eth7@end0        LOWERLAYERDOWN
eth8@end0        LOWERLAYERDOWN
eth9@end0        LOWERLAYERDOWN
eth10@end0       UP             192.168.33.42/24 ...
```

- ✅ 两组交换机全部识别（switch0: RTL8367S, switch1: RTL8367RB-VB）
- ✅ eth1~eth10 全部注册
- ✅ 接线的端口 Link Up 并拿到 IP（eth5、eth10）
- ✅ `could not find user for port` 报错次数 = **0**（之前每秒都在刷）
- ✅ `DSA: tree 0 setup` + `DSA: tree 1 setup`，两个独立 tree 正常

剩余的 `missing child interrupt-controller node` / `no interrupt support` 是无害提示（DSA 中断为可选项，驱动退化为轮询模式），不影响数据通路。

## 关键经验

1. **适配 stock 内核时，`compatible` 必须落在目标内核 driver 的 `of_match_table` 里**。仓库源码里加的 `of_match` 条目对 fnos 的预编译模块无效——只能让 DTB 去迁就模块，不能反过来。好在 `rtl8365mb` 驱动靠芯片 ID 自动检测，`compatible` 只起"敲门"作用，改了不影响实际行为。

2. **多 switch + 独立 CPU 口的场景下，upstream `tag_rtl8_4` 的硬编码 `device=0` 是个坑**。它假设"一个 DSA tree 里只有一颗 switch"或"包里的 source device 字段恒为 0"。当每个 switch 各走自己的 GMAC（各成独立 tree）时，让每颗 switch 在各自 tree 里都是 index 0，就能绕开这个硬编码——既不需要改内核，也不需要把两颗 switch 塞进同一个 tree（那会要求它们级联，物理上做不到）。

3. **排查这类"节点在、不 probe"问题的通用方法**：反编译 `/sys/firmware/fdt` 确认节点确实进了加载的 DTB → 查 `dmesg` 有无 probe 线索 → 对比目标内核模块的 `modinfo`/`of_match_table` → 看 `compatible` 是否匹配。全过程不需要能在目标机上跑 debug 内核。
