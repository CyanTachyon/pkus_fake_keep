# pkus_fake_keep

生成仿 Keep 跑步记录截图的网页工具。模拟 Keep 户外跑步活动页面，可自定义各项参数并导出为图片。

## 功能

- 📱 高度还原 Keep 跑步记录页面 UI（状态栏、用户信息、运动数据、地图路线等）
- 🗺️ 在操场/跑道底图上随机生成逼真的跑步路线，带渐变配色与起终点标记
- ⚙️ 侧边配置面板：
  - 头像上传、用户名设置
  - 城市选择
  - 时间模式（今天 / 昨天 / 自定义日期）
  - 跑步距离范围、配速范围
- 🌤️ 根据日期自动获取真实天气数据
- 📸 一键导出截图（PNG）
- 💾 配置自动保存至 localStorage

## 技术栈

- [Vue 3](https://vuejs.org/)
- [Vite](https://vite.dev/)
- [html-to-image](https://github.com/nicokoenig/html-to-image) — 截图导出

## 快速开始

### 环境要求

- Node.js ^20.19.0 || ≥22.12.0

### 安装与运行

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev
```

浏览器打开提示的地址即可使用。

### 构建

```bash
npm run build
```

### 预览构建产物

```bash
npm run preview
```

## 使用方法

1. 在右侧配置面板中调整参数（头像、用户名、城市、时间、距离、配速等）
2. 页面左侧会实时渲染对应的 Keep 跑步记录页面
3. 点击「截图」按钮导出 PNG 图片

## 关于

- 作者：[CyanTachyon](https://www.tachyon.moe)
- 项目页面：[pkus-fake-keep](https://www.tachyon.moe/posts/pkus-fake-keep)
