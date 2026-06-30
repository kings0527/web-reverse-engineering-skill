# 参考项目Git清单

> 以下项目为本Skill的学习和参考资源，按需克隆。不包含在Skill打包中。
> 克隆命令：`git clone --depth 1 <URL>`

## JSVMP与逆向核心

| 项目 | URL | 说明 | 推荐阅读 |
|------|-----|------|---------|
| baishuijianjia/jsvmp | https://github.com/baishuijianjia/jsvmp | 栈式JS虚拟机完整实现（38条操作码） | src/opcodes.js, src/vm.js, src/compiler.js |
| 0xfffb/twisted | https://github.com/0xfffb/twisted | TypeScript JSVMP（55操作码+SSA IR+混淆Pass） | src/constant.ts, src/vm/vm.ts |
| xiaoweigege/jsvmp-repository | https://github.com/xiaoweigege/jsvmp-repository | 国内网站JSVMP算法实战收集 | 全部 |
| qq185863818/MeowShield | https://github.com/qq185863818/MeowShield | 高级JS混淆（全JSVMP虚拟化+动态代码生成） | - |
| notemrovsky/tiktok-reverse-engineering | https://github.com/notemrovsky/tiktok-reverse-engineering | TikTok JS VM逆向（77操作码分析） | README.md, disasm.js |
| carcabot/tiktok-xgnarly-decoded | https://github.com/carcabot/tiktok-xgnarly-decoded | TikTok X-Gnarly签名完整逆向 | src/cipher.js, src/encode.js |

## 反检测与自动化

| 项目 | URL | 说明 | 推荐阅读 |
|------|-----|------|---------|
| ultrafunkamsterdam/nodriver | https://github.com/ultrafunkamsterdam/nodriver | 直接CDP驱动（31/31反检测通过） | nodriver/core/connection.py |
| rebrowser/rebrowser-patches | https://github.com/rebrowser/rebrowser-patches | CDP泄露修补（addBinding/隔离世界/enableDisable） | patches/ |
| Kaliiiiiiiiii-Vinyzu/patchright | https://github.com/Kaliiiiiiiiii-Vinyzu/patchright | Playwright fork反检测（32模块修补） | driver_patches/ |

## JS反混淆

| 项目 | URL | 说明 | 推荐阅读 |
|------|-----|------|---------|
| j4k0xb/webcrack | https://github.com/j4k0xb/webcrack | obfuscator.io反混淆+webpack解包 | packages/webcrack/src/deobfuscate/ |
| kuizuo/js-deobfuscator | https://github.com/kuizuo/js-deobfuscator | Babel AST自动JS反混淆（webcrack增强版） | packages/deob/src/ |

## AI辅助逆向

| 项目 | URL | 说明 | 推荐阅读 |
|------|-----|------|----------|
| Mouseww/anything-analyzer | https://github.com/Mouseww/anything-analyzer | 全能协议分析+AI+MCP Server | src/main/mcp/ |
| zhizhuodemao/js-reverse-mcp | https://github.com/zhizhuodemao/js-reverse-mcp | AI-first JS逆向MCP Server（Google出品） | src/tools/ |
| zhaoxuya520/reverse-skill | https://github.com/zhaoxuya520/reverse-skill | 逆向工程Skill知识库 | skills/js-reverse/ |

## 执行工具

| 项目 | URL | 说明 | 推荐阅读 |
|------|-----|------|----------|
| kings0527/web-trace | https://github.com/kings0527/web-trace | MCP Server Chrome Extension（JSVMP追踪+WASM分析+隐蔽Hook） | docs/SKILL-INTEGRATION.md |

## 引擎级AI逆向

| 项目 | URL | 说明 | 推荐阅读 |
|------|-----|------|----------|
| WhiteNightShadow/firefox-reverse | https://github.com/WhiteNightShadow/firefox-reverse | Firefox引擎级AI逆向智能体（需Firefox环境，45+工具） | docs/, README |
