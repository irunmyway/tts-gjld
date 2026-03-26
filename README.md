# ✨ tts-gjld · 二次元音色工坊

<p align="center">
  <img src="https://img.shields.io/badge/Vue-3.5-42b883?style=for-the-badge&logo=vuedotjs&logoColor=white" />
  <img src="https://img.shields.io/badge/Vite-8.0-646cff?style=for-the-badge&logo=vite&logoColor=white" />
  <img src="https://img.shields.io/badge/JavaScript-ESNext-f7df1e?style=for-the-badge&logo=javascript&logoColor=111" />
  <img src="https://img.shields.io/badge/SiliconFlow-TTS-7c3aed?style=for-the-badge&logo=soundcloud&logoColor=white" />
</p>

<p align="center">
  <img src="https://img.shields.io/github/stars/irunmyway/tts-gjld?style=flat-square" />
  <img src="https://img.shields.io/github/forks/irunmyway/tts-gjld?style=flat-square" />
  <img src="https://img.shields.io/github/last-commit/irunmyway/tts-gjld?style=flat-square" />
  <img src="https://img.shields.io/github/license/irunmyway/tts-gjld?style=flat-square" />
</p>

> 对接 SiliconFlow 音频接口的 Vue3 工具站。
> 支持 **音色上传 / 音色管理 / 一键试听合成 / 共享音色广场**，走一个酷炫、直给、能打的 TTS 工作流。

---

## 🌌 项目亮点

- 🎤 自定义音色上传：本地音频 + 参考文本，一键生成可复用 URI
- 🎛 音色管理面板：刷新、复制、试听、删除，全流程可视化
- 🧪 试听合成弹窗：支持 `voice` / `references` 双模式调试
- 🏪 共享音色广场：内置音色素材，上传后可直接进入试听
- 💰 余额查看：快速确认当前 API 可用额度
- 🖼 随机背景切换：轻量二次元氛围感 UI

---

## 🧩 技术栈小卡片

| 分类 | 技术 |
|---|---|
| 前端框架 | ![Vue](https://img.shields.io/badge/Vue_3-42b883?logo=vuedotjs&logoColor=white) |
| 构建工具 | ![Vite](https://img.shields.io/badge/Vite-646cff?logo=vite&logoColor=white) |
| 语言 | ![JavaScript](https://img.shields.io/badge/JavaScript-ESNext-f7df1e?logo=javascript&logoColor=111) |
| API | ![SiliconFlow](https://img.shields.io/badge/SiliconFlow-Audio_API-7c3aed) |
| 运行环境 | ![Node](https://img.shields.io/badge/Node.js-18%2B-339933?logo=nodedotjs&logoColor=white) |

---

## 📈 趋势图（A+B 组合）

### 1) GitHub 关注趋势（真实数据）

[![Star History Chart](https://api.star-history.com/image?repos=irunmyway%2tts-gjld&type=date&legend=top-left)](https://www.star-history.com/?repos=runmyway%2tts-gjld&type=date&legend=top-left)

### 2) 功能演进趋势（项目路线）

```mermaid
timeline
    title tts-gjld 功能演进
    2026-Q1 : 基础页面搭建
            : API Key 配置
            : 音色上传/列表管理
    2026-Q2 : 试听合成弹窗增强
            : references JSON 调试
            : 共享音色广场
    2026-Q3 : 交互细节优化
            : 错误提示与可用性提升
```

---

## 🚀 快速开始

### 1. 安装依赖

```bash
npm install
```

### 2. 本地开发

```bash
npm run dev
```

### 3. 构建发布

```bash
npm run build
npm run preview
```

---

## ⚙️ 使用流程

1. 打开页面后先填写并保存 SiliconFlow API Key（`sk-` 开头）
2. 上传本地参考音频，填写音色名与参考文本
3. 在音色管理表中复制 URI 或直接点「试听」
4. 在试听弹窗中输入文本，设置格式/采样率/速度/增益并合成
5. 也可进入共享音色广场，一键上传并带入试听

---

## 🔌 关键接口（SiliconFlow）

- `POST /uploads/audio/voice`：上传音色
- `GET /audio/voice/list`：获取音色列表
- `POST /audio/speech`：语音合成
- `POST /audio/voice/deletions`：删除音色
- `GET /user/info`：余额查询

参考文档：`https://docs.siliconflow.cn/cn/api-reference/audio`

---

## 📁 项目结构

```text
tts-gjld/
├─ src/
│  ├─ App.vue
│  ├─ main.js
│  └─ components/
│     └─ SpeechDebugPanel.vue
├─ public/
├─ package.json
└─ README.md
```

---

## 🗺 Roadmap

- [ ] 增加更多预置音色模板与标签筛选
- [ ] 增加试听历史记录与一键复用
- [ ] 增加移动端交互细节适配
- [ ] 增加批量管理能力（批量复制/删除）

---

## 🌟 Star 支持

如果这个项目对你有帮助，欢迎点个 Star：

👉 https://github.com/irunmyway/tts-gjld
