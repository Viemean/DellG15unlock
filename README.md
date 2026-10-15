# Dell G15 BIOS 解锁指南

适用于 Dell G15 5511和5510。

---

## 目录结构说明

- `EFI/`：UEFI 引导项，直接拷入 U 盘即可启动
- `biosTools/`：通用工具链（`uefitool`、`ifrextractor`、`setup_var.efi`、`shellx64.efi`）
- `firmwares/`：按设备型号与 BIOS 版本归档的 Setup 模块及 IFR 表单文本
- `BIOSUtilities/`：Dell PFS 固件解包依赖脚本

---

## 一、U 盘准备

1. 准备一个 **FAT32** 格式的 U 盘。
2. 将本项目中的以下文件及文件夹复制到 U 盘根目录：

```text
U盘根目录/
├── EFI/
│   └── BOOT/
│       └── BOOTX64.EFI       <-- 来自项目 EFI/BOOT/BOOTX64.EFI
└── setup_var.efi             <-- 来自项目 biosTools/setup_var.efi
```

---

## 二、启动步骤

1. 将 U 盘插入电脑。
2. 开机连续按 **F12**，进入启动项菜单。
3. 选择对应的 UEFI U 盘项启动，进入 UEFI Shell。

---

## 三、解锁指令

进入 UEFI Shell 后，先切换至 U 盘分区：
```efi
fs0:
```
*(若提示找不到或无文件，可依次尝试 `fs1:`、`fs2:`)*

---

### A. Dell G15 5511 (11代酷睿，指南版本BIOS 1.45.0)

- **基础解锁（降压 + PCIe 4.0）**
  ```efi
  # 解除 CFG Lock (MSR 0xE2) 与电压写入锁定 (Overclocking Lock，允许降压生效)
  setup_var.efi CpuSetup:0x43=0x00 CpuSetup:0xDF=0x00

  # 解锁主硬盘位 M.2 PCIe 4.0 速率 (带 -r 参数，执行后会自动重启，不带就不会)
  setup_var.efi -r SaSetup:0x2FF=0x04
  ```

- **进阶调优（可选）**
  ```efi
  # 写入 95°C 温度墙（默认 100°C）
  setup_var.efi CpuSetup:0x7F=0x05

  # （可选）开启高级超频特性（仅在使用 Intel XTU 或需要手动调整单核倍频时开启）
  setup_var.efi CpuSetup:0x135=0x01
  ```

---

### B. Dell G15 5510 (10代酷睿，BIOS 1.43.0)

> 注：10 代 Comet Lake 硬件原生为 PCIe 3.0，无需且不支持解锁 PCIe 4.0。

- **基础解锁（降压）**
  ```efi
  # 解除 CFG Lock (MSR 0xE2) 与电压写入锁定 (Overclocking Lock，允许降压生效)
  setup_var.efi CpuSetup:0x3E=0x00 CpuSetup:0xDA=0x00
  ```

- **进阶调优（可选）**
  ```efi
  # 开启 Above 4G 解码（激活 Resizable BAR）与写入 95°C 温度墙
  setup_var.efi SaSetup:0xFA=0x01 CpuSetup:0x7A=0x05

  # （可选）开启高级超频特性（仅在使用 Intel XTU 或需要手动调整单核倍频时开启）
  setup_var.efi CpuSetup:0x1B7=0x01
  ```

- **重启系统**
  ```efi
  reset
  ```

---

## 四、Above 4G 解码与 Resizable BAR

- **作用机制**：传统模式下 PCIe 外设显存被限制在 32 位（4GB 边界以下）物理地址空间，导致单次访问窗口被锁死在 256 MB。开启 **Above 4G Decoding** 后，主板允许将显存映射至 4GB 以上的 64 位连续高位地址段。
- **与 Resizable BAR 的关系**：Above 4G 解码是开启 Resizable BAR（显存全量寻址）的硬性前置条件。开启后，CPU 能够直接、并行地向整块显存（如 6GB/8GB）调度数据，减少游戏大纹理加载时的换页延迟，改善 1% Low 掉帧。
- **机型差异**：
  - **G15 5511 (11代)**：出厂固件已默认开启，无需手动干预。
  - **G15 5510 (10代)**：出厂固件默认将 `SaSetup:0xFA` 设为 `0x00`（关闭）。需通过 `SaSetup:0xFA=0x01` 手动激活以使显卡 Resizable BAR 正常握手。
- **如何查看系统是否已开启**：
  - **Linux 环境**：终端执行 `nvidia-smi -q -d MEMORY | grep -i -A 4 "BAR1"`。若 `Total` 大于 256 MiB（例如为 8192 MiB），表示已经生效，**无需解锁**；若仅为 256 MiB 则表示未开启。
  - **Windows 环境**：打开 **NVIDIA 控制面板** -> 左下角 **系统信息** -> 检查列表中 `Resizable BAR` 是否显示为 **“是”**（或通过 GPU-Z 首页右下角查看 `Resizable BAR` 是否为 **Enabled**）。若显示为“是/Enabled”，表示已经生效，**无需解锁**。

---

## 五、setup_var.efi 命令行语法与查询示例

在 UEFI Shell（`fs0:` 分区）中，可通过以下命令格式随时查询当前状态或修改数值：

### 1. 仅查看当前 BIOS 数值（不修改）
命令格式：`setup_var.efi <VarStore>:<Offset>`

- **查看 5511 当前降压锁状态**：
  ```efi
  setup_var.efi CpuSetup:0x43 CpuSetup:0xDF
  ```
- **查看 5510 当前降压锁状态**：
  ```efi
  setup_var.efi CpuSetup:0x3E CpuSetup:0xDA
  ```
- **屏幕输出解读**：
  执行后终端会显示 `current value is 0x...`：
  - 若为 `0x01`：表示处于**出厂默认锁定**状态；
  - 若为 `0x00`：表示**已经解锁成功**。

### 2. 修改写入数值
命令格式：`setup_var.efi <VarStore>:<Offset>=<目标值>`

- **单项修改**（如 5511 写入 95°C 温度墙）：
  ```efi
  setup_var.efi CpuSetup:0x7F=0x05
  ```
- **多项连续写入**（支持空格分隔多个项同时修改）：
  ```efi
  setup_var.efi CpuSetup:0x43=0x00 CpuSetup:0xDF=0x00
  ```
- **写入后自动重启**（增加 `-r` 参数）：
  ```efi
  setup_var.efi -r SaSetup:0x2FF=0x04
  ```
  *(注：加 `-r` 参数在写入成功后会立刻自动重启；若不带 `-r` 则停留在当前 Shell，可继续使用第 1 点命令验证新值，确认无误后输入 `reset` 重启)*

---

## 六、变量地址备忘表

### A. Dell G15 5511 (11代酷睿，BIOS 1.45.0)

| 功能 | 变量位置 | 目标值 | 默认值 | 作用说明 |
| :--- | :--- | :--- | :--- | :--- |
| **CFG Lock** | `CpuSetup:0x43` | `0x00` | `0x01` | 解除 MSR 0xE2 锁定，允许调整核心电压 |
| **Overclocking Lock** | `CpuSetup:0xDF` | `0x00` | `0x01` | 解除电压锁定，允许降压生效 |
| **PCIe Gen Speed** | `SaSetup:0x2FF` | `0x04` | `0x00 (Auto)` | 启用 PCIe 4.0 速率 (16.0 GT/s) |
| **OverClocking Feature** | `CpuSetup:0x135` | `0x01` | `0x00` | 激活固件高级超频与调节子模块（XTU / 调倍频） |
| **Energy Efficient Turbo**| `CpuSetup:0x130` | `0x00` | `0x01` | 关闭节能睿频，消除升降频延迟 |
| **Tcc Activation Offset** | `CpuSetup:0x7F` | `0x05` | `0x00` | 设定 95°C 硬件温度墙（默认 100°C） |
| **BIOS Lock *(可选)*** | `PchSetup:0x1C` | `0x00` | `0x01` | 解除 PCH 固件写保护（仅软刷改版 BIOS 需解锁，常规请保持开启） |

---

### B. Dell G15 5510 (10代酷睿，BIOS 1.43.0)

| 功能 | 变量位置 | 目标值 | 默认值 | 作用说明 |
| :--- | :--- | :--- | :--- | :--- |
| **CFG Lock** | `CpuSetup:0x3E` | `0x00` | `0x01` | 解除 MSR 0xE2 锁定，允许调整核心电压 |
| **Overclocking Lock** | `CpuSetup:0xDA` | `0x00` | `0x01` | 解除电压锁定，允许降压生效 |
| **OverClocking Feature** | `CpuSetup:0x1B7` | `0x01` | `0x00` | 激活固件高级超频与调节子模块（XTU / 调倍频） |
| **Energy Efficient Turbo**| `CpuSetup:0x1AF` | `0x00` | `0x02 (Auto)` | 关闭节能睿频，消除升降频延迟 |
| **Tcc Activation Offset** | `CpuSetup:0x7A` | `0x05` | `0x00` | 设定 95°C 硬件温度墙（默认 100°C） |
| **Above 4G Decoding** | `SaSetup:0xFA` | `0x01` | `0x00` | 开启 4GB 以上内存映射（激活 Resizable BAR 的前提） |
| **BIOS Lock *(可选)*** | `PchSetup:0x17` | `0x00` | `0x01` | 解除 PCH 固件写保护（仅软刷改版 BIOS 需解锁，常规请保持开启） |

