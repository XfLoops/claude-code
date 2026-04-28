# UI 系统 (Ink + React)

## 文件位置

- `src/ink/` - 自定义 Ink 框架
- `src/components/` - React 组件
- `src/screens/` - 屏幕组件
- `src/ink.ts` - Ink 渲染包装

## 1. UI 系统概述

Claude Code 使用 React + Ink 构建终端 UI，这是一个独特的架构选择，将 Web 开发范式引入 CLI 应用。

### 1.1 架构对比

```
传统 CLI UI              React + Ink UI
─────────────────────────────────────────────────
printf()                 React 组件
手动状态管理             React State
手工布局                 Flexbox (Yoga)
手动事件处理             React Hooks
难以测试                 Jest/React Testing Library
                         
优势：声明式、组件化、    优势：熟悉的开发体验、
      易于测试、生态丰富        复用 Web 技能
```

### 1.2 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      UI 系统层级                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   应用层 (App Layer)                    │ │
│  │  App.tsx                                               │ │
│  │  ├── ThemeProvider (主题)                              │ │
│  │  ├── AppStateProvider (全局状态)                       │ │
│  │  └── ErrorBoundary (错误边界)                          │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   屏幕层 (Screen Layer)                 │ │
│  │  REPL.tsx                                              │ │
│  │  ├── Messages (消息列表)                               │ │
│  │  ├── PromptInput (输入框)                              │ │
│  │  └── StatusBar (状态栏)                                │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   组件层 (Component Layer)              │ │
│  │  Messages.tsx, MessageRow.tsx                          │ │
│  │  PromptInput.tsx, Spinner.tsx                          │ │
│  │  PermissionDialog.tsx, TrustDialog.tsx                 │ │
│  │  StructuredDiff, CodeBlock                             │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   Ink 框架层 (Ink Framework)            │ │
│  │  reconciler.ts (React reconciler)                      │ │
│  │  yoga.ts (CSS Flexbox 布局)                            │ │
│  │  renderer.ts (ANSI 输出)                               │ │
│  │  events.ts (键盘/焦点事件)                             │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 技术栈

| 技术 | 用途 | 版本 |
|------|------|------|
| React | UI 框架 | 19.x |
| Ink | 终端 React 渲染器 | 5.x |
| Yoga | Flexbox 布局引擎 | 3.x |
| Zustand-style | 状态管理 | 自定义 |
| highlight.js | 语法高亮 | 11.x |

## 2. 自定义 Ink 框架

### 2.1 为什么自定义？

原生的 Ink 框架在某些场景下不够灵活，Claude Code 实现了自定义的 Ink 核心以满足特定需求：

1. **更精细的控制**: 自定义渲染逻辑
2. **性能优化**: 针对 CLI 场景优化
3. **特殊功能**: 如搜索高亮、虚拟列表
4. **与现有代码集成**: 更好的错误处理

### 2.2 核心架构

**目录**: `src/ink/`

```
src/ink/
├── src/                      # 核心实现
│   ├── reconciler.ts         # React reconciler
│   ├── renderer.ts           # 渲染器
│   ├── instance.ts           # 实例管理
│   └── output.ts             # ANSI 输出
├── components/               # Ink 组件
│   ├── Box.tsx              # 容器组件
│   ├── Text.tsx             # 文本组件
│   ├── Spacer.tsx           # 间隔组件
│   └── Newline.tsx          # 换行
├── hooks/                    # 自定义 hooks
│   ├── useInput.ts          # 键盘输入
│   ├── useTerminalSize.ts   # 终端尺寸
│   ├── useFocus.ts          # 焦点管理
│   └── useSearchHighlight.ts # 搜索高亮
├── events/                   # 事件系统
│   ├── emitter.ts           # 事件发射器
│   ├── keyboard.ts          # 键盘事件
│   └── focus.ts             # 焦点事件
├── layout/                   # 布局系统
│   └── yoga.ts              # Yoga 集成
└── termio/                   # 终端 I/O
    ├── stdout.ts            # 输出处理
    └── stdin.ts             # 输入处理
```

### 2.3 React Reconciler

**文件**: `src/ink/src/reconciler.ts`

```typescript
import Reconciler from 'react-reconciler';

/**
 * 自定义 React Reconciler
 * 
 * 将 React 组件树渲染到终端
 */
const InkReconciler = Reconciler({
  // ==================== 创建实例 ====================
  
  createInstance(
    type: string,
    props: Props,
    rootContainer: Root,
    hostContext: HostContext,
    internalHandle: OpaqueHandle
  ): InkElement {
    switch (type) {
      case 'box':
        return createBoxElement(props);
      case 'text':
        return createTextElement(props);
      default:
        throw new Error(`Unknown element type: ${type}`);
    }
  },
  
  createTextInstance(
    text: string,
    rootContainer: Root,
    hostContext: HostContext,
    internalHandle: OpaqueHandle
  ): InkTextElement {
    return createTextElement({ children: text });
  },
  
  // ==================== 更新实例 ====================
  
  prepareUpdate(
    instance: InkElement,
    type: string,
    oldProps: Props,
    newProps: Props,
    rootContainer: Root,
    hostContext: HostContext
  ): UpdatePayload | null {
    // 计算需要更新的属性
    const updatePayload: UpdatePayload = {};
    
    for (const key in newProps) {
      if (oldProps[key] !== newProps[key]) {
        updatePayload[key] = newProps[key];
      }
    }
    
    return Object.keys(updatePayload).length > 0 ? updatePayload : null;
  },
  
  commitUpdate(
    instance: InkElement,
    updatePayload: UpdatePayload,
    type: string,
    oldProps: Props,
    newProps: Props,
    internalHandle: OpaqueHandle
  ): void {
    // 应用更新
    Object.assign(instance.props, updatePayload);
    instance.markDirty();
  },
  
  // ==================== 树操作 ====================
  
  appendChild(parent: InkElement, child: InkElement): void {
    parent.children.push(child);
    child.parent = parent;
    parent.markDirty();
  },
  
  removeChild(parent: InkElement, child: InkElement): void {
    const index = parent.children.indexOf(child);
    if (index !== -1) {
      parent.children.splice(index, 1);
      child.parent = null;
      parent.markDirty();
    }
  },
  
  insertBefore(
    parent: InkElement,
    child: InkElement,
    beforeChild: InkElement
  ): void {
    const index = parent.children.indexOf(beforeChild);
    if (index !== -1) {
      parent.children.splice(index, 0, child);
      child.parent = parent;
      parent.markDirty();
    }
  },
  
  // ==================== 渲染控制 ====================
  
  scheduleTimeout: setTimeout,
  cancelTimeout: clearTimeout,
  noTimeout: -1,
  
  isPrimaryRenderer: true,
  warnsIfNotActing: true,
  
  supportsMutation: true,
  supportsPersistence: false,
  supportsHydration: false,
  
  // ==================== 其他必要方法 ====================
  
  getPublicInstance(instance: InkElement): InkElement {
    return instance;
  },
  
  getRootHostContext(): HostContext {
    return {};
  },
  
  getChildHostContext(
    parentHostContext: HostContext,
    type: string,
    rootContainer: Root
  ): HostContext {
    return parentHostContext;
  },
  
  shouldSetTextContent(type: string, props: Props): boolean {
    return false;
  },
  
  resetTextContent(instance: InkElement): void {
    // 重置文本内容
  },
  
  finalizeInitialChildren(): boolean {
    return false;
  },
  
  prepareForCommit(): void {
    // 提交前的准备
  },
  
  resetAfterCommit(): void {
    // 提交后的重置
  },
  
  clearContainer(container: Root): void {
    container.root.children = [];
  },
});

export { InkReconciler };
```

### 2.4 Yoga 布局集成

**文件**: `src/ink/layout/yoga.ts`

```typescript
import Yoga, { Node } from 'yoga-layout';

/**
 * Yoga 布局集成
 * 
 * 使用 Facebook 的 Yoga 引擎实现 CSS Flexbox 布局
 */
export function createYogaNode(element: InkElement): Node {
  const node = Yoga.Node.create();
  
  // 应用样式属性
  if (element.props.style) {
    const style = element.props.style;
    
    // Flex 方向
    if (style.flexDirection) {
      node.setFlexDirection(
        style.flexDirection === 'row' 
          ? Yoga.FLEX_DIRECTION_ROW 
          : Yoga.FLEX_DIRECTION_COLUMN
      );
    }
    
    // Justify 内容
    if (style.justifyContent) {
      node.setJustifyContent(parseJustifyContent(style.justifyContent));
    }
    
    // Align 项目
    if (style.alignItems) {
      node.setAlignItems(parseAlignItems(style.alignItems));
    }
    
    // Flex
    if (style.flex !== undefined) {
      node.setFlex(style.flex);
    }
    
    // 尺寸
    if (style.width !== undefined) {
      node.setWidth(style.width);
    }
    if (style.height !== undefined) {
      node.setHeight(style.height);
    }
    
    // Margin
    if (style.marginTop !== undefined) {
      node.setMargin(Yoga.EDGE_TOP, style.marginTop);
    }
    if (style.marginBottom !== undefined) {
      node.setMargin(Yoga.EDGE_BOTTOM, style.marginBottom);
    }
    if (style.marginLeft !== undefined) {
      node.setMargin(Yoga.EDGE_LEFT, style.marginLeft);
    }
    if (style.marginRight !== undefined) {
      node.setMargin(Yoga.EDGE_RIGHT, style.marginRight);
    }
    
    // Padding
    if (style.paddingTop !== undefined) {
      node.setPadding(Yoga.EDGE_TOP, style.paddingTop);
    }
    if (style.paddingBottom !== undefined) {
      node.setPadding(Yoga.EDGE_BOTTOM, style.paddingBottom);
    }
    if (style.paddingLeft !== undefined) {
      node.setPadding(Yoga.EDGE_LEFT, style.paddingLeft);
    }
    if (style.paddingRight !== undefined) {
      node.setPadding(Yoga.EDGE_RIGHT, style.paddingRight);
    }
  }
  
  return node;
}

/**
 * 计算布局
 */
export function calculateLayout(
  rootNode: Node,
  width: number,
  height: number
): void {
  rootNode.calculateLayout(width, height, Yoga.DIRECTION_LTR);
}

/**
 * 获取元素布局
 */
export function getLayout(node: Node): Layout {
  return {
    left: node.getComputedLeft(),
    top: node.getComputedTop(),
    width: node.getComputedWidth(),
    height: node.getComputedHeight(),
  };
}
```

### 2.5 ANSI 输出

**文件**: `src/ink/src/output.ts`

```typescript
import ansiEscapes from 'ansi-escapes';

/**
 * ANSI 输出生成
 * 
 * 将 Yoga 布局结果转换为 ANSI 转义序列
 */
export class Output {
  private lines: string[] = [];
  private height: number;
  private width: number;
  
  constructor(width: number, height: number) {
    this.width = width;
    this.height = height;
    this.clear();
  }
  
  /**
   * 在指定位置写入文本
   */
  write(x: number, y: number, text: string, style?: Style): void {
    if (y < 0 || y >= this.height) return;
    
    const line = this.lines[y] || '';
    const before = line.slice(0, x);
    const after = line.slice(x + text.length);
    
    // 应用样式
    const styledText = this.applyStyle(text, style);
    
    // 填充到行宽
    const padded = before + styledText + after;
    this.lines[y] = padded.slice(0, this.width).padEnd(this.width, ' ');
  }
  
  /**
   * 应用 ANSI 样式
   */
  private applyStyle(text: string, style?: Style): string {
    if (!style) return text;
    
    let result = text;
    
    if (style.bold) {
      result = `\u001b[1m${result}\u001b[22m`;
    }
    if (style.italic) {
      result = `\u001b[3m${result}\u001b[23m`;
    }
    if (style.underline) {
      result = `\u001b[4m${result}\u001b[24m`;
    }
    if (style.color) {
      result = `\u001b[38;5;${this.colorToAnsi(style.color)}m${result}\u001b[39m`;
    }
    if (style.backgroundColor) {
      result = `\u001b[48;5;${this.colorToAnsi(style.backgroundColor)}m${result}\u001b[49m`;
    }
    if (style.dim) {
      result = `\u001b[2m${result}\u001b[22m`;
    }
    
    return result;
  }
  
  /**
   * 颜色转 ANSI 代码
   */
  private colorToAnsi(color: string): number {
    // 支持的颜色名称映射
    const colors: Record<string, number> = {
      black: 0,
      red: 1,
      green: 2,
      yellow: 3,
      blue: 4,
      magenta: 5,
      cyan: 6,
      white: 7,
      gray: 8,
    };
    
    return colors[color] ?? 7;
  }
  
  /**
   * 生成 ANSI 输出
   */
  toString(): string {
    return this.lines.join('\n');
  }
  
  /**
   * 清空输出
   */
  clear(): void {
    this.lines = Array(this.height).fill(''.padEnd(this.width, ' '));
  }
  
  /**
   * 获取用于清屏的 ANSI 序列
   */
  getClearSequence(): string {
    return ansiEscapes.clearTerminal;
  }
}
```

## 3. 自定义 Hooks

### 3.1 useInput - 键盘输入处理

**文件**: `src/ink/hooks/useInput.ts`

```typescript
import { useEffect } from 'react';
import { useInkContext } from './useInkContext.js';

/**
 * useInput Hook
 * 
 * 处理键盘输入事件
 */
export function useInput(
  handler: (input: string, key: Key) => void,
  options?: InputOptions
): void {
  const { stdin, isRawModeSupported } = useInkContext();
  
  useEffect(() => {
    if (!isRawModeSupported) {
      return;
    }
    
    // 启用原始模式（获取每个按键）
    stdin.setRawMode(true);
    stdin.resume();
    stdin.setEncoding('utf8');
    
    const handleData = (data: Buffer) => {
      const input = data.toString();
      const key = parseKey(input);
      
      // 处理特殊键
      if (key.ctrl && input === '\u0003') {
        // Ctrl+C - 退出
        process.exit(0);
      }
      
      if (key.escape && !options?.disableExitOnEscape) {
        // Escape - 退出
        process.exit(0);
      }
      
      // 调用处理器
      handler(input, key);
    };
    
    stdin.on('data', handleData);
    
    return () => {
      stdin.off('data', handleData);
      stdin.setRawMode(false);
      stdin.pause();
    };
  }, [handler, stdin, isRawModeSupported]);
}

// 按键解析
function parseKey(input: string): Key {
  const key: Key = {
    ctrl: false,
    meta: false,
    shift: false,
    escape: false,
    return: false,
    tab: false,
    backspace: false,
    delete: false,
    up: false,
    down: false,
    left: false,
    right: false,
  };
  
  // Ctrl 键
  if (input.length === 1 && input.charCodeAt(0) <= 31 && input.charCodeAt(0) >= 1) {
    key.ctrl = true;
  }
  
  // Escape 序列
  if (input.startsWith('\u001b')) {
    key.escape = true;
    
    // 方向键
    if (input === '\u001b[A') key.up = true;
    if (input === '\u001b[B') key.down = true;
    if (input === '\u001b[C') key.right = true;
    if (input === '\u001b[D') key.left = true;
  }
  
  // 特殊键
  if (input === '\r' || input === '\n') key.return = true;
  if (input === '\t') key.tab = true;
  if (input === '\b' || input === '\x7f') key.backspace = true;
  if (input === '\u001b[3~') key.delete = true;
  
  return key;
}

type Key = {
  ctrl: boolean;
  meta: boolean;
  shift: boolean;
  escape: boolean;
  return: boolean;
  tab: boolean;
  backspace: boolean;
  delete: boolean;
  up: boolean;
  down: boolean;
  left: boolean;
  right: boolean;
};

type InputOptions = {
  disableExitOnEscape?: boolean;
};
```

### 3.2 useTerminalSize - 终端尺寸响应

```typescript
import { useState, useEffect } from 'react';
import { useInkContext } from './useInkContext.js';

/**
 * useTerminalSize Hook
 * 
 * 响应终端尺寸变化
 */
export function useTerminalSize(): TerminalSize {
  const { stdout } = useInkContext();
  const [size, setSize] = useState<TerminalSize>({
    columns: stdout.columns || 80,
    rows: stdout.rows || 24,
  });
  
  useEffect(() => {
    const handleResize = () => {
      setSize({
        columns: stdout.columns || 80,
        rows: stdout.rows || 24,
      });
    };
    
    stdout.on('resize', handleResize);
    
    return () => {
      stdout.off('resize', handleResize);
    };
  }, [stdout]);
  
  return size;
}

type TerminalSize = {
  columns: number;
  rows: number;
};
```

### 3.3 useSearchHighlight - 搜索高亮

```typescript
import { useMemo } from 'react';
import { Text } from 'ink';

/**
 * useSearchHighlight Hook
 * 
 * 在文本中高亮搜索关键词
 */
export function useSearchHighlight(
  text: string,
  query: string,
  options?: HighlightOptions
): React.ReactNode {
  return useMemo(() => {
    if (!query) {
      return <Text>{text}</Text>;
    }
    
    const parts = text.split(new RegExp(`(${escapeRegExp(query)})`, 'gi'));
    
    return (
      <Text>
        {parts.map((part, i) => {
          const isMatch = part.toLowerCase() === query.toLowerCase();
          return isMatch ? (
            <Text key={i} backgroundColor="yellow" color="black">
              {part}
            </Text>
          ) : (
            <Text key={i}>{part}</Text>
          );
        })}
      </Text>
    );
  }, [text, query]);
}

function escapeRegExp(string: string): string {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

type HighlightOptions = {
  caseSensitive?: boolean;
};
```

## 4. 核心组件

### 4.1 REPL 屏幕

**文件**: `src/screens/REPL.tsx`

```typescript
import React, { useState, useCallback, useRef } from 'react';
import { Box, useInput, useAppState } from 'ink';
import { Messages } from '../components/Messages.js';
import { PromptInput } from '../components/PromptInput/index.js';
import { StatusBar } from '../components/StatusBar.js';
import { useQueryEngine } from '../hooks/useQueryEngine.js';

/**
 * REPL 屏幕
 * 
 * 主交互界面，包含：
 * - 消息列表（历史对话）
 * - 输入框
 * - 状态栏
 */
export function REPL(): React.ReactElement {
  // 状态管理
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);
  
  // 全局状态
  const appState = useAppState();
  const queryEngine = useQueryEngine();
  
  // 键盘处理
  useInput((value, key) => {
    if (key.return && input.trim()) {
      handleSubmit(input);
    }
  });
  
  // 提交消息
  const handleSubmit = useCallback(async (text: string) => {
    setIsLoading(true);
    setInput('');
    
    // 添加用户消息到历史
    appState.addMessage({
      role: 'user',
      content: text,
    });
    
    // 发送消息
    const stream = queryEngine.submitMessage(text);
    
    // 处理流式响应
    for await (const event of stream) {
      switch (event.type) {
        case 'content':
          // 流式内容更新
          appState.appendToLastMessage(event.content.text);
          break;
          
        case 'tool_use':
          // 显示工具调用
          appState.addMessage({
            role: 'assistant',
            type: 'tool_use',
            tool: event.tool,
          });
          break;
          
        case 'tool_result':
          // 显示工具结果
          appState.addMessage({
            role: 'user',
            type: 'tool_result',
            result: event.result,
          });
          break;
          
        case 'message_complete':
          // 消息完成
          appState.finalizeLastMessage();
          break;
      }
    }
    
    setIsLoading(false);
  }, [appState, queryEngine]);
  
  // 处理斜杠命令
  const handleSlashCommand = useCallback((command: string) => {
    switch (command) {
      case '/clear':
        appState.clearMessages();
        break;
      case '/compact':
        appState.compactMessages();
        break;
      case '/help':
        appState.addMessage({
          role: 'system',
          content: getHelpText(),
        });
        break;
      default:
        appState.addMessage({
          role: 'system',
          content: `Unknown command: ${command}`,
        });
    }
  }, [appState]);
  
  return (
    <Box flexDirection="column" height="100%">
      {/* 消息列表 */}
      <Box flexGrow={1} flexDirection="column" overflow="hidden">
        <Messages messages={appState.messages} />
      </Box>
      
      {/* 分隔线 */}
      <Box borderStyle="single" borderTop width="100%" />
      
      {/* 输入区域 */}
      <Box flexDirection="column">
        <PromptInput
          value={input}
          onChange={setInput}
          onSubmit={handleSubmit}
          onSlashCommand={handleSlashCommand}
          disabled={isLoading}
          placeholder="Ask Claude..."
        />
      </Box>
      
      {/* 状态栏 */}
      <StatusBar
        isLoading={isLoading}
        tokenCount={appState.tokenCount}
        model={appState.model}
      />
    </Box>
  );
}
```

### 4.2 消息组件

**文件**: `src/components/Messages.tsx`

```typescript
import React, { useRef, useEffect } from 'react';
import { Box, Text } from 'ink';
import { MessageRow } from './MessageRow.js';
import { useVirtualList } from '../hooks/useVirtualList.js';

/**
 * 消息列表组件
 * 
 * 使用虚拟列表优化长对话的性能
 */
export function Messages({ messages }: MessagesProps): React.ReactElement {
  const containerRef = useRef<HTMLDivElement>(null);
  
  // 虚拟列表优化
  const { virtualItems, totalHeight, scrollToIndex } = useVirtualList({
    items: messages,
    estimateSize: () => 100,  // 估计每行高度
    overscan: 5,              // 预渲染数量
  });
  
  // 自动滚动到最新消息
  useEffect(() => {
    if (messages.length > 0) {
      scrollToIndex(messages.length - 1);
    }
  }, [messages.length]);
  
  return (
    <Box ref={containerRef} flexDirection="column" flexGrow={1} overflow="hidden">
      {virtualItems.map((virtualItem) => {
        const message = messages[virtualItem.index];
        return (
          <Box
            key={message.id}
            style={{
              position: 'absolute',
              top: virtualItem.offset,
              height: virtualItem.size,
            }}
          >
            <MessageRow message={message} />
          </Box>
        );
      })}
    </Box>
  );
}

interface MessagesProps {
  messages: Message[];
}
```

**文件**: `src/components/MessageRow.tsx`

```typescript
import React from 'react';
import { Box, Text } from 'ink';
import { HighlightedCode } from './HighlightedCode.js';
import { ToolUseDisplay } from './ToolUseDisplay.js';

/**
 * 单行消息组件
 * 
 * 根据消息类型渲染不同样式
 */
export function MessageRow({ message }: MessageRowProps): React.ReactElement {
  switch (message.role) {
    case 'user':
      return <UserMessage message={message} />;
    case 'assistant':
      return <AssistantMessage message={message} />;
    case 'system':
      return <SystemMessage message={message} />;
    default:
      return <Text>Unknown message type</Text>;
  }
}

function UserMessage({ message }: { message: UserMessage }): React.ReactElement {
  return (
    <Box flexDirection="column" paddingX={2} paddingY={1}>
      <Text bold color="blue">▌ You</Text>
      <Box marginLeft={2}>
        <Text>{message.content}</Text>
      </Box>
    </Box>
  );
}

function AssistantMessage({ message }: { message: AssistantMessage }): React.ReactElement {
  return (
    <Box flexDirection="column" paddingX={2} paddingY={1}>
      <Text bold color="green">▌ Claude</Text>
      <Box marginLeft={2} flexDirection="column">
        {message.content.map((block, i) => {
          switch (block.type) {
            case 'text':
              return <Text key={i}>{block.text}</Text>;
            case 'code':
              return (
                <HighlightedCode
                  key={i}
                  code={block.code}
                  language={block.language}
                />
              );
            case 'tool_use':
              return <ToolUseDisplay key={i} tool={block.tool} />;
            default:
              return null;
          }
        })}
      </Box>
    </Box>
  );
}

function SystemMessage({ message }: { message: SystemMessage }): React.ReactElement {
  return (
    <Box flexDirection="column" paddingX={2} paddingY={1}>
      <Text dimColor>ℹ {message.content}</Text>
    </Box>
  );
}
```

### 4.3 输入框组件

**文件**: `src/components/PromptInput/index.tsx`

```typescript
import React, { useState, useCallback } from 'react';
import { Box, Text, useInput } from 'ink';
import { TextInput } from './TextInput.js';
import { Autocomplete } from './Autocomplete.js';

/**
 * 提示输入组件
 * 
 * 支持：
 * - 多行输入
 * - 自动完成
 * - 斜杠命令
 * - 历史记录
 */
export function PromptInput({
  value,
  onChange,
  onSubmit,
  onSlashCommand,
  disabled,
  placeholder,
}: PromptInputProps): React.ReactElement {
  const [showAutocomplete, setShowAutocomplete] = useState(false);
  const [autocompleteItems, setAutocompleteItems] = useState<string[]>([]);
  
  // 监听输入变化以显示自动完成
  const handleChange = useCallback((newValue: string) => {
    onChange(newValue);
    
    // 检查是否需要自动完成
    if (newValue.startsWith('/')) {
      const command = newValue.slice(1);
      const matches = SLASH_COMMANDS.filter(cmd =>
        cmd.startsWith(command)
      );
      
      if (matches.length > 0) {
        setAutocompleteItems(matches);
        setShowAutocomplete(true);
      } else {
        setShowAutocomplete(false);
      }
    } else {
      setShowAutocomplete(false);
    }
  }, [onChange]);
  
  // 处理 Tab 自动完成
  useInput((input, key) => {
    if (key.tab && showAutocomplete && autocompleteItems.length > 0) {
      onChange('/' + autocompleteItems[0]);
      setShowAutocomplete(false);
    }
  });
  
  // 处理提交
  const handleSubmit = useCallback(() => {
    if (value.startsWith('/')) {
      onSlashCommand(value);
    } else {
      onSubmit(value);
    }
  }, [value, onSubmit, onSlashCommand]);
  
  return (
    <Box flexDirection="column" paddingX={1}>
      {/* 输入框 */}
      <Box>
        <Text color="cyan">❯ </Text>
        <TextInput
          value={value}
          onChange={handleChange}
          onSubmit={handleSubmit}
          disabled={disabled}
          placeholder={placeholder}
          multiline
        />
      </Box>
      
      {/* 自动完成下拉框 */}
      {showAutocomplete && (
        <Autocomplete
          items={autocompleteItems}
          onSelect={(item) => {
            onChange('/' + item);
            setShowAutocomplete(false);
          }}
        />
      )}
    </Box>
  );
}

const SLASH_COMMANDS = [
  'clear',
  'compact',
  'help',
  'cost',
  'model',
  'debug',
  'config',
];

interface PromptInputProps {
  value: string;
  onChange: (value: string) => void;
  onSubmit: (value: string) => void;
  onSlashCommand: (command: string) => void;
  disabled?: boolean;
  placeholder?: string;
}
```

## 5. 权限对话框

### 5.1 权限确认对话框

**文件**: `src/components/permissions/PermissionDialog.tsx`

```typescript
import React, { useState } from 'react';
import { Box, Text, useInput } from 'ink';
import { findToolByName } from '../../tools.js';

/**
 * 权限确认对话框
 * 
 * 在工具执行前请求用户确认
 */
export function PermissionDialog({
  toolName,
  input,
  onAllow,
  onDeny,
  onEdit,
}: PermissionDialogProps): React.ReactElement {
  const tool = findToolByName(toolName);
  const [editedInput, setEditedInput] = useState(input);
  const [isEditing, setIsEditing] = useState(false);
  
  // 渲染工具使用消息
  const toolMessage = tool?.renderToolUseMessage(editedInput, {});
  
  // 键盘处理
  useInput((_, key) => {
    if (isEditing) {
      // 编辑模式
      if (key.escape) {
        setIsEditing(false);
        setEditedInput(input);  // 恢复原始值
      }
      if (key.return) {
        setIsEditing(false);
      }
    } else {
      // 正常模式
      if (key.return) {
        onAllow(editedInput);
      }
      if (key.escape) {
        onDeny();
      }
    }
  });
  
  return (
    <Box flexDirection="column" borderStyle="round" padding={1}>
      <Text bold>Permission Required</Text>
      <Text dimColor>Claude wants to use the {toolName} tool:</Text>
      
      <Box marginY={1} paddingX={2}>
        {toolMessage}
      </Box>
      
      {/* 可编辑字段 */}
      {tool?.userFacingFields && (
        <Box flexDirection="column" marginY={1}>
          <Text dimColor>Editable fields:</Text>
          {tool.userFacingFields.map(field => (
            <EditableField
              key={field}
              name={field}
              value={editedInput[field]}
              onChange={(value) =>
                setEditedInput({ ...editedInput, [field]: value })
              }
              isEditing={isEditing}
            />
          ))}
        </Box>
      )}
      
      {/* 操作按钮 */}
      <Box marginTop={1}>
        <Text color="green">[Enter]</Text>
        <Text> Allow </Text>
        <Text color="red">[Esc]</Text>
        <Text> Deny</Text>
        {tool?.userFacingFields && (
          <>
            <Text> | </Text>
            <Text color="blue">[Tab]</Text>
            <Text> Edit</Text>
          </>
        )}
      </Box>
    </Box>
  );
}

interface PermissionDialogProps {
  toolName: string;
  input: Record<string, unknown>;
  onAllow: (input: Record<string, unknown>) => void;
  onDeny: () => void;
  onEdit?: () => void;
}
```

## 6. 样式系统

### 6.1 主题配置

**文件**: `src/components/design-system/theme.ts`

```typescript
/**
 * 主题配置
 * 
 * 支持亮色/暗色模式
 */
export const lightTheme: Theme = {
  colors: {
    primary: '#0066CC',
    secondary: '#6C757D',
    success: '#28A745',
    danger: '#DC3545',
    warning: '#FFC107',
    info: '#17A2B8',
    light: '#F8F9FA',
    dark: '#343A40',
    text: '#212529',
    background: '#FFFFFF',
    border: '#DEE2E6',
  },
  fonts: {
    normal: 'system',
    bold: 'system-bold',
    monospace: 'monospace',
  },
};

export const darkTheme: Theme = {
  colors: {
    primary: '#4DABF7',
    secondary: '#ADB5BD',
    success: '#51CF66',
    danger: '#FF6B6B',
    warning: '#FFD43B',
    info: '#74C0FC',
    light: '#F8F9FA',
    dark: '#212529',
    text: '#E9ECEF',
    background: '#1A1A1A',
    border: '#343A40',
  },
  fonts: {
    normal: 'system',
    bold: 'system-bold',
    monospace: 'monospace',
  },
};

export type Theme = {
  colors: {
    primary: string;
    secondary: string;
    success: string;
    danger: string;
    warning: string;
    info: string;
    light: string;
    dark: string;
    text: string;
    background: string;
    border: string;
  };
  fonts: {
    normal: string;
    bold: string;
    monospace: string;
  };
};
```

### 6.2 主题提供者

```typescript
import React, { createContext, useContext } from 'react';
import { lightTheme, darkTheme, Theme } from './theme.js';

const ThemeContext = createContext<Theme>(darkTheme);

export function ThemeProvider({
  children,
  theme = 'dark',
}: ThemeProviderProps): React.ReactElement {
  const currentTheme = theme === 'light' ? lightTheme : darkTheme;
  
  return (
    <ThemeContext.Provider value={currentTheme}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme(): Theme {
  return useContext(ThemeContext);
}
```

## 7. 性能优化

### 7.1 虚拟列表

```typescript
/**
 * useVirtualList Hook
 * 
 * 只渲染可见区域的内容，优化长列表性能
 */
export function useVirtualList<T>({
  items,
  estimateSize,
  overscan = 5,
}: VirtualListOptions<T>): VirtualListResult<T> {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scrollOffset, setScrollOffset] = useState(0);
  const [containerHeight, setContainerHeight] = useState(0);
  
  // 计算可见范围
  const { virtualItems, startIndex, endIndex } = useMemo(() => {
    const start = Math.floor(scrollOffset / estimateSize());
    const end = Math.ceil((scrollOffset + containerHeight) / estimateSize());
    
    // 添加 overscan
    const overscanStart = Math.max(0, start - overscan);
    const overscanEnd = Math.min(items.length - 1, end + overscan);
    
    const virtualItems = [];
    for (let i = overscanStart; i <= overscanEnd; i++) {
      virtualItems.push({
        index: i,
        offset: i * estimateSize(),
        size: estimateSize(),
        item: items[i],
      });
    }
    
    return { virtualItems, startIndex: overscanStart, endIndex: overscanEnd };
  }, [items, scrollOffset, containerHeight, estimateSize, overscan]);
  
  // 监听滚动
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;
    
    const handleScroll = () => {
      setScrollOffset(container.scrollTop);
    };
    
    container.addEventListener('scroll', handleScroll);
    return () => container.removeEventListener('scroll', handleScroll);
  }, []);
  
  // 滚动到指定索引
  const scrollToIndex = useCallback((index: number) => {
    const container = containerRef.current;
    if (!container) return;
    
    container.scrollTop = index * estimateSize();
  }, [estimateSize]);
  
  return {
    containerRef,
    virtualItems,
    totalHeight: items.length * estimateSize(),
    scrollToIndex,
  };
}
```

### 7.2 代码分割

```typescript
// 动态导入大型组件
const SyntaxHighlighter = React.lazy(() =>
  import('./SyntaxHighlighter.js')
);

// 使用 Suspense
function CodeBlock({ code, language }: CodeBlockProps) {
  return (
    <React.Suspense fallback={<Text>Loading...</Text>}>
      <SyntaxHighlighter code={code} language={language} />
    </React.Suspense>
  );
}
```

## 8. 总结

UI 系统的设计体现了以下架构原则：

1. **声明式**: 使用 React 的声明式语法描述 UI
2. **组件化**: 可复用、可组合的组件设计
3. **类型安全**: TypeScript 提供完整的类型支持
4. **性能优化**: 虚拟列表、代码分割等优化手段
5. **可测试性**: 组件易于单元测试
6. **可定制性**: 主题系统支持个性化配置

自定义 Ink 框架提供了对渲染流程的精细控制，使得 Claude Code 能够在终端环境中实现复杂的交互界面。
