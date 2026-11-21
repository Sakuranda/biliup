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

## 预览缓存与录像读取排队

FLV 原片可以直接导入工程；剪辑台按需生成用于浏览器预览和定位的 MP4 缓存。没有现成 MP4 不会使工程的帧数校验失败。

后台代理转换需要保护正在读取的原片。清理任务每分钟申请一次文件写锁，Tokio 的公平读写锁会让后来的读取请求等在写锁后面。旧实现只检查转码队列，无法发现那些尚未进入转码队列的请求，一次长转换就可能让「正在打开录像」、缩略图和原片下载一起等待。

本分支只在检查原片和发布缓存时持有文件读锁，长时间转换期间释放它，让清理和原片下载可以继续。转换完成后，先释放转码额度，再重新取得文件锁并核对原片与工程；被删除、重连或取消的素材不会发布成缓存。前台探测和导出仍优先于后台缓存转换。CPU 配额和 72 小时缓存期限沿用上述配置。

已在独立本机测试服务验证短 FLV 的工程创建、保存、撤销和重做；另用 90 秒慢速缓存跨过每分钟清理周期，同一 FFmpeg 进程持续运行，原片 Range 请求约 0.2 秒返回，缓存最终完整生成。

仓库另提供锁竞争回归测试，检查清理写锁、转码额度释放顺序及素材重连后的缓存校验。本次设备上的完整库测试因编译和链接优化耗时过长未跑完，以下命令供后续执行。

```sh
cargo test --release -p biliup-cli --lib cache_encode_releases_lifecycle_and_cpu_before_publication -- --nocapture
```

## 独立剪辑网页入口的开机重试

使用独立的 `biliup-editor-ui-hotfix.service` 在 19162 端口加载新版网页时，网卡尚未拿到局域网 IP 会使 Nginx 报 `Cannot assign requested address`。默认的快速重启会耗尽启动次数，导致入口在开机后一直关闭。下面的配置让它每 5 秒重试，直到网络就绪；只作用于已有的独立网页服务。

```sh
sudo install -d -m 0755 /etc/systemd/system/biliup-editor-ui-hotfix.service.d
sudo install -m 0644 deploy/j1900/biliup-editor-ui-hotfix-retry.conf \
  /etc/systemd/system/biliup-editor-ui-hotfix.service.d/retry.conf
sudo systemctl daemon-reload
sudo systemctl reset-failed biliup-editor-ui-hotfix.service
sudo systemctl start biliup-editor-ui-hotfix.service
systemctl show biliup-editor-ui-hotfix.service -p ActiveState -p RestartUSec -p StartLimitIntervalUSec
```

验证独立入口可以打开 `/workbench`。原来的 `biliup.service` 在上述操作中保持运行，录制无需重启。

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
