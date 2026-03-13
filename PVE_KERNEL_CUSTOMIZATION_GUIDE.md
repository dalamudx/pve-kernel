# Proxmox VE 内核定制指南：修改 QS/ES CPU 型号显示

本文档详细记录了如何编译 Proxmox VE (PVE) 内核，将 QS/ES 版本 CPU 的 "Genuine Intel 0000" 显示名称修改为指定的正式版型号（如 8260L），并确保虚拟机也能正确显示。

## 1. 环境准备

### 1.1 获取源码
克隆 PVE 内核仓库并切换到目标版本（本例以 6.17.4-2 为例）：

```bash
git clone git://git.proxmox.com/git/pve-kernel.git
cd pve-kernel
# 切换到对应的 commit (例如 6.17.4-2 的 ABI 更新提交)
git checkout 6e6197515ceed69bdbaf63a004acd70531a6665d
```

### 1.2 初始化子模块
为了加速下载，建议修改 `.gitmodules` 使用 `git://` 协议，并使用浅克隆：

```bash
# 修改 .gitmodules (可选，如果 https 慢)
sed -i 's|https://git.proxmox.com/git/|git://git.proxmox.com/git/|g' .gitmodules

# 初始化并更新子模块
git submodule sync
git submodule update --init --recursive --depth 1 --jobs 8
```

### 1.3 安装构建依赖
```bash
apt update
apt install devscripts build-essential
# 生成并安装依赖包
make build-dir-fresh # 这一步可能会因为缺依赖报错，但会生成 debian/control
mk-build-deps -ir proxmox-kernel-6.17.4/debian/control
# 如果提示构建 ZFS 缺依赖，需额外安装：
apt install dh-python python3-sphinx python3-cffi libffi-dev
```

## 2. 修改源码 (制作补丁)

为了保持源码整洁，采用 PVE 推荐的补丁方式。

### 2.1 创建补丁文件
创建文件 `patches/kernel/0036-override-cpu-model-name.patch`：

```diff
From: Custom Builder <admin@local>
Date: Mon, 2 Feb 2026 12:00:00 +0000
Subject: [PATCH] Override QS/ES model name for 8260L

--- a/arch/x86/kernel/cpu/common.c
+++ b/arch/x86/kernel/cpu/common.c
@@ -806,6 +806,12 @@ static void get_model_name(struct cpuinfo_x86 *c)
 	}
 
 	*(s + 1) = '\0';
+
+	/* Override QS/ES model name */
+	if (strstr(c->x86_model_id, "Genuine Intel(R) CPU 0000")) {
+		snprintf(c->x86_model_id, sizeof(c->x86_model_id),
+			 "Intel(R) Xeon(R) Platinum 8260L CPU @ 2.40GHz");
+	}
 }
```

## 3. 编译内核

```bash
# 1. 准备构建目录（会自动应用 patches/kernel/ 下的所有补丁）
make build-dir-fresh

# 2. 开始编译（耗时较长）
make deb
```

编译完成后，当前目录下会生成 `.deb` 文件，例如 `proxmox-kernel-6.17.4-2-pve_6.17.4-2_amd64.deb`。

## 4. 安装与验证

### 4.1 解决冲突并安装
由于我们自己编译的内核没有官方签名，与已安装的 `proxmox-kernel-*-signed` 包冲突。

```bash
# 1. 保护旧内核不被自动删除
apt-mark manual proxmox-kernel-6.17.4-1-pve-signed

# 2. 卸载冲突的签名版内核与元包
apt remove proxmox-kernel-6.17 proxmox-kernel-6.17.4-2-pve-signed

# 3. 安装定制内核
dpkg -i proxmox-kernel-6.17.4-2-pve_6.17.4-2_amd64.deb

# 4. 重启
reboot
```

### 4.2 验证宿主机
重启后检查：
```bash
lscpu | grep "Model name"
# 应显示: Intel(R) Xeon(R) Platinum 8260L CPU @ 2.40GHz
```

## 5. 虚拟机配置 (重要)

仅修改宿主机内核 **不会** 改变虚拟机的 CPU 显示，因为 KVM/QEMU 默认直接读取物理硬件 CPUID，绕过了宿主机的 `/proc/cpuinfo`。

### 5.1 单个虚拟机修复
修改虚拟机配置文件 `/etc/pve/qemu-server/<VMID>.conf`，添加 `args` 参数来覆盖 Model ID。

**确保现有 CPU 类型为 'Host' (推荐)**：
```ini
args: -cpu 'host,model_id=Intel(R) Xeon(R) Platinum 8260L CPU @ 2.40GHz'
```

### 5.2 批量修复命令
如果您有大量虚拟机，可以使用以下命令批量设置（需先停止虚拟机）：

```bash
# 为所有虚拟机添加 args 参数
for vmid in $(qm list | awk 'NR>1 {print $1}'); do 
  qm set $vmid --args "-cpu 'host,model_id=Intel(R) Xeon(R) Platinum 8260L CPU @ 2.40GHz'"
  echo "Updated VM $vmid"
done

# 批量重启（可选，慎用）
# for vmid in ...; do qm stop $vmid && qm start $vmid; done
```
