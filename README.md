# Dell G15 5511 BIOS 解锁指南

适用于 Dell G15 5511（BIOS 1.45.0 及同结构版本）。

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
2. 开机连续按 **F12**，进入启动项菜单（One-Time Boot Menu）。
3. 选择对应的 UEFI U 盘项启动，进入 UEFI Shell。

---

## 三、解锁指令

进入 UEFI Shell 后，按以下顺序执行：

### 1. 切换至 U 盘分区
```efi
fs0:
```
*(若提示找不到或无文件，可依次尝试 `fs1:`、`fs2:`)*

### 2. 执行解锁命令

- **基础解锁（降压 + PCIe 4.0）**
  ```efi
  # 解除 CFG Lock (MSR 0xE2) 与 Overclocking Lock (超频锁)
  setup_var.efi CpuSetup:0x43=0x00 CpuSetup:0xDF=0x00

  # 解锁 M.2 PCIe 4.0 速率 (带 -r 参数，执行后自动重启)
  setup_var.efi -r SaSetup:0x2FF=0x04
  ```

- **可选进阶解锁**
  ```efi
  # 开启超频特性开关并关闭节能睿频（提升睿频平滑度）
  setup_var.efi CpuSetup:0x135=0x01 CpuSetup:0x130=0x00

  # 硬件级 95°C 温度墙偏移与解除 BIOS 写入保护
  setup_var.efi CpuSetup:0x7F=0x05 PchSetup:0x1C=0x00
  ```

### 3. 重启系统
若命令未附带 `-r` 自动重启，可手动输入：
```efi
reset
```

---

## 四、变量地址备忘

| 功能 | 变量位置 | 目标值 | 默认值 | 作用说明 |
| :--- | :--- | :--- | :--- | :--- |
| **CFG Lock** | `CpuSetup:0x43` | `0x00` | `0x01` | 解除 MSR 0xE2 锁定，允许调整核心电压 |
| **Overclocking Lock** | `CpuSetup:0xDF` | `0x00` | `0x01` | 解除电压邮箱锁定，允许运行时降压生效 |
| **PCIe Gen Speed** | `SaSetup:0x2FF` | `0x04` | `0x03` | 启用 PCIe 4.0 速率 (16.0 GT/s) |
| **OverClocking Feature** | `CpuSetup:0x135` | `0x01` | `0x00` | 激活固件超频与调节子模块 |
| **Energy Efficient Turbo**| `CpuSetup:0x130` | `0x00` | `0x01` | 关闭节能睿频，消除升降频延迟 |
| **Tcc Activation Offset** | `CpuSetup:0x7F` | `0x05` | `0x00` | 设定 95°C 硬件温度墙（默认 100°C） |
| **BIOS Lock** | `PchSetup:0x1C` | `0x00` | `0x01` | 解除 PCH 固件写保护 |
