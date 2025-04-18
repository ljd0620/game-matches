# Game Matches

Game Matches 是一个基于 Vue 3 和 Vite 构建的休闲消除类游戏。玩家通过拖动方块进行匹配，获得分数并挑战排行榜。

## 功能特性

- **消除逻辑**：支持匹配 3 个或更多相同颜色的方块进行消除。
- **特殊方块**：
  - 💣 炸弹：清除周围的方块。
  - 🌈 彩虹：清除所有相同颜色的方块。
- **计时模式**：倒计时 60 秒，挑战高分。
- **分数排行榜**：记录玩家的游戏时间和分数，支持本地存储。
- **响应式设计**：适配不同屏幕尺寸，支持触控操作。

## 技术栈

- **前端框架**：Vue 3
- **构建工具**：Vite
- **状态管理**：Vue Composition API
- **样式**：Scoped CSS

## 项目结构

```plaintext
e:\VSCodeProjects\game-matches
├── public/               # 静态资源
├── src/
│   ├── assets/           # 项目资源（图片、字体等）
│   ├── components/       # Vue 组件
│   │   └── GameBoard.vue # 游戏主界面
│   ├── App.vue           # 根组件
│   ├── main.js           # 项目入口文件
│   └── styles/           # 全局样式
├── [README.md](http://_vscodecontentref_/1)             # 项目说明文件
├── [package.json](http://_vscodecontentref_/2)          # 项目依赖配置
└── [vite.config.js](http://_vscodecontentref_/3)        # Vite 配置文件
```

## 安装与运行
1. 克隆仓库：
   ```bash
   git clone URL_ADDRESS   git clone https://github.com/yourusername/game-matches.git
   cd game-matches
   ```
2. 安装依赖：
   ```bash
   npm install
   ```
3. 启动开发服务器：
   ```bash
   npm run dev
   ```
4. 构建生产版本：
   ```bash
   npm run build
   ```
5. 本地预览生成版本：
   ```bash
   npm run preview
   ```
## 游戏规则
1. 拖动方块，匹配 3 个或更多相同颜色的方块即可消除。
2. 特殊方块：
    - 炸弹 (💣)：清除周围 8 个方块。
    - 彩虹 (🌈)：清除所有相同颜色的方块。
3. 倒计时结束后游戏结束，分数将记录到排行榜。