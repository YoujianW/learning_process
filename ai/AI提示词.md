#AI提示词

## 项目导读

- 我看不懂这个工程的代码，请你带我读一下，让我了解代码是如何运作的，不要编译环境，只需要带我看懂这个工程就好了，多举令人易懂的例子来辅助讲解，不要太深奥



## 安装SKill

> 安装skill后，对话框输入/xxxskill 即可选择这个skill，再输入提示词即可

- https://github.com/easyeda/easyeda-api-skill 帮我安装这个skill

- 请帮我完整安装并配置 Codex with ChatGPT，全程自动，我是不懂技术的小白，
    所有事情你自己做：
    1. 环境自检：需要 git 和 Node.js ≥ 20，缺什么就自动安装
        （macOS 用 Homebrew，Windows 用 winget），同时安装 cloudflared。
    2. 下载：把 https://github.com/XiaoDuoYa/codex-with-chatgpt 克隆到
        ~/codex-with-chatgpt（已存在就 git pull 更新）。
    3. 构建：在该目录里执行 corepack pnpm install 和 corepack pnpm build。
    4. 安装 Skill：把仓库里的 skill/SKILL.md 复制到
        ~/.codex/skills/codex-with-chatgpt/SKILL.md，并把文件中
        "The codex-with-chatgpt checkout lives at:" 那一行的路径改成实际克隆路径。
    5. 首次配置：按 SKILL.md 里的 first-time setup 流程执行
        （运行 c2c setup，用内置浏览器打开 ChatGPT 配置连接器并输入配对码）。
        全程只用内置浏览器，禁止打开任何第三方浏览器。
    6. 只有遇到需要我登录（ChatGPT / Cloudflare）、验证码或两步验证时才叫我，
        而且一次只告诉我一个动作。
    7. 完成后给我看 ✓ 清单，并确认文件读取测试通过。我不懂 MCP、OAuth、
        Tunnel、端口这些词，不要向我解释；出了问题先自己修。



##CodeX

###登录时手机号验证受阻

[这里使用接码平台临时收验证码](https://juejin.cn/post/7636614559459573812)，购买成功率比较高得接码手机号进行接码，充2美元就行。但是使用国内银行卡还无法付款，显示未开通海外付款，这里推荐花呗，不需要开通海外付款

