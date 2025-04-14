在Arch Linux中配置U盘自动挂载可通过以下两种主流方案实现，根据需求选择适合的方法：

---

### **方案一：使用`udisks2` + `udiskie`（推荐用于桌面环境）**
1. **安装必要工具**
```bash
sudo pacman -S udisks2 udiskie ntfs-3g exfat-utils # 支持NTFS/exFAT格式
```

2. **配置挂载路径**
创建或修改udev规则文件，使U盘挂载到`/media`而非默认的`/run/media/USER`目录：
```bash
sudo nano /etc/udev/rules.d/99-udisks2.rules
```
添加内容：
```udev
# 强制共享挂载到/media
ENV{ID_FS_USAGE}=="filesystem|other|crypto", ENV{UDISKS_FILESYSTEM_SHARED}="1"
```

3. **自动启动挂载服务**
在用户会话启动时（如`~/.xinitrc`或桌面环境启动脚本）添加：
```bash
udiskie -t & # 启动托盘图标并自动挂载
```

---

### **方案二：自定义udev规则（适用于无桌面环境或高级配置）**
1. **创建udev规则文件**
```bash
sudo nano /etc/udev/rules.d/99-usb-automount.rules
```
写入以下规则（示例支持自动挂载和卸载）：
```udev
ACTION=="add", KERNEL=="sd[b-z][0-9]", RUN+="/usr/bin/mkdir -p /mnt/%E{ID_MODEL}_%k"
ACTION=="add", KERNEL=="sd[b-z][0-9]", RUN+="/usr/bin/mount -t auto -o uid=1000,gid=1000,rw,noexec,nodev,noatime /dev/%k /mnt/%E{ID_MODEL}_%k"
ACTION=="remove", KERNEL=="sd[b-z][0-9]", RUN+="/usr/bin/umount -l /mnt/%E{ID_MODEL}_%k"
ACTION=="remove", KERNEL=="sd[b-z][0-9]", RUN+="/usr/bin/rmdir /mnt/%E{ID_MODEL}_%k"
```
- **参数说明**：
- `uid/gid=1000`：替换为实际用户ID（通过`id -u`查看）
- `noexec`：禁止执行U盘中的可执行文件（可选）
- `%E{ID_MODEL}`：使用U盘型号作为挂载目录名，避免重复

2. **重载udev规则**
```bash
sudo udevadm control --reload
```

---

### **补充说明**
1. **文件系统支持**
- **NTFS**：需安装`ntfs-3g`
- **exFAT**：需安装`exfat-utils`
- **FAT32**：内核原生支持

2. **权限问题**
若挂载后普通用户无法读写，可添加`-o umask=000`参数或通过`chmod`修改目录权限。

3. **日志调试**
通过`journalctl -f`实时查看挂载日志，排查规则匹配或权限错误。

---

### **扩展：自动挂载进阶方法**
- **使用autofs**：通过`/etc/auto.master`配置按需挂载，适合服务器环境。
- **桌面集成**：GNOME/KDE等桌面环境通常已集成`udisks2`，无需额外配置即可自动挂载到`/run/media`。

建议优先使用`udisks2`方案，兼容性更好且维护成本低。若需深度定制挂载行为（如自定义路径、权限），再选择自定义udev规则方案。