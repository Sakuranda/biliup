# TrueNAS SCALE 25.10 / Intel N100

`n100-truenas` 基于 `j1900-debian13`。镜像运行环境仍是 Debian 13。Chromium Headless Shell 和 FFmpeg 随镜像提供，容器无需宿主机的 systemd 服务或 cgroup 写权限。剪辑进程使用较低 CPU 优先级，Compose 将服务限制在 2 个 CPU 和 2 GiB 内存。Chromium 在非 root 容器里运行，其内置沙箱通过启动参数关闭，以便在 Docker 默认限制下运行内置渲染页面。

## 部署

1. 在 TrueNAS Apps 中选应用池，将 Docker 镜像和容器存放在该池。
2. 在数据池创建 `/mnt/stvx000/files/biliup`，容器以 UID/GID `568:568` 运行。赋予该账户目录写权限，或通过 `BILIUP_UID` 和 `BILIUP_GID` 指定已有数据的所有者。
3. 将 `compose.yaml` 用于 TrueNAS **Apps → Install via YAML**。也可用 `docker compose -f deploy/n100/compose.yaml up -d`。启动后打开 NAS 的 19159 端口并完成 Biliup 的首次登录设置。
4. 启用 Intel 硬件加速时，先确认宿主机的 Intel DRM 渲染节点，通过 Compose 的 `devices` 映射该节点，并用 `group_add` 加入宿主机渲染节点的数值 GID。将 `BILIUP_VAAPI_DECODE_DEVICE` 设置为容器内的节点路径，例如 `/dev/dri/renderD128`，即可为转码、剪辑、代理和抽帧的视频输入请求 VAAPI 解码。音轨处理和流拷贝不需要视频解码。硬解后的帧可以交给现有字幕、遮挡等滤镜。

整个 `/opt` 挂载到数据盘，数据库、录制文件、日志和编辑工程随该目录保存。清理服务每小时检查一次，并只移除 72 小时前生成的剪辑预览代理。建议在需要回滚前备份该目录。

## 更新与回滚

提交推送到 `n100-truenas` 后，GitHub Actions 构建并检查 amd64 镜像。每次成功测试都会附带可用 `docker load -i biliup-n100.tar.gz` 导入的离线镜像，并向 GHCR 发布 `n100-truenas` 和 `n100-<完整提交哈希>` 标签。生产部署可把 `BILIUP_IMAGE` 固定在已验证的提交标签。回滚时切回旧标签，并按需恢复更新前备份的数据。
