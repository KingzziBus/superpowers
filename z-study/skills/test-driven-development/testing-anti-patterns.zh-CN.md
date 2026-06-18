# 测试反模式

**在以下情况加载此参考：** 编写或修改测试、添加 mocks，或想要给生产代码添加仅测试方法时。

## 概述

测试必须验证真实行为，而非 mock 行为。Mocks 是隔离的手段，而非被测试的对象。

**核心原则：** 测试代码做什么，而非 mocks 做什么。

**严格遵循 TDD 可以防止这些反模式。**

## 铁律

```
1. 永远不要测试 mock 行为
2. 永远不要给生产类添加仅测试方法
3. 永远不要在不理解依赖的情况下进行 mock
```

## 反模式 1：测试 Mock 行为

**违规：**
```typescript
// ❌ 坏：测试 mock 存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错的：**
- 你在验证 mock 工作，而非组件工作
- 测试在 mock 存在时通过，不存在时失败
- 告诉你的是关于真实行为的零信息

**你的 human partner 的纠正：**"我们在测试一个 mock 的行为吗？"

**修复：**
```typescript
// ✅ 好：测试真实组件或不 mock 它
test('renders sidebar', () => {
  render(<Page />);  // 不要 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果 sidebar 必须被 mock 以隔离：
// 不要断言 mock —— 测试 Page 在 sidebar 存在时的行为
```

### 门控函数

```
BEFORE asserting on any mock element:
  Ask: "Am I testing real component behavior or just mock existence?"

  IF testing mock existence:
    STOP - Delete the assertion or unmock the component

  Test real behavior instead
```

## 反模式 2：生产类中的仅测试方法

**违规：**
```typescript
// ❌ 坏：destroy() 仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错的：**
- 生产类被仅测试代码污染
- 如果在生产中意外调用很危险
- 违反 YAGNI 和关注点分离
- 混淆对象生命周期与实体生命周期

**修复：**
```typescript
// ✅ 好：测试工具处理测试清理
// Session 没有 destroy() —— 在生产中它是无状态的

// 在 test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试中
afterEach(() => cleanupSession(session));
```

### 门控函数

```
BEFORE adding any method to production class:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in test utilities instead

  Ask: "Does this class own this resource's lifecycle?"

  IF no:
    STOP - Wrong class for this method
```

## 反模式 3：未理解就 Mock

**违规：**
```typescript
// ❌ 坏：Mock 破坏了测试逻辑
test('detects duplicate server', () => {
  // Mock 阻止了测试依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出 - 但不会！
});
```

**为什么这是错的：**
- Mocked 的方法有测试依赖的副作用（写入配置）
- 过度 mock "以防万一" 破坏了实际行为
- 测试因错误的原因通过或神秘失败

**修复：**
```typescript
// ✅ 好：在正确的层级 mock
test('detects duplicate server', () => {
  // Mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只 mock 慢的服务器启动

  await addServer(config);  // 配置被写入
  await addServer(config);  // 重复被检测到 ✓
});
```

### 门控函数

```
BEFORE mocking any method:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real method have?"
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (the actual slow/external operation)
    OR use test doubles that preserve necessary behavior
    NOT the high-level method the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Observe what actually needs to happen
    THEN add minimal mocking at the right level

  Red flags:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

## 反模式 4：不完整的 Mocks

**违规：**
```typescript
// ❌ 坏：部分 mock —— 只有你认为你需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺少：下游代码使用的 metadata
};

// 稍后：当代码访问 response.metadata.requestId 时破坏
```

**为什么这是错的：**
- **部分 mock 隐藏结构假设** —— 你只 mock 你知道的字段
- **下游代码可能依赖你没有包含的字段** —— 静默失败
- **测试通过但集成失败** —— Mock 不完整，真实 API 完整
- **虚假自信** —— 测试对真实行为什么都没证明

**铁律：** Mock 真实存在的完整数据结构，而非你直接测试使用的字段。

**修复：**
```typescript
// ✅ 好：镜像真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### 门控函数

```
BEFORE creating mock responses:
  Check: "What fields does the real API response contain?"

  Actions:
    1. Examine actual API response from docs/examples
    2. Include ALL fields system might consume downstream
    3. Verify mock matches real response schema completely

  Critical:
    If you're creating a mock, you must understand the ENTIRE structure
    Partial mocks fail silently when code depends on omitted fields

  If uncertain: Include all documented fields
```

## 反模式 5：集成测试作为事后补充

**违规：**
```
✅ 实现完成
❌ 没有写测试
"准备好测试了"
```

**为什么这是错的：**
- 测试是实现的一部分，而非可选的后续步骤
- TDD 会捕获这个问题
- 没有测试就不能声称完成

**修复：**
```
TDD 循环：
1. 写失败测试
2. 实现以通过
3. 重构
4. 然后才声称完成
```

## 当 Mocks 变得太复杂时

**警告信号：**
- Mock setup 比测试逻辑还长
- Mock 一切以让测试通过
- Mocks 缺少真实组件拥有的方法
- Mock 改变时测试就破坏

**你的 human partner 的问题：**"我们需要在这里使用 mock 吗？"

**考虑：** 真实组件的集成测试通常比复杂 mocks 更简单

## TDD 防止这些反模式

**TDD 为何有帮助：**
1. **先写测试** → 迫使你思考你真正在测试什么
2. **看到它失败** → 确认测试测试的是真实行为，而非 mocks
3. **最少的实现** → 没有仅测试的方法潜入
4. **真实依赖** → 你在 mock 之前看到测试实际需要什么

**如果你在测试 mock 行为，你违反了 TDD** —— 你在没先看到测试对真实代码失败的情况下添加了 mocks。

## 快速参考

| 反模式 | 修复 |
|--------------|-----|
| 断言 mock 元素 | 测试真实组件或不 mock |
| 生产中的仅测试方法 | 移到测试工具中 |
| 未理解就 mock | 先理解依赖，最小化 mock |
| 不完整的 mocks | 完整镜像真实 API |
| 测试作为事后补充 | TDD —— 测试优先 |
| 过度复杂的 mocks | 考虑集成测试 |

## 红旗

- 断言检查 `*-mock` 测试 ID
- 方法仅在测试文件中被调用
- Mock setup 占测试的 50% 以上
- 当你移除 mock 时测试失败
- 无法解释为何需要 mock
- "为了安全" 而 mock

## 底线

**Mocks 是用于隔离的工具，而非被测试的对象。**

如果 TDD 揭示你在测试 mock 行为，说明你走错了。

修复：测试真实行为或质疑你为何要 mock。
