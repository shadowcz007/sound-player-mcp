# 🎵 sound-player-mcp

## 📖 项目概述

### 项目名称
NodeSoundPlayer

### 功能简介
NodeSoundPlayer 是一个声音播放系统，允许用户管理声音分类并播放分类中的预设声音（包括音频文件和代码生成的合成声音）。通过 MCP 框架，用户可以通过工具接口获取分类信息或触发分类声音播放。

### ✨ 主要特性
- 📁 **分类管理**：获取所有声音分类的列表
- 🎵 **声音播放**：播放指定分类中的所有声音，支持音频文件和代码生成的声音
- 🔌 **MCP 集成**：通过 MCP 工具接口，提供标准化的工具调用方式，方便与其他系统集成
- 🚀 **可扩展性**：支持添加新的声音类型和分类

---

## 💻 项目结构

```
NodeSoundPlayer/
├── sounds/              # 存放音频文件的目录
│   ├── category1/
│   │   ├── sound1.mp3
│   │   └── sound2.wav
│   └── Adopted/
│       └── sound3.mp3
├── config.json          # 分类和声音的配置文件
├── index.js            # 主程序入口（MCP 服务器）
├── tools/              # 工具模块
│   ├── getCategories.js # 获取分类信息的工具
│   └── playCategory.js  # 播放分类的工具
└── package.json        # 项目依赖和脚本
```

### 📝 配置文件示例 (config.json)

```json
{
  "categories": {
    "category1": [
      { "type": "file", "path": "sounds/category1/sound1.mp3" },
      { "type": "generated", "wave": "sine", "frequency": 440, "duration": 2 }
    ],
    "category2": [
      { "type": "file", "path": "sounds/category2/sound3.mp3" }
    ]
  }
}
```

---

## 🛠️ MCP 工具实现

### 1️⃣ GetCategories 工具

**工具名称**：`getCategories`  
**功能**：返回系统中所有声音分类的列表  
**输入参数**：无（空对象）  
**输出**：一个包含分类名称的数组

```javascript
import { z } from "zod";
import { promises as fs } from "fs";

// 工具输入 schema（无输入参数）
const getCategoriesInput = z.object({});

// 工具实现
export const getCategories = {
  name: "getCategories",
  schema: getCategoriesInput,
  handler: async () => {
    try {
      const config = JSON.parse(await fs.readFile("config.json", "utf8"));
      const categories = Object.keys(config.categories);
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(categories),
          },
        ],
      };
    } catch (error) {
      return {
        content: [
          {
            type: "text",
            text: `Error fetching categories: \${error.message}`,
          },
        ],
      };
    }
  },
};
```

### 2️⃣ PlayCategory 工具

**工具名称**：`playCategory`  
**功能**：播放指定分类中的所有声音  
**输入参数**：
- `category`：分类名称（字符串，必需）
- `loop`：是否循环播放（布尔值，可选，默认为 false）
- `interval`：声音之间的播放间隔（秒，数字，可选，默认为 0）

```javascript
import { z } from "zod";
import { promises as fs } from "fs";
import playSound from "play-sound";
import { Speaker } from "speaker";
import Generator from "audio-generator";

// 工具输入 schema
const playCategoryInput = z.object({
  category: z.string().min(1, "Category name is required"),
  loop: z.boolean().optional().default(false),
  interval: z.number().min(0).optional().default(0),
});

// 播放单个声音的辅助函数
async function playSingleSound(sound) {
  return new Promise((resolve) => {
    if (sound.type === "file") {
      playSound().play(sound.path, () => resolve());
    } else if (sound.type === "generated") {
      const generator = Generator({ frequency: sound.frequency });
      const speaker = new Speaker();
      generator.pipe(speaker);
      setTimeout(resolve, sound.duration * 1000);
    }
  });
}

// 工具实现
export const playCategory = {
  name: "playCategory",
  schema: playCategoryInput,
  handler: async ({ category, loop, interval }) => {
    try {
      const config = JSON.parse(await fs.readFile("config.json", "utf8"));
      const sounds = config.categories[category];

      if (!sounds) {
        return {
          content: [{ type: "text", text: `Category "\${category}" not found` }],
        };
      }

      do {
        for (const sound of sounds) {
          await playSingleSound(sound);
          if (interval > 0) {
            await new Promise((resolve) => setTimeout(resolve, interval * 1000));
          }
        }
      } while (loop);

      return {
        content: [
          { type: "text", text: `Played all sounds in category "\${category}"` }
        ],
      };
    } catch (error) {
      return {
        content: [
          { type: "text", text: `Error playing category "\${category}": \${error.message}` }
        ],
      };
    }
  },
};
```

---

## 🚀 使用方式

### 获取分类信息

**示例请求**：
```json
{
  "tool": "getCategories",
  "input": {}
}
```

**示例响应**：
```json
{
  "content": [
    {
      "type": "text",
      "text": "[\"category1\",\"category2\"]"
    }
  ]
}
```

### 播放分类

**示例请求**：
```json
{
  "tool": "playCategory",
  "input": {
    "category": "category1",
    "loop": false,
    "interval": 1
  }
}
```

**示例响应**：
```json
{
  "content": [
    {
      "type": "text",
      "text": "Played all sounds in category \"category1\""
    }
  ]
}
```

---

## 🔧 实现细节

- ✅ **输入验证**：使用 zod 确保工具输入参数的正确性
- 🛡️ **错误处理**：工具包含错误处理逻辑，返回清晰的错误信息
- 🎵 **声音播放**：
  - 音频文件通过 play-sound 播放，支持 MP3、WAV 等格式
  - 合成声音通过 audio-generator 和 speaker 生成并播放
- ⚡ **异步处理**：使用 Promise 和 async/await 确保声音按顺序播放
- 🔌 **MCP 兼容性**：工具输出符合 MCP 协议的 content 格式

## 📦 依赖配置 (package.json)

```json
{
  "name": "NodeSoundPlayer",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.22.4",
    "play-sound": "^1.1.5",
    "speaker": "^0.5.3",
    "audio-generator": "^2.0.0"
  },
  "scripts": {
    "start": "node index.js"
  }
}
```

