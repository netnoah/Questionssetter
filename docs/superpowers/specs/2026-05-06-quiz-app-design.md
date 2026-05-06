# AI Quiz Generator — Design Spec

## Overview

An AI-powered quiz generation application running as a Claude Code custom skill. Users trigger question generation via slash commands (e.g., `/出题 袋鼠数学 五年级面积题 10`), and an interactive self-contained HTML file opens in the browser for answering.

## Architecture

```
User input: /出题 袋鼠数学 五年级面积题 10
                |
         Claude Code Skill parses parameters
                |
         +---------------------+
         | Question Type Config |  <- YAML file defining rules
         | kangaroo-math.yaml   |
         +----------+----------+
                    |
         AI generates structured JSON per config
                    |
         JSON injected into HTML template
                    |
         Self-contained HTML opens in browser
```

### Project Structure

```
Questionssetter/
├── .claude/commands/          # Claude Code custom commands
│   └── 出题.md               # Skill prompt definition
├── question-types/           # Question type configurations
│   └── kangaroo-math.yaml    # Kangaroo Math sample
├── templates/                # HTML templates
│   └── quiz.html             # Interactive quiz page template
├── output/                   # Generated HTML files (shareable)
└── docs/                     # Design documents
```

### Core Components

1. **Skill Command** (`/出题`): Parses arguments — question type, topic, count
2. **Question Type Config** (YAML): Per-type prompt templates, output schema, UI overrides
3. **HTML Template**: Generic interactive quiz page accepting JSON data
4. **Generation Logic**: Skill prompt guiding AI to generate questions per type rules

## Command Format and Argument Parsing

### Format

```
/出题 <题型名称> <主题> [数量]
```

- Default count: 10

### Parsing Rules

Arguments are passed as a single string. The skill prompt parses them as follows:

1. **Split the input by spaces**
2. **Identify the question type**: Match the first token against the `name` field in all YAML files under `question-types/`. If no match, try combining first two tokens. Matching is exact (not partial) against the `name` field in YAML.
3. **Identify the count**: Scan tokens from right to left. The rightmost token that is a pure integer (e.g., `10`, `5`, `20`) is the count. If no integer token found, default to 10.
4. **The remaining tokens between the type and count (or end of input)** form the topic. If no topic remains (e.g., `/出题 袋鼠数学`), use the type's `description` field as a fallback prompt hint.

**Examples**:

| Input | Type | Topic | Count |
|-------|------|-------|-------|
| `袋鼠数学 五年级面积题 10` | 袋鼠数学 | 五年级面积题 | 10 |
| `袋鼠数学 五年级面积题` | 袋鼠数学 | 五年级面积题 | 10 (default) |
| `袋鼠数学 五年级 面积题` | 袋鼠数学 | 五年级 面积题 | 10 (default) |
| `袋鼠数学 五年级面积题 5` | 袋鼠数学 | 五年级面积题 | 5 |

### Error: Type Not Found

If no type matches, list all available type names from YAML files:

```
未找到题型 "xxx"。可用的题型有：
- 袋鼠数学
- 小学语文
请重新输入。
```

## Question Type Configuration

Each question type is a YAML file in `question-types/`. Adding a new type requires only a new YAML file — zero code changes.

### YAML Schema

```yaml
# Required fields
name: string           # Display name, used for command matching
description: string    # Short description
icon: string           # Emoji or character for UI header

# Prompt configuration
prompt_template: string  # Template with {topic}, {count}, {output_format} placeholders

# Output format definition (informal schema used in prompt)
output_format:
  description: string    # Human-readable description of required JSON structure
  example: string        # Example JSON output showing the exact structure

# Optional UI overrides
ui:
  primary_color: string   # Hex color for theme (default: "#4A90D9")
  option_count: number    # Number of options per question (default: 5)
  option_style: string    # "horizontal" or "vertical" (default: "vertical") — desktop only, mobile always vertical
```

### Example: `kangaroo-math.yaml`

```yaml
name: 袋鼠数学
description: 袋鼠数学竞赛风格题目
icon: "🦘"

prompt_template: |
  你是一位袋鼠数学竞赛出题专家。请根据以下要求出题：
  主题：{topic}
  数量：{count} 道选择题（每题 5 个选项，A-E）

  出题要求：
  - 题目风格模仿袋鼠数学竞赛：趣味性强、贴近生活、侧重逻辑思维
  - 难度适合对应年级
  - 需要图文配合的题用 SVG 绘制配图（SVG 放在 image 字段）
  - 每题提供详细解析和解题思路（放在 hint 字段）

  按以下格式输出一个 JSON 数组，不要输出任何其他内容：
  {output_format}

output_format:
  description: |
    一个 JSON 数组，每个元素包含：
    - id: 题号（从1开始）
    - question: 题目文本（可包含 HTML 标签用于数学公式）
    - image: SVG 代码字符串（可选，无配图时不要包含此字段）
    - options: 选项数组，每项包含 label（A-E）和 text（选项内容）
    - answer: 正确答案的 label
    - hint: 解析提示（可包含 HTML）
  example: |
    [
      {
        "id": 1,
        "question": "一个长方形的长是12米，宽是8米，面积是多少？",
        "options": [
          {"label": "A", "text": "20平方米"},
          {"label": "B", "text": "40平方米"},
          {"label": "C", "text": "96平方米"},
          {"label": "D", "text": "48平方米"},
          {"label": "E", "text": "84平方米"}
        ],
        "answer": "C",
        "hint": "长方形面积 = 长 × 宽 = 12 × 8 = 96平方米"
      }
    ]

ui:
  primary_color: "#4A90D9"
  option_count: 5
```

### Validation Strategy

Since this runs as a Claude Code skill prompt (not a program), validation is **prompt-based**:

1. The skill prompt instructs the AI to first output the JSON, then self-check
2. The skill prompt includes explicit validation rules: check `id` is sequential, `answer` matches one option label, `options` has the correct count
3. On receiving the output, the skill prompt instructs the AI to parse the JSON and verify structure before proceeding
4. If the AI cannot produce valid JSON after 2 retries, report the error to the user

## Skill Prompt Specification

### `.claude/commands/出题.md` Structure

The skill prompt is the orchestration layer. It receives the user's input string and executes the following steps using Claude Code tools (Read, Write, Bash):

**Step-by-step execution**:

1. **Parse arguments** from `$ARGUMENTS` using the parsing rules above
2. **Read question type config**: Use `Read` tool to load the matching YAML file from `question-types/`
3. **Assemble prompt**: Replace `{topic}`, `{count}`, and `{output_format}` in the `prompt_template`
4. **Generate questions**: Act as the quiz expert and generate questions following the assembled prompt. Output only valid JSON.
5. **Validate output**: Self-check the JSON:
   - Is it a valid JSON array?
   - Does each item have `id`, `question`, `options`, `answer`, `hint`?
   - Are `id` values sequential starting from 1?
   - Does each `answer` match one of the option labels?
   - If any check fails, regenerate (max 2 retries)
6. **Read HTML template**: Use `Read` tool to load `templates/quiz.html`
7. **Inject data**: Replace template placeholders with actual values:
   - `<!--QUIZ_META-->` with a JSON object containing `title`, `icon`, `primary_color`
   - `<!--QUIZ_DATA-->` with the generated questions JSON array
8. **Ensure output directory exists**: Use `Bash` to create `output/` if missing
9. **Write output file**: Use `Write` tool to save as `output/<type>_<topic>_<timestamp>.html`. Sanitize filename: replace spaces with underscores, keep Chinese characters as-is, use format `YYYYMMDD_HHmmss` for timestamp to ensure uniqueness.
10. **Open in browser**: Use `Bash` to run `start output/<filename>.html` (Windows) to open the file

## Interactive HTML Page

### Layout (One Question at a Time with Navigation)

```
+------------------------------------------+
|  🦘 袋鼠数学 · 五年级面积题     第 3/10 题  |
+------------------------------------------+
|                                          |
|  3. 一个长方形花圃的长是12米，             |
|     宽是8米。如果在花圃四周               |
|     铺一条1米宽的小路，小路的              |
|     面积是多少平方米？                     |
|                                          |
|  +----------------------------+           |
|  |    [SVG Geometry Figure]    |           |
|  +----------------------------+           |
|                                          |
|  ○ A. 36平方米                           |
|  ○ B. 40平方米                           |
|  ○ C. 42平方米                           |
|  ● D. 44平方米  <- selected              |
|  ○ E. 48平方米                           |
|                                          |
|  ✅ 回答正确！答案是 D                    |
|                                          |
|  ▶ 查看解题提示（点击展开）               |
|  +----------------------------+           |
|  | Detailed step-by-step       |           |
|  | solution explanation        |           |
|  +----------------------------+           |
|                                          |
+------------------------------------------+
| [1] [2] [3] [4] [5] [6] [7] [8] [9] [10] |
|  ✓   ✓   ●   ·   ·   ·   ·   ·   ·   ·  |
|               [← Prev] [Next →]           |
+------------------------------------------+
```

### Interaction Behaviors

| Action | Behavior |
|--------|----------|
| Select option | Immediately show correct/incorrect, lock the question |
| Correct feedback | Green highlight + "回答正确！" |
| Incorrect feedback | Red highlight + show correct answer |
| Hint | Collapsed by default; click to expand; shows detailed solution |
| Navigation | Bottom question number buttons; ✓/✗ for answered; highlight current; gray for unanswered |
| SVG images | Embedded directly in HTML, sanitized before rendering |

### SVG Security

SVG content is sanitized before rendering via innerHTML:

- Allowlist of safe SVG elements: `svg`, `path`, `rect`, `circle`, `ellipse`, `line`, `polyline`, `polygon`, `text`, `tspan`, `g`, `defs`, `linearGradient`, `radialGradient`, `stop`, `title`, `desc`
- Strip all `on*` event attributes (onclick, onload, etc.)
- Strip `<script>` tags
- Strip `<foreignObject>` elements
- Strip `javascript:` URLs in href/xlink:href attributes
- Require `viewBox` attribute on root `<svg>`

### Responsive Design

| Breakpoint | Layout Changes |
|------------|---------------|
| Desktop (>768px) | Options displayed as per YAML `option_style`; full navigation bar |
| Mobile (≤768px) | Options always vertical; navigation bar becomes horizontal scroll; SVG scales to 100% width; touch targets minimum 44px height |

Additional mobile considerations:
- Navigation buttons scroll horizontally when >10 questions
- Option buttons have minimum 44px touch target height
- SVG images scale to container width with `max-width: 100%`

### Statistics Report Page (After All Questions)

Displayed after all questions are answered.

```
+------------------------------------------+
|           🎉 答题报告                     |
|                                          |
|       正确：8/10    正确率：80%           |
|       ████████░░                          |
|                                          |
|  +-- 题目回顾 ------------------------+  |
|  | 1. ✅ 长方形面积计算                |  |
|  | 2. ✅ 三角形面积比较                |  |
|  | 3. ❌ 小路面积 → D (你选了 B)       |  |
|  | ...                                 |  |
|  +-------------------------------------+  |
|                                          |
|  [重新做错题]  [导出为PDF]                |
+------------------------------------------+
```

**Button behaviors**:

- **重新做错题 (Retry wrong answers)**: Filters the quiz to only show incorrectly answered questions. Resets the answer state for those questions. After re-completing, shows an updated report. Implemented purely in JS by filtering the questions array.
- **导出为PDF (Export as PDF)**: Triggers `window.print()` with a print stylesheet. Print stylesheet: hide navigation bar, show all questions in sequence, show answers and hints expanded, hide interactive buttons.
- **[再来一套] removed**: Since the HTML is self-contained and cannot invoke Claude Code, this button is not included. Users should re-run the `/出题` command for a new set.

### Accessibility

- Semantic HTML: `<main>`, `<section>`, `<button>`, `<fieldset>/<legend>` for options
- ARIA labels on navigation buttons and question indicators
- Keyboard navigation: Tab through options, Enter to select, arrow keys for prev/next
- Color is not the only indicator of correct/incorrect (icons and text also used)
- `<noscript>` fallback: "请启用 JavaScript 以使用此答题页面"

## HTML Template Design

### Template Structure (`templates/quiz.html`)

The template uses HTML comment markers for injection (avoids collision with user content):

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><!--QUIZ_TITLE--></title>
  <style>
    :root {
      --primary: <!--QUIZ_COLOR-->;
      --correct: #22c55e;
      --wrong: #ef4444;
    }
    /* ... all styles ... */
    /* Print stylesheet */
    @media print {
      .navigation, .action-buttons { display: none; }
      .question { page-break-after: always; }
      .hint-content { display: block !important; }
    }
  </style>
</head>
<body>
  <noscript>请启用 JavaScript 以使用此答题页面</noscript>
  <div id="app"></div>

  <script id="quiz-meta" type="application/json"><!--QUIZ_META--></script>
  <script id="quiz-data" type="application/json"><!--QUIZ_DATA--></script>

  <script>
    // State, Render, Actions, Report modules
    // All questions data loaded from #quiz-data
    // Metadata (title, icon, color) from #quiz-meta
  </script>
</body>
</html>
```

### Template Variables

| Marker | Source | Description |
|--------|--------|-------------|
| `<!--QUIZ_TITLE-->` | `<type_name> · <topic>` | Page title |
| `<!--QUIZ_COLOR-->` | `ui.primary_color` from YAML | CSS variable value |
| `<!--QUIZ_META-->` | JSON `{title, icon, primary_color}` | UI metadata |
| `<!--QUIZ_DATA-->` | AI-generated questions JSON array | Quiz content |

### JS Module Organization (within single file)

| Module | Responsibility |
|--------|---------------|
| State | Manage quiz data, current question index, answer records, completion status |
| Render | Render question page or report page to `#app` based on state |
| Actions | Handle option click, hint expand, navigation jump, retry wrong answers |
| Report | Calculate accuracy, generate review list |
| Sanitize | Strip dangerous elements/attributes from SVG before innerHTML insertion |

## Generation Flow

```
User: /出题 袋鼠数学 五年级面积题 5
  |
  +-- 1. Parse args: type="袋鼠数学", topic="五年级面积题", count=5
  |
  +-- 2. Read question-types/kangaroo-math.yaml
  |
  +-- 3. Assemble prompt (replace {topic}, {count}, {output_format})
  |
  +-- 4. AI generates quiz JSON
  |     -> Self-validate: check fields, id sequence, answer validity
  |     -> Retry up to 2 times if invalid
  |
  +-- 5. Read templates/quiz.html
  |     -> Replace <!--QUIZ_TITLE--> with "袋鼠数学 · 五年级面积题"
  |     -> Replace <!--QUIZ_COLOR--> with "#4A90D9"
  |     -> Replace <!--QUIZ_META--> with metadata JSON
  |     -> Replace <!--QUIZ_DATA--> with questions JSON
  |
  +-- 6. Ensure output/ directory exists (mkdir if needed)
  |
  +-- 7. Write output/袋鼠数学_五年级面积题_20260506_143022.html
  |
  +-- 8. Execute: start output/袋鼠数学_五年级面积题_20260506_143022.html
        -> Browser opens automatically
```

### Error Handling

| Error | Response |
|-------|----------|
| Question type not found | List available types, ask user to retry |
| Invalid JSON from AI (after validation) | Auto-retry up to 2 times, then report error |
| File write failure | Check output directory permissions |
| SVG contains dangerous elements | Sanitize before rendering (strip script, on* attrs, foreignObject) |
| Output directory missing | Auto-create before writing |

## Design Decisions Summary

| Dimension | Decision |
|-----------|----------|
| Platform | Claude Code Skill + browser preview |
| Output | Self-contained single HTML file |
| Command format | `/出题 <type> <topic> [count]` |
| Argument parsing | Type by first token match, count as rightmost integer, rest is topic |
| Type extensibility | New YAML file, zero code |
| Default count | 10 questions |
| Display mode | One-at-a-time with free navigation |
| Feedback | Immediate on selection, then locked |
| Hints | Collapsed by default, click to expand |
| Graphics | AI-generated SVG, sanitized before rendering |
| Report | Accuracy + per-question review after completion |
| Retry wrong | Client-side JS filter, re-shows only wrong questions |
| PDF export | window.print() with print stylesheet |
| Dependencies | Zero — pure HTML/CSS/JS |
| Template injection | HTML comment markers to avoid collision |
| Validation | Prompt-based self-check + structural verification |
| Responsive | Mobile-first, breakpoints at 768px |
| Accessibility | Semantic HTML, ARIA, keyboard nav, noscript fallback |
