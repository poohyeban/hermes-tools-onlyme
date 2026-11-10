# Hermes 工具镜像

这个仓库在 GitHub Actions 上构建两个独立的 Linux AMD64 镜像，并发布到 GitHub Container Registry（GHCR）。两个镜像来自同一次代码提交，部署时可分别选择版本和摘要。

| 构建目录 | 镜像内容 | 版本标签 |
| --- | --- | --- |
| `opensandbox-mcp/` | OpenSandbox 官方 MCP 程序及 Python 依赖，运行用户为 UID 10000 | `1.2.0` |
| `tailnet-runtime/` | Debian、Python、OpenSSH 客户端和 UID 65532 的运行账户 | `2` |

## 本地构建和检查

在仓库根目录运行以下命令。构建机需要 Docker Buildx，以及访问基础镜像仓库、PyPI 和 Debian 软件源的网络连接。

```sh
docker buildx build --platform linux/amd64 --load -t opensandbox-mcp:test ./opensandbox-mcp
docker run --rm --network none opensandbox-mcp:test --help

docker buildx build --platform linux/amd64 --load -t tailnet-runtime:test ./tailnet-runtime
docker run --rm --network none --user 65532:65532 tailnet-runtime:test sh -ec 'python3 --version && ssh -V'
```

MCP 构建会按 `requirements.lock` 下载 wheel 并校验每个 SHA-256。更新 MCP 时应同时更新包版本、对应 AMD64 wheel 的哈希和工作流的版本标签。两个 Dockerfile 都固定了基础镜像摘要。Debian 软件源会更新，重新构建可能产生新摘要；回退时应使用之前发布的摘要。

## 使用 GitHub Actions 发布

1. 在仓库内启用 Actions，并允许本仓库工作流写入关联的 Packages。工作流使用 GitHub 为当前任务签发的凭据。
2. 将构建文件的修改推送到 `main`，或在 **Actions → Build and publish images → Run workflow** 手动执行。
3. 等待两个构建任务通过。每项都会在断网容器内做启动检查，再推送版本标签与 `git-<完整提交号>` 标签。
4. 打开此次运行的 **Summary**，复制两个 `Verified image` 地址。格式为 `ghcr.io/<owner>/<repository>/<component>@sha256:<digest>`。在生产 Compose 的 `image` 字段使用这个完整地址。

## 在部署机器拉取

私有包需要具有包读取权限的凭据。使用可读取该包且具有 `read:packages` 的 classic PAT，在部署机器的交互式密码提示中输入。将下面的占位符换成实际值。

```sh
docker login ghcr.io -u <github-account>
docker pull ghcr.io/<owner>/<repository>/opensandbox-mcp@sha256:<digest>
docker pull ghcr.io/<owner>/<repository>/tailnet-runtime@sha256:<digest>
```

MCP 服务在运行时从部署系统取得沙箱 API 地址和密钥。网络客户端运行时所需的启动脚本、SSH 配置、Tailscale 程序、CA 证书和持久化节点状态由部署系统挂载。构建镜像时只需要上表列出的目录。

## 备份与恢复

记录每份快照使用的镜像地址和摘要。还需保存应用数据、运行配置、凭据、节点状态及挂载脚本，并确保恢复机器可以读取私有包。在所有仍保留的快照都不再引用某个摘要之后，才可清理对应的镜像版本。
