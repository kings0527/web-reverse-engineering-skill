# Web Reverse Engineering Skill

> AI驱动的系统化Web逆向工程方法论与实战指南

## 概述

本Skill为AI编程助手（Codex、Qoder、Claude Code、Cursor等）提供Web逆向工程的系统化指导，覆盖从反爬技术识别到算法还原的完整工作流。

## 特性

- **系统方法论**：四层攻击面模型、六步分析法、10分钟快速判定
- **JS混淆对抗**：字符串解密、控制流反平坦化、JSVMP五步逆向法
- **环境修补**：Proxy监测法、BOM/DOM伪造、反检测技术
- **CDP检测绕过**：12项检测向量、nodriver/patchright/rebrowser三方案
- **协议指纹**：HTTP/2 SETTINGS帧、QUIC传输参数、JA3/JA4
- **WASM逆向**：wabt/Ghidra完整工作流、加密算法特征识别
- **多重保护击破**：组合保护决策矩阵、分层逐步突破
- **长期运维**：版本追踪、快速反演、升级应对

## 安装使用

### Cursor
```bash
git clone https://github.com/anthropic-skills/web-reverse-engineering-skill.git
# .cursorrules 会被自动读取
```

### Qoder / Claude Code / Codex
在Agent的Skills或Rules配置中引用 `SKILL.md`。

### 通用
将 `SKILL.md` 提供给AI助手作为系统上下文。

## 目录结构

```
├── SKILL.md          ← AI Agent直接消费的主指令文件
├── .cursorrules      ← Cursor兼容
├── .clinerules       ← Cline兼容
├── references/       ← 详细参考文档（Agent按需加载）
└── LICENSE
```

## references/ 参考文档

| 文件 | 内容 |
|------|------|
| env-patching.md | 环境修补实战（Proxy监测法、实战案例） |
| cdp-bypass.md | CDP反检测三方案详解 |
| deobfuscation.md | JS反混淆12步流水线 |
| jsvmp-architecture.md | JSVMP架构深度分析 |
| jsvmp-reverse-methodology.md | JSVMP逆向五步法 |
| protocol-fingerprinting.md | HTTP/2 & QUIC协议指纹 |
| wasm-reverse.md | WASM逆向完整指南 |
| layered-protection-bypass.md | 多重保护分层击破 |
| anti-crawler-maintenance.md | 反爬升级应对策略 |
| network-interception.md | mitmproxy网络拦截 |
| chrome-extension-helper.md | Chrome Extension辅助 |
| debugging-techniques.md | DevTools高级调试 |
| ai-reverse-tools.md | AI辅助逆向工具 |
| git-references.md | 参考项目Git清单 |

## 适用范围

- Web请求签名算法逆向
- JS/WASM加密代码分析
- 浏览器反检测与指纹绕过
- 协议层(TLS/HTTP2/QUIC)对抗

## 不适用

- 移动端App原生逆向（Frida/IDA）
- 服务端漏洞挖掘
- 社会工程学

## License

MIT
