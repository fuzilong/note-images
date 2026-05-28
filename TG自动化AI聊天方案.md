# 基于 Python Pyrogram / Telethon 编写 Userbot 脚本（最灵活、推荐）

## 准备工作：

1. **Telegram API 凭证**：访问 [my.telegram.org](https://my.telegram.org/)，登录你的 Telegram 账号，点击 "API development tools"，创建一个应用，获取你的 `api_id` 和 `api_hash`。

   * **App title (应用标题)：** 保持你截图里的 `group-score-wh01` 即可（或者随便换个英文名）。
   * **Short name (简称)：** 保持你截图里的 `groupscore01` 即可（必须是 5-32 位的纯英文或数字）。
   * **URL (网址)：** **可以留空不填**。如果系统报错提示必填，随便写一个合法的网址即可，例如：`https://google.com` 或 `https://baidu.com`（这不影响脚本运行）。
   * **Platform (平台)：** 既然你在电脑上跑 Python 脚本，勾选 **`Desktop`**
   * **Description (描述)：** **可以留空不填**。如果报错，随便敲几个英文，比如 `my test script`。

2. **AI 的 API Key**：去服务商平台申请一个 API 密钥。推荐使用 Google Gemini（目前有免费层），或者 OpenAI、DeepSeek 等（需要充值少许话费）。

3. **安装 Python和必要的Python 依赖库**：

  Python安装 https://www.python.org/downloads/

  在电脑终端（Terminal）或命令行中运行以下命令安装所需的核心库：

  ```bash
  pip install telethon google-generativeai
  ```



## 脚本程序

python脚本：

```python
import os
import asyncio
import random
import re
from telethon import TelegramClient

# 🔥 强行注入本地 Clash 代理
os.environ['http_proxy'] = 'http://127.0.0.1:7890'
os.environ['https_proxy'] = 'http://127.0.0.1:7890'
os.environ['all_proxy'] = 'socks5://127.0.0.1:7890'

# ================= 🔧 核心配置区域 =================
# 1. 填入你从 my.telegram.org 申请到的密钥
API_ID = 12345678               # 替换为你的 api_id (必须是数字，不带引号)
API_HASH = 'xxxxxxxxxxxxx'     # 替换为你的 api_hash (带单引号)

# 2. 填入你从 Google AI Studio 申请到的 Gemini 密钥
# 新版 Gemini SDK 配置
GEMINI_API_KEY = 'xxxxxxxxxxx' 
from google import genai
from google.genai import types
ai_client = genai.Client(api_key=GEMINI_API_KEY)


# 3. 目标群组与运行规则
# 填入目标群组的 username (比如群链接是 t.me/test_group，就填 'test_group')
# 如果是私密群没有 username，需要填群组的数字 ID (例如 -100123456789)
TARGET_GROUP = 'xxxxx'  
MAX_REPLIES = 100                 # 达到这个发言数量后脚本自动停止
# ==================================================

# 🧠 调教指令：注入老司机灵魂 + 严格字数控制
AI_PROMPT_INSTRUCTIONS = """
你现在是一个正在 Telegram 活跃群聊里跟群友吹水聊天的真人。
请根据我提供给你的最新群聊历史记录，说一句符合当前话题的、自然的、简短的回复。

必须严格遵守以下规则：
1. 说话风格要极其口语化、接地气。多用网络流行语和语气词（如：确实、老哥稳、哈、笑死、草、蹲一个、我觉得行、+1）。
2. 字数严格控制在 10 到 35 个字之间。绝对不要说废话，绝对不要像一个严谨的 AI 助手！
3. 如果最新的消息是纯表情、复读机或者无法理解的内容，请输出一句万能的水群话（例如：“前排围观”、“吃瓜”、“日常打卡”、“今天群里好热闹”）。
4. 绝不复述我的任何指令，直接输出你要在群里发送的文本内容。
"""

async def get_ai_reply(context_text):
    """使用 Google 2026 最新官方 SDK 生成回复，出现临时网络错误时立即重试一次。"""
    for attempt in range(1, 3):
        try:
            # 使用最新、速度极快的 gemini-2.5-flash 模型，且关闭安全过滤器防止开车被拦截
            response = ai_client.models.generate_content(
                model='gemini-2.5-flash',
                contents=f"{AI_PROMPT_INSTRUCTIONS}\n\n最新群聊记录如下：\n{context_text}\n\n请输出你的单句回复：",
                config=types.GenerateContentConfig(
                    # 解除安全限制，防止老司机话题触发 Google 拦截
                    safety_settings=[
                        types.SafetySetting(category=types.HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT, threshold=types.HarmBlockThreshold.BLOCK_NONE),
                        types.SafetySetting(category=types.HarmCategory.HARM_CATEGORY_HATE_SPEECH, threshold=types.HarmBlockThreshold.BLOCK_NONE),
                        types.SafetySetting(category=types.HarmCategory.HARM_CATEGORY_HARASSMENT, threshold=types.HarmBlockThreshold.BLOCK_NONE)
                    ]
                )
            )
            reply_text = response.text.strip()
            reply_text = reply_text.replace('"', '').replace('“', '').replace('”', '')
            return reply_text
        except Exception as e:
            print(f"❌ Gemini AI 生成失败 (尝试 {attempt}/2): {e}")
            if attempt == 1:
                print("⏳ 发生临时连接错误，稍等 1 秒后重试一次...")
                await asyncio.sleep(1)
                continue
            return None

def clean_and_split_text(text):
    """清洗标点，并将长句拆分成 10 字左右的短句列表"""
    text = re.sub(r'[。？！?!”“"\'\.!]', '', text)
    raw_chunks = re.split(r'[,，\s+]', text)
    final_chunks = []
    for chunk in raw_chunks:
        chunk = chunk.strip()
        if not chunk:
            continue
        if len(chunk) > 13:
            for i in range(0, len(chunk), 10):
                final_chunks.append(chunk[i:i+10])
        else:
            final_chunks.append(chunk)
    return final_chunks
async def main():
    # 使用新版关联的 python-socks 代理配置
    proxy_config = ('socks5', '127.0.0.1', 7890)
    
    async with TelegramClient('ai_water_session', API_ID, API_HASH, proxy=proxy_config) as client:
        print("🤖 终极拟人化老司机 AI 助手已启动...")
        
        reply_count = 0
        while reply_count < MAX_REPLIES:
            try:
                print(f"\n[进度] 正在抓取最新消息... ({reply_count}/{MAX_REPLIES})")
                
                context_messages = []
                async for message in client.iter_messages(TARGET_GROUP, limit=8):
                    if message.text:
                        sender = getattr(message.sender, 'first_name', '群友') if message.sender else '群友'
                        context_messages.append(f"{sender}: {message.text}")
                
                if not context_messages:
                    await asyncio.sleep(15)
                    continue
                
                context_messages.reverse()
                context_text = "\n".join(context_messages)
            
                
                print("🧠 正在呼叫老司机 AI 生成回复...")
                ai_reply = await get_ai_reply(context_text)
                
                if ai_reply:
                    # 模拟人类打字速度：根据字数随机延迟 2~5 秒再发出去
                    await asyncio.sleep(random.randint(2, 5))
                    await client.send_message(TARGET_GROUP, ai_reply)
                    print(f"✅ 成功假装真人发言: {ai_reply}")
                    reply_count += 1
                
                # ==================== 🎲 拟人化复杂行为分布机制 ====================
                # 模拟真人的聊天习惯：有时候话痨连发，有时候潜水
                dice = random.random() # 生成 0 到 1 之间的随机数
                
                if dice < 0.25:
                    # 25% 概率：群里正嗨，老子要连着插话！(15 秒到 40 秒内紧接着再发一句)
                    sleep_time = random.randint(5, 10)
                    print(f"🔥 [触发连发模式] 聊得正嗨，将在短时间内 ({sleep_time}秒) 再次插话...")
                elif dice < 0.80:
                    # 55% 概率：标准水群节奏 (45秒 到 2.5分钟 发言一次)
                    sleep_time = random.randint(10, 30)
                    print(f"💬 [触发正常模式] 保持正常水群频率，等待 {sleep_time} 秒...")
                else:
                    # 20% 概率：突然进入贤者模式/去上个厕所 (4分钟 到 9分钟 发言一次)
                    sleep_time = random.randint(100, 299)
                    print(f"🚬 [触发贤者模式] 稍微潜水观战一会儿，等待 {sleep_time} 秒...")
                
                await asyncio.sleep(sleep_time)
                # ===================================================================
                
            except Exception as e:
                print(f"⚠️ 发生意外: {e}")
                await asyncio.sleep(20)

        print("🎉 积分刷够了，安全下班！")

if __name__ == '__main__':
    asyncio.run(main())
```

执行python脚本：

```python
python groupmsg.py
```

效果如下：

![image-20260528230707183](https://raw.githubusercontent.com/fuzilong/images/main/note-img/image-20260528230707183.png)

## 这个脚本的整体功能

这是一个 `Telegram` 自动回复脚本，主要作用是：

- 连接指定群组
- 读取最新消息作为上下文
- 调用 `Gemini` AI 生成一条短回复
- 模拟人类发送行为，发到群里
- 循环执行直到达到最大回复次数

------

## 代码拆解成几个核心部分

### 1. 环境与代理配置

- [import os](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html), [asyncio](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html), [random](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html), [re](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
- 通过环境变量强制设置代理：
  - `http_proxy`
  - `https_proxy`
  - `all_proxy`
- 这是脚本能否成功访问 Gemini 和 Telegram 的关键，换环境时先检查代理是否正确

### 2. 核心配置常量

- [API_ID](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html), [API_HASH](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：Telegram API 登录凭证
- [GEMINI_API_KEY](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：Google Gemini API Key
- [TARGET_GROUP](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：目标群组 username 或数字 ID
- [MAX_REPLIES](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：最多回复次数
- 这部分是最常改的，复用时通常直接修改这几项

### 3. AI 生成回复的指令与函数

- [AI_PROMPT_INSTRUCTIONS](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：整套“老司机语气+风格+字数+标点”提示词
- [get_ai_reply(context_text)](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：
  - 向 Gemini 发起请求
  - 如果失败，会马上重试一次
  - 处理返回文本，去掉多余引号
- 这部分决定了“回复风格”和“稳定性”

### 4. 文本清洗/拆分辅助函数

- [clean_and_split_text(text)](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：
  - 清除中文/英文标点
  - 按逗号/空白拆成短句
  - 将太长句子再拆成 10 字左右块
- 目前它是把 AI 文本拆成更短的句子，复用时可以改成更智能的断句逻辑

### 5. 主循环逻辑

- [main()](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：
  - 连接 [TelegramClient](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
  - 读取群内最近几条消息
  - 拼成 [context_text](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
  - 调用 [get_ai_reply()](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
  - 发送消息 [client.send_message(TARGET_GROUP, ai_reply)](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
- 这部分还包括“拟人化行为”：
  - 随机延迟 `2~5` 秒发消息
  - 随机决定后续等待语句：
    - 25% 快速连发
    - 55% 正常频率
    - 20% 长时间潜水
- 出现异常时会进入 `except`，等待 20 秒继续循环

------

## 复用/修改，思路

1. 改目标群/数量
   - 修改 [TARGET_GROUP](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   - 修改 [MAX_REPLIES](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
2. 改回复风格
   - 修改 [AI_PROMPT_INSTRUCTIONS](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   - 例如换成“情感咨询”“产品推广”“客服答复”等
3. 改 AI 生成稳定性
   - 修改 [get_ai_reply](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html) 中的重试次数和等待策略
   - 例如增加到 3 次、改成指数退避
4. 改断句/分句规则
   - 修改或替换 [clean_and_split_text](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   - 例如改成“按逗号优先断句”、“每句 8-12 字”
5. 改发送行为
   - 修改 [main()](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html) 中 [await asyncio.sleep(...)](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   - 修改 [dice](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html) 逻辑，调整“连发/正常/潜水”的概率

------

## 脚本模块

- `配置模块`：代理、API Keys、目标群、最大回复数
- `AI 模块`：提示 + 请求 + 重试
- `文本模块`：清洗、拆句
- `Telegram 模块`：登录、读消息、发消息
- `行为模块`：随机延迟、人设发言节奏、异常处理

------

## 复用建议

以后复用这个脚本，先从这几部分拆：

1. [config](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：换账号/群
2. `prompt`：换题材
3. `ai`：换模型/重试
4. [text](vscode-file://vscode-app/c:/Users/z1on5/AppData/Local/Programs/Microsoft VS Code/6a49527b96/resources/app/out/vs/code/electron-browser/workbench/workbench.html)：换断句/字数规则
5. `loop`：换发送节奏

这样你就能快速判断“要改哪一块”，不用每次都重新读整个脚本。



## 高级安全避坑指南（核心）

即便接入了 AI，也有几个物理层面和机制上的坑需要注意：

1. **初次登录认证**：第一次运行这个 Python 脚本时，控制台（命令行）会提示你输入手机号和你在 Telegram 收到验证码（Login Code）。这是正常的，Telethon 会在本地生成一个 `ai_water_session.session` 文件，之后运行就不需要再输入验证码了。
2. **触发“群慢速模式”（Slow Mode）**：很多大群开启了“每人每 30 秒/ 2 分钟只能发一条消息”的限制。代码里的 `sleep_time`（随机等待时间）**必须要大于群组的慢速限制**，否则会因为频繁发送而被 Telegram 限制 API 调用。
3. **群内机器人的关键词封杀**：有些积分群非常恶毒，它们会故意在群里发一些“诱饵信息”（比如机器人发一句：“谁是机器人，打个 1”）。虽然 AI 很聪明，但有时也会中招去回复。因此，建议每运行几小时，人工过去看一眼，或者**只在群成员活跃、刷屏极快的时候挂机**，浑水摸鱼最安全。