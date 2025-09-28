# lxd_images 使用教程

本文档介绍如何利用仓库内脚本同步官方配置、构建并验证 LXD 容器镜像。

## 1. 环境准备

- 建议使用 Debian/Ubuntu 系统，确保具备 `sudo` 权限。
- 预安装以下依赖（脚本缺少时会尝试自动安装）：`zip`、`jq`、`umoci`、`snapd`、`debootstrap`。
- `lxd-imagebuilder` 通过 `snap` 安装，脚本会自动执行 `sudo snap install --edge lxd-imagebuilder --classic`。
- 如使用自建环境，请预先安装 LXD 并完成 `lxd init` 初始化。

## 2. 同步并修改官方 YAML

脚本 `clone_modify_yaml.sh` 会：

1. 从 `https://images.lxd.canonical.com/streams/v1/images.json` 获取最新映像信息；
2. 下载各发行版的 `image.yaml` 至 `images_yaml/`；
3. 自动插入常用工具（curl、wget、openssh-server、cron 等）并写入 `.modified` 标记。

执行命令：

```bash
bash clone_modify_yaml.sh
```

运行后 `images_yaml/` 目录会生成：

- 原始 YAML 文件（如 `debian.yaml`）；
- `.meta` 文件保存远端 ETag/Last-Modified，便于增量更新；
- `.modified` 标记避免重复修改。

## 3. 构建 LXD 容器镜像

脚本 `build_images.sh` 支持多发行版、架构与 variant：

```bash
bash build_images.sh <distro> <is_build> <arch>
```

- `<distro>`：例如 `debian`、`ubuntu`、`kali`、`alpine` 等；
- `<is_build>`：`true` 进行构建，`false` 仅输出将生成的 zip 名称；
- `<arch>`：`amd64` 或 `arm64`（支持别名 `x86_64`、`aarch64`）。

示例：

```bash
# 列出将生成的镜像文件
bash build_images.sh debian false amd64

# 实际构建并打包
sudo bash build_images.sh debian true amd64
```

构建成功后会得到形如 `debian_12_bookworm_amd64_cloud.zip` 的压缩包，内含 `lxd.tar.xz` 与 `rootfs.squashfs`，可直接导入 LXD：

```bash
lxc image import lxd.tar.xz rootfs.squashfs --alias my-debian
```

## 4. 更新镜像列表（可选）

`clone_modify_yaml.sh` 末尾提供示例逻辑，可批量检测镜像是否已发布，并将名称写入：

- `x86_64_all_images.txt`
- `arm64_all_images.txt`

根据需要调整脚本中的发行版与架构列表后再次运行即可。

## 5. 验证镜像（可选）

脚本 `test.sh` 用于下载并验证镜像：

```bash
sudo bash test.sh <镜像文件名>
```

流程：

1. 从 GitHub Releases 拉取对应 zip；
2. 解压并导入本地 LXD，启动容器；
3. 检测 SSH 服务与网络连通性；
4. 将成功镜像写入 `x86_64_fixed_images.txt` 或 `arm64_fixed_images.txt`。

## 6. 常见问题

- **snap 安装失败**：确保 `snapd` 已启动，可执行 `sudo systemctl start snapd` 并稍候再试。
- **lxd-imagebuilder 下载缓慢**：可在脚本外部预先安装后再运行构建脚本。
- **构建报错**：检查 `images_yaml/<distro>.yaml` 是否手动修改过，或查看脚本输出的 `lxd-imagebuilder` 命令日志。

## 7. 后续操作

- 导入自定义镜像后，可按需创建并运行容器：

  ```bash
  lxc init my-debian test
  lxc start test
  lxc exec test -- bash
  ```

- 完成测试后，记得清理临时镜像或容器：

  ```bash
  lxc delete -f test
  lxc image delete my-debian
  ```

通过以上步骤即可完成 LXD 容器镜像的同步、构建与验证流程。
