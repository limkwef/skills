---
name: test-module
description: 对指定模块进行系统性功能测试，输出 Playwright 网页版测试报告
metadata:
  type: skill
---

# 测试模块

对指定模块进行系统性功能测试，覆盖前端 UI 交互和 API 接口，最终输出 Playwright HTML 报告。

## 前置确认

执行前先确认以下事项，如有问题直接告知用户：

- [ ] 后端服务正在运行（默认 `http://localhost:8080`）
- [ ] 前端服务正在运行（默认 `http://localhost:3000`）
- [ ] `frontend/node_modules` 存在（如缺少，先 `cd frontend && npm install`）

## 工作流程

### Phase 1：代码分析

**按顺序读取以下文件，构建模块完整画像：**

1. **路由定义** — `frontend/src/router/index.js`：找到模块对应的路由路径、路由守卫、角色限制
2. **API 封装** — `frontend/src/api/*.js`：找到模块调用的所有后端接口（URL、方法、参数）
3. **页面组件** — `frontend/src/views/<模块>/**/*.vue`：找到表单字段、按钮操作、交互逻辑
4. **状态管理** — `frontend/src/stores/*.js`：找到模块依赖的 Pinia store（用户状态等）
5. **后端接口**（如有权限）— Controller 层的注解、参数校验、权限控制

**输出：** 在对话中简要列出分析结果——路由路径、API 端点清单、关键交互点。

### Phase 2：测试用例设计

根据 Phase 1 的分析结果，对照下方 **用例设计规则表** 累加最少用例数，然后按优先级排序：

- **P0**：正常流程 + 异常输入 + 安全（必须覆盖）
- **P1**：边界条件（尽量覆盖）
- **P2**：降级容错（时间允许时覆盖）

**输出：** 在对话中列出用例清单（编号、优先级、描述、预期结果），等用户确认后再编写脚本。

### Phase 3：编写 Playwright 测试脚本

1. 在 `frontend/tests/` 下创建 `<模块名>.spec.ts`
2. 每个用例 = 一个 `test()` 块，包含操作步骤 + `expect` 断言
3. 需要登录的页面，使用 `login()` 辅助函数（见模板）
4. 运行命令：

```bash
cd frontend && npx playwright test tests/<模块名>.spec.ts --reporter=html
```

**截图策略：** 依赖 `playwright.config.js` 中配置的 `screenshot: 'on'` 和 `trace: 'on'`，由 Playwright 自动为每个用例截图和录制 trace，无需手动写截图代码。所有截图和 trace 直接内嵌在 HTML 报告中查看。

**测试文件模板：**

```ts
import { test, expect } from '@playwright/test';

// ---------- 辅助函数 ----------

/** 登录并跳转到指定页面 */
async function login(page, targetPath = '/') {
  await page.goto('/login');
  await page.fill('input[placeholder*="用户名"], input[name="username"]', 'testuser');
  await page.fill('input[placeholder*="密码"], input[name="password"]', 'Test123456');
  await page.click('button:has-text("登录")');
  // 等待登录完成，根据实际项目调整
  await page.waitForURL(/\/(dashboard|admin)/, { timeout: 10000 });
  if (targetPath !== '/') {
    await page.goto(targetPath);
  }
}

// ---------- 测试套件 ----------

test.describe('模块名', () => {
  test('P0 - 正常用例: xxx', async ({ page }) => {
    await login(page, '/module-path');
    // 操作步骤
    await expect(page.locator('.success-msg')).toBeVisible();
  });

  test('P1 - 边界用例: xxx', async ({ page }) => {
    await login(page, '/module-path');
    // ...
  });
});
```

**关键注意事项：**

- 使用 `baseURL`（已配置 `http://localhost:3000`），`page.goto()` 写相对路径如 `/admin/users`
- 选择器优先用 `data-testid`，其次用 `placeholder` / `role` / `text`，避免依赖易变的 CSS 类名
- 涉及文件上传时，用 `page.setInputFiles()` 而非模拟点击
- 涉及等待数据加载时，用 `waitForResponse()` 或 `waitForSelector()` 而非 `page.waitForTimeout()`

### Phase 4：运行与调试

**首次运行：**

```bash
cd frontend && npx playwright test tests/<模块名>.spec.ts --reporter=html
```

**如有失败用例：**

1. 查看 trace：`npx playwright show-trace trace.zip` 或在 HTML 报告中点击失败用例查看
2. 只重跑失败的：`npx playwright test tests/<模块名>.spec.ts --last-failed`
3. 调试模式（会打开浏览器可视化操作）：`npx playwright test tests/<模块名>.spec.ts --debug`
4. 根据失败原因修复测试脚本或标记 bug

**注意：** `playwright.config.js` 已配置 `trace: 'on'` 和 `screenshot: 'on'`，无需额外修改。

### Phase 5：查看报告

```bash
cd frontend && npx playwright show-report
```

HTML 报告包含：
- 每个用例的执行状态（通过/失败/跳过）
- 每步操作的 trace 录制
- 网络请求与响应详情
- 控制台日志与错误
- 每个用例的自动截图（通过 + 失败均有）
- 失败用例的专属失败截图

### Phase 6：输出摘要

在对话中输出 Markdown 格式的测试结论摘要：

```markdown
## 测试摘要 — <模块名>

| 指标 | 值 |
|------|-----|
| 总用例数 | X |
| 通过 | X |
| 失败 | X |
| 跳过 | X |
| 耗时 | Xs |

### 失败用例
- **P0 - 用例名**: 失败原因简述

### 主要发现
1. ...
2. ...

### 建议
1. ...
```

## 用例设计规则

| 代码特征 | 最少加几条 | 示例 |
|---------|----------|------|
| 每个 API 端点 | +1 | GET /api/users 正常返回 |
| 端点含路径参数 | +1 | GET /api/users/999999（不存在 ID） |
| 有可选参数 `required=false` | +1 | 缺省参数时用默认值 |
| 有分页参数 | +1 | page=0 正常 / page=999999 空结果 |
| 有文件上传 | +2 | 超限文件 + 非法格式文件 |
| 有搜索/关键词 | +1 | 特殊字符 `<>""` + 空关键词 |
| 有鉴权注解/路由守卫 | +1 | 未登录访问跳转登录页 |
| 有角色区分 | +1 | 低权限用户访问管理页面 → 403/重定向 |
| 有状态字段（如 status） | +状态数 | 遍历每种状态值 |
| 调用外部 API | +1 | 模拟超时/异常的降级处理 |
| 有批量操作 | +1 | 空列表 + 部分项失败 |
| 逻辑删除 | +1 | 删除后再查询 + 重复删除 |
| 表单提交 | +2 | 必填项为空 + 格式非法 |
| 下拉/多选项 | +1 | 空选择 + 全选 |

## 原则

1. **测试数据写具体** — 不用 `testData`，用 `'admin@test.com'`
2. **每个用例只测一件事** — 一个 test() 只验证一个行为
3. **用例描述含优先级** — `P0 - 正常登录` / `P1 - 超长用户名`
4. **断言要明确** — 用 `toBeVisible()` / `toHaveText()` / `toHaveURL()`，不用 `toBeTruthy()`
5. **不依赖执行顺序** — 每个 test() 独立，不靠前一个用例的状态
6. **选择器稳定优先** — `data-testid` > `role` > `placeholder` > `text` > CSS class
