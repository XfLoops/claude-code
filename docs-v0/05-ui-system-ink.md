# UI 系统详解 (Ink 框架)

## Ink 架构概述

Claude Code 使用 **React + Ink** 构建终端 UI。Ink 是一个基于 React 的终端 UI 渲染框架，本项目使用了自定义的 Ink 分支。

```
┌─────────────────────────────────────────────────────────────────┐
│                      Ink 架构                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    React                                 │   │
│  │              (组件化 UI)                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Custom Reconciler                           │   │
│  │            (ink/reconciler.ts)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Virtual DOM (ink/dom.ts)                    │   │
│  │         ┌─────────┐ ┌─────────┐ ┌─────────┐            │   │
│  │         │  Box    │ │  Text   │ │Spacer   │            │   │
│  │         └─────────┘ └─────────┘ └─────────┘            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Layout Engine                               │   │
│  │         (ink/layout/engine.ts)                          │   │
│  │              Yoga Layout                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Output Rendering                            │   │
│  │        (ink/render-to-screen.ts)                        │   │
│  │              ANSI Escape Codes                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Ink 核心组件

### 自定义 Reconciler

```typescript
// src/ink/reconciler.ts

import { reconciler as baseReconciler } from 'react-reconciler'

export const reconciler = baseReconciler({
  // 创建宿主配置
  supportsMutation: true,
  supportsHydration: false,
  supportsPersistence: false,
  
  // 创建实例
  createInstance(type, props, rootContainer, hostContext, internalHandle) {
    return createElement(type, props, rootContainer, hostContext)
  },
  
  // 创建文本实例
  createTextInstance(text, rootContainer, hostContext, internalHandle) {
    return createTextNode(text, rootContainer)
  },
  
  // 添加子节点
  appendChild(parent, child) {
    parent.appendChild(child)
  },
  
  // 插入子节点
  insertBefore(parent, child, beforeChild) {
    parent.insertBefore(child, beforeChild)
  },
  
  // 移除子节点
  removeChild(parent, child) {
    parent.removeChild(child)
  },
  
  // 属性更新
  prepareUpdate(instance, type, oldProps, newProps, rootContainer, hostContext) {
    return diffProps(oldProps, newProps)
  },
  
  commitUpdate(instance, updatePayload, type, oldProps, newProps, internalHandle) {
    instance.updateProps(updatePayload)
  },
  
  // 其他宿主方法...
})
```

### DOM 元素类型

```typescript
// src/ink/dom.ts

export type DOMElementType = 'box' | 'text' | 'newline' | 'spacer'

export interface DOMElement {
  type: DOMElementType
  props: Record<string, unknown>
  children: DOMElement[]
  parent: DOMElement | null
  
  // 布局属性
  yogaNode: YogaNode
  computedStyle: ComputedStyle
  
  // 渲染属性
  output: OutputBuffer
  
  // 方法
  appendChild(child: DOMElement): void
  insertBefore(child: DOMElement, beforeChild: DOMElement): void
  removeChild(child: DOMElement): void
  updateProps(props: Record<string, unknown>): void
  calculateLayout(width: number, height: number): void
  render(): string
}

export interface TextElement extends DOMElement {
  type: 'text'
  text: string
  style: TextStyle
}

export interface BoxElement extends DOMElement {
  type: 'box'
  style: BoxStyle
  border?: BorderConfig
  padding?: SpacingConfig
  margin?: SpacingConfig
}
```

## 核心 UI 组件

### Box 组件

```typescript
// src/ink/components/Box.tsx

import React from 'react'

export interface BoxProps {
  // 尺寸
  width?: number | string
  height?: number | string
  minWidth?: number
  minHeight?: number
  maxWidth?: number
  maxHeight?: number
  
  // Flex 布局
  display?: 'flex' | 'none'
  flexDirection?: 'row' | 'column' | 'row-reverse' | 'column-reverse'
  flexWrap?: 'wrap' | 'nowrap' | 'wrap-reverse'
  flexGrow?: number
  flexShrink?: number
  flexBasis?: number | string
  
  // 对齐
  justifyContent?: 'flex-start' | 'flex-end' | 'center' | 'space-between' | 'space-around'
  alignItems?: 'flex-start' | 'flex-end' | 'center' | 'stretch' | 'baseline'
  alignSelf?: 'auto' | 'flex-start' | 'flex-end' | 'center' | 'stretch' | 'baseline'
  
  // 间距
  padding?: number | Spacing
  paddingTop?: number
  paddingBottom?: number
  paddingLeft?: number
  paddingRight?: number
  margin?: number | Spacing
  marginTop?: number
  marginBottom?: number
  marginLeft?: number
  marginRight?: number
  gap?: number
  rowGap?: number
  columnGap?: number
  
  // 边框
  borderStyle?: 'single' | 'double' | 'round' | 'bold' | 'singleDouble' | 'doubleSingle' | 'classic'
  borderColor?: string
  borderTop?: boolean
  borderBottom?: boolean
  borderLeft?: boolean
  borderRight?: boolean
  
  // 背景
  backgroundColor?: string
  backgroundOpacity?: number
  
  // 溢出
  overflow?: 'visible' | 'hidden'
  
  // 其他
  dimColor?: boolean
  textWrap?: 'wrap' | 'end' | 'truncate' | 'truncate-start' | 'truncate-middle'
}

export const Box: React.FC<BoxProps> = (props) => {
  // 转换为 Yoga 布局属性
  return React.createElement('box', props)
}
```

### Text 组件

```typescript
// src/ink/components/Text.tsx

import React from 'react'

export interface TextProps {
  // 内容
  children?: React.ReactNode
  
  // 样式
  color?: string
  backgroundColor?: string
  
  // 文字修饰
  bold?: boolean
  italic?: boolean
  underline?: boolean
  strikethrough?: boolean
  dimColor?: boolean
  inverse?: boolean
  hidden?: boolean
  
  // 换行
  wrap?: 'wrap' | 'end' | 'truncate' | 'truncate-start' | 'truncate-middle'
  
  // 对齐
  align?: 'left' | 'center' | 'right'
}

export const Text: React.FC<TextProps> = (props) => {
  return React.createElement('text', props)
}
```

## 自定义 Hooks

### useInput

```typescript
// src/ink/hooks/use-input.ts

import { useEffect, useCallback } from 'react'
import { useStdin } from './use-stdin.js'
import { parseKeypress } from '../parse-keypress.js'

export type Key = {
  // 修饰键
  ctrl: boolean
  shift: boolean
  alt: boolean
  meta: boolean
  
  // 特殊键
  upArrow: boolean
  downArrow: boolean
  leftArrow: boolean
  rightArrow: boolean
  pageUp: boolean
  pageDown: boolean
  home: boolean
  end: boolean
  return: boolean
  escape: boolean
  backspace: boolean
  tab: boolean
  delete: boolean
  
  // 功能键
  f1: boolean
  f2: boolean
  // ... f3-f12
}

export function useInput(
  handler: (input: string, key: Key) => void,
  options?: {
    isActive?: boolean
  }
) {
  const { stdin, setRawMode } = useStdin()
  
  const handleInput = useCallback((data: Buffer) => {
    const { input, key } = parseKeypress(data)
    handler(input, key)
  }, [handler])
  
  useEffect(() => {
    if (options?.isActive === false) return
    
    setRawMode(true)
    stdin.on('data', handleInput)
    
    return () => {
      stdin.off('data', handleInput)
      setRawMode(false)
    }
  }, [stdin, setRawMode, handleInput, options?.isActive])
}
```

### useTerminalSize

```typescript
// src/hooks/useTerminalSize.ts

import { useState, useEffect } from 'react'

export interface TerminalSize {
  width: number
  height: number
}

export function useTerminalSize(): TerminalSize {
  const [size, setSize] = useState<TerminalSize>({
    width: process.stdout.columns || 80,
    height: process.stdout.rows || 24,
  })
  
  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: process.stdout.columns || 80,
        height: process.stdout.rows || 24,
      })
    }
    
    process.stdout.on('resize', handleResize)
    return () => {
      process.stdout.off('resize', handleResize)
    }
  }, [])
  
  return size
}
```

### useSearchHighlight

```typescript
// src/ink/hooks/use-search-highlight.ts

import { useState, useCallback } from 'react'

export type SearchState = {
  query: string
  isActive: boolean
  matches: SearchMatch[]
  currentMatchIndex: number
}

export type SearchMatch = {
  line: number
  start: number
  end: number
}

export function useSearchHighlight() {
  const [searchState, setSearchState] = useState<SearchState>({
    query: '',
    isActive: false,
    matches: [],
    currentMatchIndex: -1,
  })
  
  const startSearch = useCallback((query: string) => {
    setSearchState(prev => ({
      ...prev,
      query,
      isActive: true,
      matches: [],
      currentMatchIndex: -1,
    }))
  }, [])
  
  const updateMatches = useCallback((matches: SearchMatch[]) => {
    setSearchState(prev => ({
      ...prev,
      matches,
      currentMatchIndex: matches.length > 0 ? 0 : -1,
    }))
  }, [])
  
  const nextMatch = useCallback(() => {
    setSearchState(prev => ({
      ...prev,
      currentMatchIndex: 
        prev.currentMatchIndex < prev.matches.length - 1
          ? prev.currentMatchIndex + 1
          : 0,
    }))
  }, [])
  
  const previousMatch = useCallback(() => {
    setSearchState(prev => ({
      ...prev,
      currentMatchIndex: 
        prev.currentMatchIndex > 0
          ? prev.currentMatchIndex - 1
          : prev.matches.length - 1,
    }))
  }, [])
  
  const exitSearch = useCallback(() => {
    setSearchState({
      query: '',
      isActive: false,
      matches: [],
      currentMatchIndex: -1,
    })
  }, [])
  
  return {
    searchState,
    startSearch,
    updateMatches,
    nextMatch,
    previousMatch,
    exitSearch,
  }
}
```

## REPL 屏幕组件

### REPL 组件结构

```typescript
// src/screens/REPL.tsx (简化版)

import React, { useState, useCallback, useEffect } from 'react'
import { Box, Text, useInput } from '../ink.js'
import { Messages } from '../components/Messages.js'
import PromptInput from '../components/PromptInput/PromptInput.js'
import { PermissionRequest } from '../components/permissions/PermissionRequest.js'
import { useAppState, useSetAppState } from '../state/AppState.js'

export const REPL: React.FC<REPLProps> = ({
  initialMessages = [],
  commands,
  tools,
  /* ... */
}) => {
  // 状态
  const appState = useAppState(s => s)
  const setAppState = useSetAppState()
  const [isReady, setIsReady] = useState(false)
  
  // 消息处理
  const handleSubmit = useCallback(async (input: string) => {
    // 1. 创建用户消息
    const userMessage = createUserMessage({ content: input })
    
    // 2. 添加到状态
    setAppState(prev => ({
      ...prev,
      messages: [...prev.messages, userMessage],
    }))
    
    // 3. 启动查询
    await startQuery(userMessage)
  }, [setAppState])
  
  // 查询处理
  const startQuery = async (userMessage: UserMessage) => {
    // 1. 构建上下文
    const context = buildToolUseContext({
      getAppState: () => appState,
      setAppState,
      /* ... */
    })
    
    // 2. 调用 query
    const stream = query({
      messages: appState.messages,
      systemPrompt: await getSystemPrompt(),
      toolUseContext: context,
      /* ... */
    })
    
    // 3. 消费流
    for await (const event of stream) {
      handleStreamEvent(event, setAppState)
    }
  }
  
  // 键盘处理
  useInput((input, key) => {
    if (key.escape) {
      // 取消当前操作
      appState.abortController.abort()
    } else if (key.ctrl && input === 'c') {
      // 复制
      handleCopy()
    } else if (key.ctrl && input === 'v') {
      // 粘贴
      handlePaste()
    }
  })
  
  return (
    <Box flexDirection="column" height="100%">
      {/* 消息列表 */}
      <Box flexGrow={1} overflow="hidden">
        <Messages
          messages={appState.messages}
          isSearching={searchState.isActive}
          searchMatches={searchState.matches}
          currentMatchIndex={searchState.currentMatchIndex}
        />
      </Box>
      
      {/* 权限请求 */}
      {appState.pendingPermission && (
        <PermissionRequest
          permission={appState.pendingPermission}
          onAllow={() => handlePermission(true)}
          onDeny={() => handlePermission(false)}
        />
      )}
      
      {/* 输入框 */}
      <Box height={inputHeight} flexShrink={0}>
        <PromptInput
          onSubmit={handleSubmit}
          isProcessing={appState.isProcessing}
          commands={commands}
        />
      </Box>
    </Box>
  )
}
```

## 消息渲染组件

### Messages 组件

```typescript
// src/components/Messages.tsx

import React, { useRef, useMemo } from 'react'
import { Box, Text } from '../ink.js'
import { MessageRow } from './MessageRow.js'
import { VirtualMessageList } from './VirtualMessageList.js'

export interface MessagesProps {
  messages: Message[]
  isSearching: boolean
  searchMatches: SearchMatch[]
  currentMatchIndex: number
  onScroll?: (offset: number) => void
}

export const Messages: React.FC<MessagesProps> = ({
  messages,
  isSearching,
  searchMatches,
  currentMatchIndex,
  onScroll,
}) => {
  const containerRef = useRef<Box>(null)
  
  // 分组消息（折叠工具调用）
  const groupedMessages = useMemo(() => {
    return groupMessages(messages)
  }, [messages])
  
  return (
    <VirtualMessageList
      ref={containerRef}
      items={groupedMessages}
      renderItem={(message, index) => (
        <MessageRow
          key={message.uuid}
          message={message}
          isHighlighted={isSearching && isMatch(index, searchMatches, currentMatchIndex)}
          isSearchActive={isSearching}
        />
      )}
      onScroll={onScroll}
    />
  )
}
```

### MessageRow 组件

```typescript
// src/components/MessageRow.tsx

import React from 'react'
import { Box, Text } from '../ink.js'
import type { RenderableMessage } from '../types/message.js'

export interface MessageRowProps {
  message: RenderableMessage
  isHighlighted?: boolean
  isSearchActive?: boolean
}

export const MessageRow: React.FC<MessageRowProps> = ({
  message,
  isHighlighted,
  isSearchActive,
}) => {
  switch (message.type) {
    case 'user':
      return <UserMessageRow message={message} isHighlighted={isHighlighted} />
    case 'assistant':
      return <AssistantMessageRow message={message} isHighlighted={isHighlighted} />
    case 'system':
      return <SystemMessageRow message={message} isHighlighted={isHighlighted} />
    case 'progress':
      return <ProgressMessageRow message={message} />
    case 'grouped_tool_use':
      return <GroupedToolUseRow message={message} />
    default:
      return null
  }
}

const UserMessageRow: React.FC<{ message: UserMessage; isHighlighted?: boolean }> = ({
  message,
  isHighlighted,
}) => (
  <Box 
    padding={1} 
    borderStyle="single" 
    borderColor={isHighlighted ? 'yellow' : 'blue'}
    backgroundColor={isHighlighted ? 'rgb(255,255,0,0.1)' : undefined}
  >
    <Text bold color="blue">You</Text>
    <Text>{getMessageText(message)}</Text>
  </Box>
)

const AssistantMessageRow: React.FC<{ message: AssistantMessage; isHighlighted?: boolean }> = ({
  message,
  isHighlighted,
}) => (
  <Box 
    padding={1}
    backgroundColor={isHighlighted ? 'rgb(255,255,0,0.1)' : undefined}
  >
    <Text bold color="green">Claude</Text>
    <Text>{renderAssistantContent(message)}</Text>
  </Box>
)
```

## 虚拟列表

### VirtualMessageList 组件

```typescript
// src/components/VirtualMessageList.tsx

import React, { useRef, useState, useCallback, useEffect } from 'react'
import { Box } from '../ink.js'

export interface VirtualMessageListProps<T> {
  items: T[]
  renderItem: (item: T, index: number) => React.ReactNode
  onScroll?: (offset: number) => void
  overscan?: number
}

export const VirtualMessageList = <T,>({
  items,
  renderItem,
  onScroll,
  overscan = 5,
}: VirtualMessageListProps<T>) => {
  const containerRef = useRef<Box>(null)
  const [visibleRange, setVisibleRange] = useState({ start: 0, end: 20 })
  const [itemHeights, setItemHeights] = useState<Map<number, number>>(new Map())
  
  // 计算可见范围
  const calculateVisibleRange = useCallback(() => {
    const container = containerRef.current
    if (!container) return
    
    const { scrollTop, clientHeight } = container
    let currentOffset = 0
    let startIndex = 0
    let endIndex = items.length - 1
    
    // 找到起始索引
    for (let i = 0; i < items.length; i++) {
      const height = itemHeights.get(i) || DEFAULT_ITEM_HEIGHT
      if (currentOffset + height > scrollTop) {
        startIndex = Math.max(0, i - overscan)
        break
      }
      currentOffset += height
    }
    
    // 找到结束索引
    currentOffset = 0
    for (let i = 0; i < items.length; i++) {
      const height = itemHeights.get(i) || DEFAULT_ITEM_HEIGHT
      currentOffset += height
      if (currentOffset > scrollTop + clientHeight) {
        endIndex = Math.min(items.length - 1, i + overscan)
        break
      }
    }
    
    setVisibleRange({ start: startIndex, end: endIndex })
  }, [items, itemHeights, overscan])
  
  // 测量项目高度
  const measureItem = useCallback((index: number, height: number) => {
    setItemHeights(prev => {
      const next = new Map(prev)
      next.set(index, height)
      return next
    })
  }, [])
  
  // 渲染可见项目
  const visibleItems = items.slice(visibleRange.start, visibleRange.end + 1)
  
  return (
    <Box
      ref={containerRef}
      flexDirection="column"
      overflow="hidden"
      onScroll={calculateVisibleRange}
    >
      {/* 顶部占位 */}
      {visibleRange.start > 0 && (
        <Box height={calculateTotalHeight(0, visibleRange.start)} />
      )}
      
      {/* 可见项目 */}
      {visibleItems.map((item, index) => (
        <MeasureableItem
          key={visibleRange.start + index}
          index={visibleRange.start + index}
          onMeasure={measureItem}
        >
          {renderItem(item, visibleRange.start + index)}
        </MeasureableItem>
      ))}
      
      {/* 底部占位 */}
      {visibleRange.end < items.length - 1 && (
        <Box height={calculateTotalHeight(visibleRange.end + 1, items.length)} />
      )}
    </Box>
  )
}
```

## 主题系统

### ThemeProvider

```typescript
// src/components/design-system/ThemeProvider.tsx

import React, { useContext, useState, useEffect } from 'react'

export type ThemeName = 'light' | 'dark' | 'high-contrast'

export interface Theme {
  name: ThemeName
  colors: {
    primary: string
    secondary: string
    success: string
    error: string
    warning: string
    info: string
    text: string
    textMuted: string
    background: string
    backgroundMuted: string
    border: string
  }
}

const themes: Record<ThemeName, Theme> = {
  light: {
    name: 'light',
    colors: {
      primary: '#0066CC',
      secondary: '#6C757D',
      success: '#28A745',
      error: '#DC3545',
      warning: '#FFC107',
      info: '#17A2B8',
      text: '#212529',
      textMuted: '#6C757D',
      background: '#FFFFFF',
      backgroundMuted: '#F8F9FA',
      border: '#DEE2E6',
    },
  },
  dark: {
    name: 'dark',
    colors: {
      primary: '#0D6EFD',
      secondary: '#ADB5BD',
      success: '#198754',
      error: '#DC3545',
      warning: '#FFC107',
      info: '#0DCAF0',
      text: '#F8F9FA',
      textMuted: '#ADB5BD',
      background: '#212529',
      backgroundMuted: '#343A40',
      border: '#495057',
    },
  },
  'high-contrast': {
    name: 'high-contrast',
    colors: {
      primary: '#00FFFF',
      secondary: '#FFFF00',
      success: '#00FF00',
      error: '#FF0000',
      warning: '#FFFF00',
      info: '#00FFFF',
      text: '#FFFFFF',
      textMuted: '#FFFF00',
      background: '#000000',
      backgroundMuted: '#000000',
      border: '#FFFFFF',
    },
  },
}

const ThemeContext = React.createContext<Theme>(themes.dark)

export const ThemeProvider: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const [theme, setTheme] = useState<Theme>(themes.dark)
  
  // 检测系统主题
  useEffect(() => {
    const detectTheme = () => {
      // 检测终端背景色
      const isDark = detectTerminalBackground()
      setTheme(isDark ? themes.dark : themes.light)
    }
    
    detectTheme()
  }, [])
  
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  )
}

export function useTheme(): Theme {
  return useContext(ThemeContext)
}

export function usePreviewTheme(themeName: ThemeName): Theme {
  return themes[themeName]
}
```

## 渲染输出

### 渲染到终端

```typescript
// src/ink/render-to-screen.ts

import { type OutputBuffer } from './output.js'
import { type Terminal } from './terminal.js'

export function renderToScreen(
  buffer: OutputBuffer,
  terminal: Terminal,
  options?: {
    clear?: boolean
    debug?: boolean
  }
): void {
  const output = buffer.toString()
  
  if (options?.clear) {
    terminal.clear()
  }
  
  // 写入输出
  terminal.write(output)
  
  // 更新光标位置
  terminal.updateCursor(buffer.getCursorPosition())
}
```

### ANSI 处理

```typescript
// src/ink/Ansi.tsx

import React from 'react'

export interface AnsiProps {
  children: string
  
  // ANSI 解析选项
  parseHyperlinks?: boolean
  parseColors?: boolean
  parseStyles?: boolean
}

export const Ansi: React.FC<AnsiProps> = ({
  children,
  parseHyperlinks = true,
  parseColors = true,
  parseStyles = true,
}) => {
  // 解析 ANSI 转义序列
  const parsed = parseAnsi(children, {
    parseHyperlinks,
    parseColors,
    parseStyles,
  })
  
  return (
    <>
      {parsed.map((segment, index) => (
        <Text
          key={index}
          color={segment.color}
          backgroundColor={segment.backgroundColor}
          bold={segment.bold}
          italic={segment.italic}
          underline={segment.underline}
        >
          {segment.text}
        </Text>
      ))}
    </>
  )
}
```

## 性能优化

### React Compiler 优化

```typescript
// 使用 React Compiler 的 memoization
import { c as _c } from "react/compiler-runtime"

export function AppStateProvider(t0) {
  const $ = _c(13)  // 缓存大小为 13
  const { children, initialState, onChangeAppState } = t0
  
  // 缓存 store 创建
  let t1
  if ($[0] !== initialState || $[1] !== onChangeAppState) {
    t1 = () => createStore(initialState ?? getDefaultAppState(), onChangeAppState)
    $[0] = initialState
    $[1] = onChangeAppState
    $[2] = t1
  } else {
    t1 = $[2]
  }
  
  const [store] = useState(t1)
  
  // ... 更多缓存
}
```

### 增量渲染

```typescript
// src/ink/frame.ts

export class Frame {
  private lastOutput: string = ''
  private pendingOutput: string = ''
  private renderScheduled: boolean = false
  
  scheduleRender(output: string): void {
    this.pendingOutput = output
    
    if (!this.renderScheduled) {
      this.renderScheduled = true
      
      // 使用 requestAnimationFrame 模式
      setImmediate(() => {
        this.renderScheduled = false
        this.flush()
      })
    }
  }
  
  private flush(): void {
    if (this.pendingOutput === this.lastOutput) {
      return
    }
    
    // 计算 diff
    const diff = calculateDiff(this.lastOutput, this.pendingOutput)
    
    // 只输出变化的部分
    applyDiff(diff)
    
    this.lastOutput = this.pendingOutput
  }
}
```
