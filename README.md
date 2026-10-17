# ZrLog Process Artifact

[GitHub Marketplace](https://github.com/marketplace/actions/zrlog-process-artifact) ·
[Releases](https://github.com/zrlog/zrlog-process-artifact/releases)

GitHub Action and Bash client for processing Linux native binaries through an authenticated
`zrlog-artifact-service` endpoint, with SHA-256 and size verification.

供 ZrLog 主程序和插件构建复用的 Native 制品处理客户端，提供 GitHub Composite Action
和 Bash 命令行入口。客户端将原始可执行文件上传到 `zrlog-artifact-service`，轮询处理任务，
下载结果并核对 SHA-256 和文件大小后，原位替换输入文件。

本仓库维护服务接入、重试、结果校验和临时结果清理。应用编译、插件下载、ZIP/DEB
组装、更新清单和最终发布由各消费者维护；处理服务的服务端实现也独立维护。

## GitHub Actions 接入

Linux runner 需要 Bash、curl（支持 `--fail-with-body`）、jq、sha256sum 和常规 GNU 工具。
处理前完成 Native 编译，处理后再计算最终制品摘要、组包或上传。

快捷接入使用 `@v1`，跟随兼容的 1.x 稳定版本：

```yaml
- name: Process Linux native artifact
  if: runner.os == 'Linux'
  uses: zrlog/zrlog-process-artifact@v1
  with:
    artifact-file: build/my-plugin.bin
    artifact-name: my-plugin
    artifact-version: 1.0.0
    artifact-architecture: Linux-amd64
    service-url: ${{ vars.ARTIFACT_SERVICE_URL }}
    service-token: ${{ secrets.ARTIFACT_SERVICE_TOKEN }}
```

也可以使用 `@v1.0.1` 固定版本。内部正式构建继续使用经验证的完整提交 SHA，
避免移动标签改变构建输入；在检出本仓库后，用 `git rev-parse 'v1.0.1^{commit}'`
取得该版本的 40 位提交 SHA，再将 `@v1` 替换为 `@<full-commit-sha>`。

| 输入 | 必填 | 说明 |
| --- | --- | --- |
| `artifact-file` | 是 | 已生成的本地可执行文件；校验成功后原位替换并设为可执行。 |
| `artifact-name` | 是 | 稳定的制品名称。 |
| `artifact-version` | 是 | 本次制品的真实版本。 |
| `artifact-architecture` | 是 | 操作系统和架构，例如 `Linux-amd64`、`Linux-arm64`。 |
| `service-url` | 是 | 服务完整基址，包含部署的上下文路径。 |
| `service-token` | 是 | 服务 Bearer token，由消费者的 Actions Secret 提供。 |
| `poll-interval-seconds` | 否 | 轮询间隔，默认 `5` 秒。 |
| `process-timeout-seconds` | 否 | 上传完成后的轮询时限，包含排队和处理，默认 `14400` 秒（4 小时）；不包含上传、下载耗时。 |

名称、版本和架构只接受字母、数字、点、下划线与连字符。间隔和时限必须为正整数。
Action 没有额外输出；后续步骤继续使用 `artifact-file` 指定的文件。

工作流的 job 总时限需要覆盖编译、上传、等待处理、下载和后续组包。
建议将调用方 job 的 `timeout-minutes` 设为 `360`（6 小时），为默认 4 小时等待留出其他步骤的时间；
需要不同等待时限时，通过 `process-timeout-seconds` 或命令行环境变量覆盖。

服务可配置为 `https://webdav.zrlog.com/artifact`。服务使用
`ARTIFACT_CONTEXT_PATH=/artifact` 时，基址必须包含 `/artifact`；客户端在其后追加
`/api/v1/...`，尾部斜杠可省略。消费者配置 `ARTIFACT_SERVICE_URL` Variable 和
`ARTIFACT_SERVICE_TOKEN` Secret，token 对应服务端的 `ARTIFACT_API_TOKEN`。
本仓库的 Variable 不会自动传递给其他仓库，消费者需各自配置或使用授权的组织配置。

## 命令行接入

在包含最终组包逻辑的脚本中使用客户端时，先检出固定提交，再调用同一份 Bash 实现：

```bash
git clone https://github.com/zrlog/zrlog-process-artifact.git /tmp/zrlog-process-artifact
git -C /tmp/zrlog-process-artifact checkout --detach <full-commit-sha>
export ARTIFACT_SERVICE_URL=https://webdav.zrlog.com/artifact
# 通过环境注入 ARTIFACT_SERVICE_TOKEN。
bash /tmp/zrlog-process-artifact/scripts/process-artifact.sh \
  build/my-plugin.bin my-plugin 1.0.0 Linux-amd64
```

可选环境变量为 `ARTIFACT_POLL_INTERVAL_SECONDS`、`ARTIFACT_PROCESS_TIMEOUT_SECONDS`。
命令行参数与 Action 的前四个输入对应，不依赖消费者的 `build.properties`。

## 版本约定

- `v1.0.0` 等完整版本标签固定对应提交，发布后不移动。
- `v1` 指向最新经过验证的兼容 1.x 稳定版本，只在正式版本发布时更新。
- 改变必填输入、结果校验或失败处理契约时，按兼容性影响选择新的主版本。
- 每次发布前等待目标提交的客户端 CI 通过，再创建版本标签和 Release，并同步主版本标签。

版本变化见 [CHANGELOG.md](CHANGELOG.md)，已发布版本见
[GitHub Releases](https://github.com/zrlog/zrlog-process-artifact/releases)。

## 校验与故障处理

客户端依次调用上传接口 `POST /api/v1/uploads`、任务接口
`GET /api/v1/jobs/detail?id=...` 和结果接口 `GET /api/v1/jobs/result?id=...`。
`PENDING`、`PROCESSING` 会继续轮询，`FAILED`、未知状态、响应无效或超时会终止构建。

上传、轮询和清理响应先写入临时文件，curl 重试时丢弃上一次响应，避免网关 504 HTML
与成功 JSON 串接。下载也使用临时文件；SHA-256、文件大小全部匹配后才替换原文件。
处理或校验失败时保留原文件，退出时清理本地临时文件。

替换成功后通过 `DELETE /api/v1/jobs/result?id=...` 确认删除服务端临时结果。
清理失败只记录警告，未确认的结果由服务端 TTL 回收。

## 验证

```bash
bash -n scripts/process-artifact.sh
python3 -m unittest discover -s tests -v
```

测试通过本机 HTTP 夹具执行真实 Bash 和 curl，覆盖网关重试、下载校验、原文件保护
和清理确认，不调用线上服务。CI 在 push 和 pull request 时运行这些检查。

需要检查托管 runner 到服务的 HTTPS 连通性时，手动运行
`.github/workflows/service-probe.yml`。可传入 `service_url`，或配置本仓库的
`ARTIFACT_SERVICE_URL`；工作流同时记录 `https://artifact.zrlog.com/artifact` 的结果。
探测只发送不带认证的 `/health` GET，归档 `service-probe.json`，不上传二进制。
查看归档中的 curl 退出码和 HTTP 状态判断结果；工作流成功仅表示诊断记录已生成。

## 迁移来源

客户端、Action 和原有五项回归测试迁自
[`94fzb/zrlog`](https://github.com/94fzb/zrlog/tree/bcb8d809/.github/actions/process-artifact)。
旧消费者固定的历史提交仍可使用；新接入与后续维护统一使用本仓库。

采用 [Apache License 2.0](LICENSE)。
