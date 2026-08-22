# HKUSTGZ Printer Direct

在保持 Shadowrocket 连接的情况下，让 HKUST(GZ) 校园公共打印机域名使用校园 DNS 并直连。

这是一个非官方、最小范围的 Shadowrocket 本地模块。仓库不包含代理节点、订阅、账号、安装包或学校内部文件。

## Status

- macOS 打印客户端安装：已验证
- `HKUSTGZ_Printer` 队列：已验证为 idle / enabled
- Shadowrocket：保持连接
- 修复范围：仅 `printer.hkust-gz.edu.cn` 和校内 `10.0.0.0/8` 地址

## Symptom

学校提供的 macOS 安装包在 `Running package scripts...` 阶段退出：

```text
Setup will now quit because failed to download required system patch from console,
please check network! (FILE:Mac.ini)
```

![Installer Mac.ini download error](assets/mac-ini-download-error.png)

安装包签名和 Apple notarization 均正常，因此故障不在安装包完整性。

## Investigation

1. 展开安装包并查看安装日志，确认脚本需要访问：

   ```text
   https://printer.hkust-gz.edu.cn:8110/ini/Mac.ini
   ```

2. 校园网络 DHCP 已下发校园 DNS：

   ```text
   10.90.63.2
   10.90.63.3
   ```

3. Shadowrocket 连接时，打印域名被其 fake-IP DNS 解析为 `198.18.*`，随后 TLS 握手失败。
4. 直接使用校园 DNS 查询时，域名返回校内 `10.*` 地址；对同一 URL 的访问成功返回 HTTP 200。
5. 现有代理配置已经允许 `10.0.0.0/8` 直连，因此缺失的是“该域名使用校园 DNS”的局部例外，不是全局关闭代理。

## Root Cause

`printer.hkust-gz.edu.cn` 是校内服务。Shadowrocket 的 fake-IP DNS 接管了域名解析，使请求没有获得校园 DNS 返回的内网地址。即使内网 IP 已设置为 DIRECT，域名在解析阶段仍然走错。

## Module

[`HKUSTGZ_Printer_Direct.sgmodule`](HKUSTGZ_Printer_Direct.sgmodule) 做两件事：

```ini
[Host]
printer.hkust-gz.edu.cn = server:10.90.63.2

[Rule]
DOMAIN,printer.hkust-gz.edu.cn,DIRECT
IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
```

- 只让打印域名使用校园 DNS `10.90.63.2`。
- 让打印域名和校园内网地址直连，不改动其他网站的代理策略。

## Install

- [ ] 连接 HKUST(GZ) 校园网络。
- [ ] 保持 Shadowrocket 连接。
- [ ] 下载本仓库的 `HKUSTGZ_Printer_Direct.sgmodule`。
- [ ] 在 Shadowrocket 的 `配置 -> 模块` 中导入并启用。
- [ ] 重新运行学校提供的打印客户端安装包。
- [ ] 安装完成后检查打印队列。

```bash
dig @10.90.63.2 +short printer.hkust-gz.edu.cn
lpstat -p -d
```

成功时，DNS 应返回 `10.*` 校内地址；打印队列应显示 `idle` / `enabled`。

## Troubleshooting Runbook

- [ ] 确认当前连接的是校园网络，不是手机热点或校外网络。
- [ ] 确认模块仍为启用状态；机场订阅更新后不应删除本地模块。
- [ ] 用校园 DNS 检查域名是否返回 `10.*` 地址。
- [ ] 若 DNS 正常但安装仍失败，检查系统日期时间和安装日志。
- [ ] 若安装成功但无法打印，用 `lpstat -p -d` 检查队列状态。
- [ ] 若校园 DNS 地址或打印域名发生变化，以学校最新 IT 指南为准并更新模块。

## Security Notes

- 模块不包含代理服务器、凭证、Cookie 或个人信息。
- 不建议把打印服务器的当前 `10.*` 地址硬编码进模块；它可能变化，校园 DNS 才是权威入口。
- 不要为了打印永久关闭 Shadowrocket。局部 DNS 和 DIRECT 规则可以把影响限制在单个校内服务。

## License

MIT
