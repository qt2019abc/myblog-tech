
## 前言

在日常开发中，代码审查（Code Review）是保障代码质量、规范团队协作的核心环节，但人工 CR 存在**效率低、标准不统一、耗时耗力**等问题。结合 Google Gemini 官方 Code Reviewer 能力，针对 **GitLab 代码仓库** + **AI 编码 IDE（Qoder/Cursor 等）** 场景，整理一套可直接落地的 AI Code Review 方案，包含标准化 Skill 配置、IDE 集成、GitLab MR 自动化触发全流程，开箱即用。

## 一、方案核心定位

本方案基于 `google-gemini/code-reviewer` 官方能力优化，**专为 GitLab 场景定制**，支持两大核心场景：

1. **本地开发实时审查**：AI IDE 中写完代码即时检测，提前规避问题
2. **GitLab MR 自动化审查**：提交合并请求后自动触发 AI 审查，输出结构化报告
    
    最终实现**开发即审查、提交即校验**，统一团队代码规范，降低人工 CR 成本。

## 二、核心依赖：GitLab 专用 Code Review Skill

这是 AI 执行代码审查的**核心规则契约**，直接作为 `SKILL.md` 使用，适配 GitLab Merge Request 触发机制，兼容所有大模型 / AI IDE。

markdown

```
# Skill: GitLab 自动化代码审查
## 基础信息
名称：gitlab-code-reviewer
适配平台：GitLab
触发场景：GitLab Merge Request 创建/更新、本地代码变更审查
核心目标：保障代码正确性、安全性、可维护性，严格遵循项目规范，输出可落地的审查意见

## 审查范围
1. 仅审查 GitLab MR 中的增量变更代码（diff 内容），不审查全量历史代码
2. 支持本地 Git 已暂存/未暂存代码变更审查
3. 自动读取 MR 标题、描述、关联需求，理解业务上下文

## 审查维度（优先级从高到低）
1. 安全性：硬编码密钥、SQL注入、XSS、权限漏洞、输入未校验
2. 正确性：逻辑Bug、空指针、边界条件遗漏、异常处理缺失
3. 可维护性：命名不规范、函数过长、重复代码、注释缺失
4. 性能优化：低效循环、N+1查询、资源未释放
5. 代码规范：严格对齐项目既定规范（如PEP8、Eslint）

## 输出规范
1. 格式：Markdown 结构化，可直接作为 GitLab MR 评论
2. 每条意见必须包含：文件路径+行号、问题描述、风险影响、修复建议（带代码示例）
3. 结论明确：通过审查 / 需修复后合并 / 建议优化
4. 语气专业建设性，不主观臆断，不纠结个人编码偏好

## 禁用规则
1. 不审查与本次变更无关的历史代码
2. 不提出无依据的优化建议，业务逻辑不确定时标注「需人工确认」
3. 不指责开发者，聚焦代码问题而非个人
```

## 三、方案一：GitLab MR 自动化 AI 审查（核心生产场景）

### 1. 工作流程

开发者提交 GitLab Merge Request → GitLab Webhook/CI 触发 → AI 调用上述 Skill 审查代码 → 审查结果自动评论到 MR 页面 → 开发者根据意见修复 → 人工 CR 聚焦核心逻辑

### 2. 落地配置（GitLab CI 最简模板）

在项目根目录创建 `.gitlab-ci.yml`，集成 AI 审查服务：

yaml

```
# AI Code Review 自动化任务
ai_code_review:
  stage: test
  only:
    - merge_requests  # 仅MR触发
  script:
    # 调用AI审查接口，传入MR信息和代码diff
    - python3 ai_review.py --project $CI_PROJECT_PATH --mr-iid $CI_MERGE_REQUEST_IID --diff $CI_MERGE_REQUEST_DIFF
  # 审查结果自动评论到GitLab MR
  after_script:
    - curl -X POST -H "PRIVATE-TOKEN: $GITLAB_TOKEN" -d "body=$(cat review_result.md)" "$CI_API_V4_URL/projects/$CI_PROJECT_ID/merge_requests/$CI_MERGE_REQUEST_IID/notes"
```

### 3. 优势

- 无需人工操作，全自动触发
- 审查标准统一，所有 MR 遵循同一套规则
- 提前拦截高危问题，降低线上故障风险

## 四、方案二：AI IDE 集成（Qoder/Cursor 本地实时审查）

将 Skill 与 AI 编码 IDE 结合，实现**开发过程中即时审查**，不用等到提交 MR 再发现问题。

### 1. 支持工具

Qoder、Cursor、GitHub Copilot、CodeWhisperer 等支持自定义技能 / 提示词的 AI IDE

### 2. Qoder 集成步骤（最通用）

1. **创建自定义技能**
    
    在 Qoder 中新建技能，命名为「GitLab 代码审查」，将上述 `SKILL.md` 完整粘贴为系统提示词。
    
2. **一键触发审查**
    
    - 单文件审查：选中代码 → 右键 → 运行「GitLab 代码审查」
    - 全量变更审查：命令行输入 `@qoder review staged`，自动检测本地 Git 变更
    
3. **实时预览结果**
    
    AI 直接在编辑器内输出 inline 建议、侧边栏报告，支持一键修复代码。
    

### 3. 进阶：Pre-Commit 本地拦截

配置 Git 提交钩子，**未通过 AI 审查禁止提交代码**，从源头阻断问题代码：

bash

运行

```
# .git/hooks/pre-commit
#!/bin/bash
# 调用Qoder执行AI审查
qoder skill "GitLab代码审查" --diff $(git diff --staged) > review.log
# 检测高危问题，拦截提交
if grep -E "安全漏洞|逻辑错误" review.log; then
  echo "❌ 代码存在高危问题，禁止提交，请修复后重试"
  exit 1
fi
```

## 五、方案对比与价值总结

表格

|场景|传统人工 CR|本 AI Code Review 方案|
|---|---|---|
|效率|耗时 1-2 小时 / 次|1-3 分钟自动完成|
|标准|因人而异，不统一|全团队严格遵循同一套 Skill|
|时机|提交 MR 后审查|开发中 + 提交前 + MR 后全流程覆盖|
|成本|资深工程师大量精力|初级问题 AI 处理，人工聚焦核心|

### 核心价值

1. **降本提效**：减少 80% 以上的初级 CR 工作量，开发效率提升 50%+
2. **质量保障**：标准化审查，安全、逻辑问题提前拦截
3. **无缝衔接**：本地 IDE + GitLab 全流程打通，无额外学习成本
4. **可扩展性**：Skill 可根据团队规范自定义，适配所有编程语言

## 六、快速落地指南

1. 复制本文 `SKILL.md`，作为 AI 代码审查的核心规则
2. 配置 GitLab CI，实现 MR 自动化审查
3. 在 AI IDE（Qoder/Cursor）中导入自定义技能，开启本地实时审查
4. 配置 Pre-Commit 钩子，实现提交拦截

---

## 总结

1. 本方案基于 Google Gemini 官方 Code Reviewer 优化，**GitLab 专属、可直接落地**
2. 提供标准化 `SKILL.md`，统一 AI 审查规则，适配所有大模型 / AI IDE
3. 两大核心能力：**GitLab MR 自动化审查** + **AI IDE 本地实时审查**
4. 低成本、高效率，完美解决团队代码审查痛点