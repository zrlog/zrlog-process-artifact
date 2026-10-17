# Changelog

## v1.0.1 — 2026-10-17

- Action 和命令行默认处理等待时限从 `900` 秒（15 分钟）延长至 `14400` 秒（4 小时），覆盖上传后的排队和处理。
- 保留输入与环境变量覆盖方式；说明调用方 job 总时限需要为编译、传输和组包预留时间。

## v1.0.0 — 2026-10-17

- 已发布至 [GitHub Marketplace](https://github.com/marketplace/actions/zrlog-process-artifact)。
- 首个独立版本，提供根目录 Composite Action 与共用 Bash 客户端。
- 支持上传 Native 制品、轮询处理任务、校验 SHA-256 和大小后原位替换，并确认清理临时结果。
- 保留 HTTP 重试的响应隔离、失败时原文件保护，以及清理失败不阻断已验证结果的行为。
- 提供五项本机 HTTP 回归测试和独立 CI、手动 HTTPS 探测。
- 提供 `@v1`、`@v1.0.0` 与完整提交 SHA 三种引用方式；内部构建继续固定完整 SHA。
