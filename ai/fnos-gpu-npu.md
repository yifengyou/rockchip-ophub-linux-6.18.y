# fnos 环境下 RK3588 GPU/NPU 启用（仅改 DTB）

## 背景

Aigo S60（RK3588）SoC 内置：
- **GPU**：Mali-Valhall CSF（`0xfb000000`），RK3588 GPU arch 10.8.6 r0p0
- **NPU**：3 核 RKNN（`0xfdab0000` / `0xfdac0000` / `0xfdad0000`），共 6 TOPS

在 fnos（飞腾 OS，内核 `6.18.18.c1145-trim #1145`）上启用这两块加速器，约束条件与之前修交换机时一致：**fnos 内核和它自带的内核模块均不可修改**，只能改设备树 `rk3588-aigo-s60.dtb`。这一点决定了所有问题都必须在 DTB 层面解决——compatible 必须匹配 fnos 自带模块的 `of_match_table`，节点结构必须匹配驱动 `probe()` 期望的布局。

## 关键前提：fnos 用的是 Rockchip 私有驱动，不是 upstream

fnos 把 Rockchip BSP 私有驱动作为 out-of-tree 模块放在 `/usr/lib/modules/.../updates/trim/` 下，这些驱动期望的 DT 节点结构与 upstream 6.18 dtsi 完全不同。这是整个排查的核心。

| 设备 | upstream 6.18 dtsi | fnos 私有驱动期望 | 官方 BSP DTB（`/boot/dtb/rockchip-bak/bad01`） |
|------|--------------------|-------------------|---------------------------------------------|
| GPU | `compatible = "rockchip,rk3588-mali", "arm,mali-valhall-csf"`；用 upstream `panfrost` | `compatible` 含 `rockchip,rk3588-mali-csf` 或 `arm,mali-valhall`；用 `rkgpu_bifrost_csf.ko` | `compatible = "arm,mali-bifrost"`；`status = "disabled"` |
| NPU | 3 个独立节点 `rockchip,rk3588-rknn-core`（`rknn_core_0/1/2`），每核 1 个 `reg` 区 | **单节点** `rockchip,rk3588-rknpu`，3 个 `reg` 区，3 个中断 | 单节点 `rockchip,rk3588-rknpu`，`status = "okay"` |

### 确认 fnos 驱动 of_match 的方法

```bash
# GPU — rkgpu_bifrost_csf.ko 匹配什么 compatible
$ strings /usr/lib/modules/6.18.18.c1145-trim/updates/trim/rk_gpu/rkgpu_bifrost_csf.ko | grep -iE "rockchip,|arm,mali"
rockchip,rk3588-mali-csf
arm,mali-valhall
...

# NPU — rknpu.ko 匹配什么 compatible
$ strings /usr/lib/modules/6.18.18.c1145-trim/updates/trim/rk_vcodec/rknpu.ko | grep -iE "rockchip,rk.*rknpu"
rockchip,rk3568-rknpu
rockchip,rk3588-rknpu

# NPU 驱动还确认了期望的 reg/interrupt 命名
$ strings .../rknpu.ko | grep -iE "npu._irq|memory resource"
npu0_irq
npu1_irq
npu2_irq
RKNPU: failed to get memory resource for rknpu
```

### 确认 fnos iommu 驱动

```bash
$ grep ROCKCHIP_IOMMU /boot/config-6.18.18.c1145-trim
CONFIG_ROCKCHIP_IOMMU=y          # 内置，非模块

$ strings /boot/vmlinuz-6.18.18.c1145-trim | grep -iE "rockchip,iommu|rockchip,rk.*iommu"
rockchip,iommu
rockchip,rk3588-iommu
rockchip,rk3568-iommu
...
# 注意：没有 "rockchip,iommu-v2" —— 这是后面 NPU iommu 处理的关键
```

## GPU 启用

GPU 相对简单——只需让 `compatible` 命中 `rkgpu_bifrost_csf.ko` 的 of_match，加上供电 regulator，把 `status` 改成 `okay`。

### 改动（`arch/arm64/boot/dts/rockchip/rk3588-aigo-s60.dts`）

```dts
&gpu {
    /* fnos 的 rkgpu_bifrost_csf.ko of_match_table 列的是
     * "rockchip,rk3588-mali-csf" 和 "arm,mali-valhall"，而 upstream
     * dtsi 只给 "arm,mali-valhall-csf"（不在那张表里）。把 vendor
     * string 放第一位确保命中。 */
    compatible = "rockchip,rk3588-mali-csf", "rockchip,rk3588-mali",
                 "arm,mali-valhall-csf";
    mali-supply = <&vdd_gpu_s0>;
    status = "okay";
};
```

`vdd_gpu_s0` 在 `&rk806` PMIC 节点里已有定义（`dcdc-reg1`），无需新增。upstream 的 `gpu_opp_table`（在 `rk3588-opp.dtsi`）已是简单 `operating-points-v2` 格式，fnos 的 `rockchip_opp_select` 模块能正常处理。

### 验证

```
[    2.593425] rkgpu_bifrost_csf fb000000.gpu: Kernel DDK version g18p0-01eac0
[    2.600299] rkgpu_bifrost_csf fb000000.gpu: GPU identified as 0x7 arch 10.8.6 r0p0 status 0
[    2.605713] rkgpu_bifrost_csf fb000000.gpu: Probed as mali0

$ ls /sys/class/misc/ | grep mali
mali0

$ cat /proc/iomem | grep fb000000
fb000000-fb1fffff : fb000000.gpu
```

`Failed to get leakage` 和 `r0p0 status 5 not found in HW issues table; falling back to closest match: r0p0 status 0` 是无害的——前者是因为没接 BSP 的 nvmem leakage cell，后者是硅版本小差异，驱动有 fallback。`debugfs: 'fb000000.gpu-mali' already exists in 'vdd_gpu_s0'` 也是无害的重复注册警告。

## NPU 启用

NPU 比较复杂，因为 upstream 6.18 dtsi 的 NPU 节点结构（3 个独立 `rknn-core` 节点）与 fnos `rknpu.ko` 期望的（单节点 3 核合一）完全不同。中途踩了三个坑，下面按踩坑顺序记录。

### 改动总览

```dts
/* 删除 upstream 的另外两个核节点，让它们的 reg 区域释放出来 */
/delete-node/ &rknn_core_1;
/delete-node/ &rknn_core_2;

/* 三个 mmu 节点保留（为了 phandle 引用合法）但 disable，
 * 让 rockchip-iommu 驱动不 probe（见下文"坑3"） */
&rknn_mmu_0 { status = "disabled"; };
&rknn_mmu_1 { status = "disabled"; };
&rknn_mmu_2 { status = "disabled"; };

/* 把 rknn_core_0（已经在 0xfdab0000）改造成 BSP 风格的单节点 */
&rknn_core_0 {
    compatible = "rockchip,rk3588-rknpu";
    reg = <0x0 0xfdab0000 0x0 0x10000>,
          <0x0 0xfdac0000 0x0 0x10000>,
          <0x0 0xfdad0000 0x0 0x10000>;
    reg-names = "pc", "cna", "core";
    interrupts = <GIC_SPI 110 IRQ_TYPE_LEVEL_HIGH 0>,
                 <GIC_SPI 111 IRQ_TYPE_LEVEL_HIGH 0>,
                 <GIC_SPI 112 IRQ_TYPE_LEVEL_HIGH 0>;
    interrupt-names = "npu0_irq", "npu1_irq", "npu2_irq";
    /* ... clocks/resets/power-domains 略 ... */
    iommus = <&rknn_mmu_0>;      /* 指向 disabled 的 mmu */
    rknpu-supply = <&vdd_npu_s0>;
    status = "okay";
};

&{/} {
    npu_opp_table: opp-table-npu { /* 8 个 opp 条目 */ };
};
```

### 坑1：在 `&{/}` 里新建 `npu@fdab0000` 导致 DTC 节点合并

第一次尝试是在 `&{/}` 块里新建一个 `rknpu: npu@fdab0000` 节点，同时把 upstream 的 3 个 `rknn_core_*` 都 `status = "disabled"`。结果反编译 DTB 发现：upstream 的 `rknn_core_0` 也是 `npu@fdab0000`（相同 unit-address），DTC 把两个节点**合并**了，`compatible` 变成了一堆混乱的字符串，新节点的属性被塞进旧节点里。

**教训**：DTC 对相同 unit-address 的节点会合并，不管是不是在不同源文件里定义的。要复用一个已有节点的位置，正确做法是直接 `&label` override 它，而不是在 `&{/}` 里按地址新建。

### 坑2：`status = "disabled"` 不释放 `reg` 区域 → `-EBUSY`

第二个版本改用 `&rknn_core_0` override + `status = "disabled"` 关掉 `rknn_core_1/2`。但 `rknpu.ko` probe 时仍报：

```
RKNPU fdab0000.npu: error -EBUSY: can't request region for resource [mem 0xfdab0000-0xfdabffff]
RKNPU fdab0000.npu: error -EBUSY: can't request region for resource [mem 0xfdac0000-0xfdacffff]
RKNPU fdab0000.npu: error -EBUSY: can't request region for resource [mem 0xfdad0000-0xfdadffff]
```

原因：platform bus 对 `status = "disabled"` 的节点**不绑定驱动**，但**仍然在 `iomem` 里保留它的 `reg` 区域**（因为节点本身还是 `of_platform_populate` 的一部分）。新 NPU 节点请求 `0xfdac0000-0xfdacffff` 时，被 disabled 的 `rknn_core_1` 节点已经占着这个区域。

**解决**：用 `/delete-node/` 彻底删除 `rknn_core_1` 和 `rknn_core_2`，而不只是 disable。`/delete-node/ &label;` 会让节点从 DTB 里完全消失，`reg` 区域也释放了。

### 坑3：upstream iommu 驱动 probe 后与 NPU 抢 `reg` 区域

第三个版本：`/delete-node/` 删 `rknn_core_1/2`，同时在 `&{/}` 里新建一个统一 iommu 节点（`rockchip,rk3588-iommu`，覆盖 4 个 reg 区 `fdab9000/fdaba000/fdaca000/fdada000`）。NPU 的 `iommus` 指向它。结果：

```
[    0.862278] platform fdab0000.npu: Adding to iommu group 0
[    2.067404] RKNPU fdab0000.npu: RKNPU: rknpu iommu is enabled, using iommu mode
[    2.068711] RKNPU fdab0000.npu: error -EBUSY: can't request region for resource [mem 0xfdab0000-0xfdabffff]
```

`/proc/iomem` 显示 iommu probe 成功并占了 4 个 0x100 区域：
```
fdab9000-fdab90ff : fdab9000.iommu iommu@fdab9000
fdaba000-fdaba0ff : fdab9000.iommu iommu@fdab9000
fdaca000-fdaca0ff : fdab9000.iommu iommu@fdab9000
fdada000-fdada0ff : fdab9000.iommu iommu@fdab9000
```

`-EBUSY` 的来源是 NPU 的 reg 区 `0xfdab0000 + 0x10000 = 0xfdab0000-0xfdabffff`（64KB），覆盖了 iommu 的 `0xfdab9000`。upstream `rockchip-iommu` 驱动 probe 时 `request_mem_region(0xfdab9000, 0x100)` 成功，之后 NPU `request_mem_region(0xfdab0000, 0x10000)` 就 EBUSY 了。

**关键观察**：官方 BSP（`bad01`）的 NPU iommu 用的是 `compatible = "rockchip,iommu-v2"`，但 fnos 内置的 `rockchip-iommu` 驱动**不匹配** `rockchip,iommu-v2`（只匹配 `rockchip,rk3588-iommu` / `rockchip,rk3568-iommu`）。所以在官方 BSP DTB 上，NPU iommu 节点存在但**驱动从不 probe**，`request_mem_region` 从不发生，NPU 一直跑在 **non-iommu 模式**。

**解决**：不要试图让 NPU iommu 工作。保持 upstream `rknn_mmu_0/1/2` 节点存在（为了 `iommus` phandle 引用合法）但 `status = "disabled"`，这样 iommu 驱动不 probe、不抢 region，`rknpu.ko` 自动落到 non-iommu 模式：

```
[    2.063722] RKNPU fdab0000.npu: RKNPU: rknpu iommu is disabled, using non-iommu mode
[    2.065411] [drm] Initialized rknpu 0.9.8 for fdab0000.npu on minor 0
```

non-iommu 模式对 NPU 推理功能没有影响——它只是意味着 NPU 用 `dma_map_single` 而不是 IOMMU 映射，连续内存分配。`rknpu.ko` 里有明确的 fallback 路径（`"rknpu iommu device-tree entry not found!, using non-iommu mode"` / `"rknpu iommu is disabled, using non-iommu mode"`）。

### 坑4：opp-table `opp-microvolt` 与 supply 数量不匹配

第二个版本里 NPU 节点同时设了 `rknpu-supply` 和 `mem-supply`（两个供电），但 opp-table 每个 opp 只给了 3 个值（1 组 min/target/max）。`rockchip_opp_select` 模块报：

```
Invalid number of elements in microvolt property (3) with supplies (2)
```

它期望每个 opp 给 2 组（6 个值），因为节点有 2 个 supply。最简单的修法：去掉 `mem-supply`，只保留 `rknpu-supply`（一个 supply，对应 3 个值的 opp-microvolt）。

### 最终验证

```
[    2.063722] RKNPU fdab0000.npu: RKNPU: rknpu iommu is disabled, using non-iommu mode
[    2.065411] [drm] Initialized rknpu 0.9.8 for fdab0000.npu on minor 0
[    2.069529] RKNPU fdab0000.npu: RKNPU: bin=0

$ cat /sys/devices/platform/fdab0000.npu/version
RKNPU driver: v0.9.8

$ ls /dev/dri/by-path/
platform-fdab0000.npu-card
platform-fdab0000.npu-render

$ cat /proc/iomem | grep -iE "npu|fdab|fdac|fdad"
fdab0000-fdabffff : fdab0000.npu pc
fdac0000-fdacffff : fdab0000.npu cna
fdad0000-fdadffff : fdab0000.npu core
```

三核的 reg 区域都成功映射，NPU 作为 DRM render 节点（`/dev/dri/renderD128` 旁的 NPU render）可用。

## 修改的文件

只改了一个文件：

- `arch/arm64/boot/dts/rockchip/rk3588-aigo-s60.dts`
  - `&gpu` override（~line 529）：compatible 加 `rockchip,rk3588-mali-csf`，加 `mali-supply`，`status = "okay"`
  - NPU 段（~line 920-1040）：`/delete-node/` 两个多余的 core 节点，disable 三个 mmu 节点，`&rknn_core_0` override 成 BSP 风格单节点，`&{/}` 里加 `npu_opp_table`

内核源码和驱动模块**均未改动**。

## 部署

```bash
# 编译
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rockchip/rk3588-aigo-s60.dtb -j$(nproc)

# 推送（先备份）
sshpass -p root ssh root@192.168.33.41 \
  'cp /boot/dtb/rockchip/rk3588-aigo-s60.dtb /boot/dtb/rockchip/rk3588-aigo-s60.dtb.bak-gpu-npu'
sshpass -p root scp arch/arm64/boot/dts/rockchip/rk3588-aigo-s60.dtb \
  root@192.168.33.41:/boot/dtb/rockchip/rk3588-aigo-s60.dtb

# 重启
sshpass -p root ssh root@192.168.33.41 'reboot'
```

## 备注

- `Failed to get leakage`（GPU 和 NPU 都有）是无害的——BSP 用 nvmem cell 读芯片漏电值做 OPP 微调，我们不接 nvmem cell，驱动用默认 OPP，不影响功能。
- NPU 跑在 non-iommu 模式是**有意为之**，与官方 BSP 在 fnos 上的实际行为一致。不要试图"修复"成 iommu 模式——upstream `rockchip-iommu` 驱动的 `request_mem_region` 会与 NPU 的 64KB reg 窗口冲突。
- 机器有偶发的 GICv3 LPI crash（`CPUx: Failed to disable LPIs`，约每几次重启触发一次，离线 ~8 分钟后恢复），与 GPU/NPU 改动无关，是 TF-A 的 LPI prop table 没在 DT `reserved-memory` 里预留导致的。部署 DTB 时如果机器长时间不回来，多半是这个，等它自己恢复即可。
