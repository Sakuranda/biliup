# 在 J1900 / Debian 13 上部署 biliup 剪辑功能

本分支以 biliup v1.3.5 为基线，适用于 Debian 13 上已配置的 `biliup.service`。部署文件对应 `User=biliup`、`Group=biliup`、`WorkingDirectory=/srv/biliup` 和 `/opt/biliup/biliup`。系统使用 cgroup v2，systemd 支持 `DelegateSubgroup`。

## 构建

准备 Node.js 20.9 或更新版本以及支持依赖的 Rust 工具链。本次构建使用 Node.js 24 和 Rust 1.100.0。在 Debian 上准备编译与运行依赖。

```sh
sudo apt-get install --no-install-recommends \
  build-essential pkg-config cmake libssl-dev \
  ffmpeg chromium-headless-shell chromium-sandbox
git clone --branch j1900-debian13 https://github.com/Sakuranda/biliup.git
cd biliup
npm ci
npm run build
cargo build --release --locked --bin biliup
```

`npm run build` 会生成 Rust 二进制所需的静态网页和 `resources/editor/export-runtime.js`。Linux 无头渲染使用 Debian 的 Chromium Headless Shell 和 SwiftShader。

## 更新现有服务

在没有进行中的录制、下载或导出任务时操作。先备份 `/srv/biliup/data/`，并确认服务配置与本页开头的路径一致。以下命令在仓库根目录运行，使用同一个 shell 保存回滚目录变量。

```sh
systemctl cat biliup.service
backup="/opt/biliup/backup-$(date +%Y%m%d-%H%M%S)"
sudo install -d -m 0700 "$backup"
sudo cp -a /opt/biliup/biliup "$backup/biliup"
sudo stat -c '%a %U %G %n' /srv/biliup/renders/editor-proxies /srv/biliup/renders/editor-cache
sudo install -d -m 0755 /etc/systemd/system/biliup.service.d
sudo install -m 0755 deploy/j1900/editor-ffmpeg deploy/j1900/editor-browser /opt/biliup/
sudo install -m 0644 deploy/j1900/editor-render.conf /etc/systemd/system/biliup.service.d/
sudo install -m 0644 deploy/j1900/biliup-editor-cache-clean.service \
  deploy/j1900/biliup-editor-cache-clean.timer /etc/systemd/system/
sudo install -d -o biliup -g biliup -m 0700 \
  /srv/biliup/renders/editor-proxies /srv/biliup/renders/editor-cache
sudo chown biliup:biliup /srv/biliup/renders/editor-proxies /srv/biliup/renders/editor-cache
sudo chmod 0700 /srv/biliup/renders/editor-proxies /srv/biliup/renders/editor-cache
sudo install -m 0755 target/release/biliup /opt/biliup/biliup.next
sudo mv -T /opt/biliup/biliup.next /opt/biliup/biliup
sudo systemctl daemon-reload
sudo systemctl restart biliup.service
sudo systemctl enable --now biliup-editor-cache-clean.timer
```

如果之前关闭预览时也把已存在的项目缓存目录设为只读，请给这些项目目录恢复 `biliup:biliup` 和 `0700`。

## 验证

1. 登录 Web 界面，展开「直播场次」中一场已结束的录制。选择「下载」，浏览器应保存原始录像分段。
2. 打开一段 FLV 的剪辑工程，确认预览代理生成成功并能导出短片。剪辑运行后可检查配额和节流次数。

   ```sh
   cat /sys/fs/cgroup/system.slice/biliup.service/editor/cpu.max
   cat /sys/fs/cgroup/system.slice/biliup.service/editor/cpu.stat
   ```

   `50000 100000` 表示每 100 毫秒可用 50 毫秒 CPU 时间。所有进入 `editor` 子组的转码和渲染进程共享这一配额。

3. 检查缓存清理定时器。它只匹配 `renders/editor-proxies/<工程 ID>/<64 位十六进制哈希>.mp4` 中生成满 72 小时的普通文件。

   ```sh
   systemctl list-timers biliup-editor-cache-clean.timer
   sudo systemctl start biliup-editor-cache-clean.service
   journalctl -u biliup-editor-cache-clean.service -n 10 --no-pager
   ```

## 回滚

将 `backup` 指向上面保留的备份目录，停用定时器、移除本分支安装的配置，并换回原来的二进制。若需要恢复禁用预览时的目录权限，使用更新前 `stat` 记录的属主和模式。

```sh
sudo systemctl disable --now biliup-editor-cache-clean.timer
sudo rm /etc/systemd/system/biliup.service.d/editor-render.conf \
  /etc/systemd/system/biliup-editor-cache-clean.service \
  /etc/systemd/system/biliup-editor-cache-clean.timer
sudo install -m 0755 "$backup/biliup" /opt/biliup/biliup.rollback
sudo mv -T /opt/biliup/biliup.rollback /opt/biliup/biliup
sudo systemctl daemon-reload
sudo systemctl restart biliup.service
```
