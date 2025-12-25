灵感盲盒 (Inspiration Gacha) 架构规划

1. 目录结构 (Directory Structure)

建议采用功能模块化的目录组织方式，方便后期扩展复杂的算法逻辑。

src/
├── assets/             # 静态资源 (图片、图标、Sfx音效)
├── components/         # 公用 UI 组件 (Atomic Design)
│   ├── ui/             # 基础组件 (Button, Modal, Toast)
│   ├── gacha/          # 抽卡相关 (CardDisplay, GachaAnimation)
│   └── editor/         # 编辑与展示 (PromptDisplay, ImageCanvas)
├── config/             # 全局配置 (API Endpoints, Gacha Rates)
├── data/               # 静态卡池数据 (核心库)
│   ├── subjects.ts     # 主体词库
│   ├── styles.ts       # 风格词库
│   ├── poses.ts        # 姿态/ControlNet 库 (新)
│   └── loras.ts        # 专用 LoRA 库 (新)
├── hooks/              # 自定义 Hook (业务逻辑抽离)
│   ├── useGacha.ts     # 抽卡核心逻辑
│   ├── useGemini.ts    # LLM 润色逻辑
│   └── useImageGen.ts  # 图像生成接口 (支持 ControlNet/LoRA 参数)
├── services/           # API 请求封装
│   ├── gemini.ts       # Gemini API 调用
│   └── imagen.ts       # 图像生成 API 调用
├── store/              # 状态管理 (Zustand)
│   ├── useCardStore.ts # 已抽中的卡片、当前组合、插槽锁定状态
│   └── useUserStore.ts # 用户偏好、生成历史
├── types/              # TypeScript 类型定义
│   └── index.d.ts      # Card, Prompt, User 等类型
├── utils/              # 工具函数 (随机算法、Prompt 拼接、LoRA 解析)
└── App.tsx             # 路由与入口


2. 核心模块设计 (Core Modules)

A. 数据驱动的抽卡逻辑 (Data-Driven Gacha)

卡片不再只是一个单词，它是一个带有参数的“指令包”。

// types/index.d.ts
export interface Card {
  id: string;
  type: 'subject' | 'style' | 'pose' | 'background' | 'lora';
  label: string; // 中文显示 (如: "赛博歌姬")
  value: string; // 基础提示词 (如: "cyberpunk idol")
  rarity: 'N' | 'R' | 'SR' | 'SSR';
  // 高级扩展参数
  params?: {
    lora_id?: string;      // 对应的 LoRA 模型标识
    lora_weight?: number;  // 权重
    control_net?: {        // ControlNet 相关
      type: 'openpose' | 'canny' | 'depth';
      image_url?: string;  // 参考图
    };
    negative_prompt?: string; // 某些卡片自带的反向提示词
  };
}


B. LLM 润色层 (The Polisher Layer)

逻辑分发：LLM 只负责润色 label 和 value 部分的自然语言描述。

参数合并：在最终提交生成前，useGemini 会将 LLM 生成的美化文案与卡片中的 params（如 <lora:xxx:0.8>）进行程序化拼接。

C. 扩展性协议 (Extension Protocol)

为了支持“后期添加背景/姿态控制”，我们设计了以下机制：

插槽锁定 (Slot Locking)：

在 UI 上每个分类（主体、风格、背景）都有一个“锁”。

当用户锁定“背景”槽位时，点击“十连抽”只会刷新未锁定的槽位，从而实现背景不变、更换人物。

LoRA 叠加态：

除了主卡槽，可以额外设计一个“附件槽”。

抽中的 LoRA 卡片会显示在侧边栏，用户可以手动调整每个 LoRA 的权重进度条。

姿态参考 (ControlNet)：

当 type: 'pose' 的卡片被激活时，它会向生成引擎传递一个预设的 pose_id。

后端接口会根据这个 ID 加载对应的 OpenPose 骨架图。

3. 技术栈建议 (Tech Stack)

构建工具: Vite

状态管理: Zustand (管理卡片插槽的锁定状态非常方便)

动画: Framer Motion (实现卡片飞入、旋转、发光效果)

通信: Axios (支持复杂的请求参数拼接)

4. 开发计划升级

MVP 阶段：实现基础的分类抽卡和 LLM 简单润色。

视觉增强：加入卡片稀有度特效和插槽锁定 UI。

高级控制 (本期重点讨论)：

引入 loras.ts 数据源。

在生成预览界面增加“参数面板”，显示当前生效的 LoRA 和 ControlNet 参数。