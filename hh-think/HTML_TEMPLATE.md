# HTML Template Reference

Base structure for idea-validation HTML output. Adapt per problem type.

## Minimal Template

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>idea-{topic}-{date}</title>
<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
<style>
  body { font-family: system-ui, sans-serif; max-width: 960px; margin: 2rem auto; padding: 0 1rem; }
  h1 { font-size: 1.4rem; }
  h2 { font-size: 1.1rem; margin-top: 1.5rem; }
  .diagram { background: #f8f8f8; border-radius: 8px; padding: 1rem; margin: 1rem 0; }
  .note { color: #666; font-size: 0.9rem; line-height: 1.6; }
  .tag { display: inline-block; background: #e8e8e8; border-radius: 4px; padding: 2px 8px; margin: 2px; font-size: 0.85rem; }
  .conclusion { background: #e6f3e6; border-radius: 8px; padding: 1rem; margin-top: 2rem; }
</style>
</head>
<body>

<h1>{Topic} 方案审核</h1>
<p class="note">生成日期: {date} | 状态: 待审核</p>

<h2>核心问题</h2>
<p class="note">{1-2 lines describing what the user is really asking}</p>

<h2>{Diagram Section Title}</h2>
<div class="diagram">
<pre class="mermaid">
{mermaid diagram code}
</pre>
</div>

<p class="note">{1-2 lines of annotation, not a paragraph}</p>

<h2>结论</h2>
<div class="conclusion">
<p>{Direct answer: is the idea sound? What's the recommended path? Hard risks?}</p>
</div>

<script>mermaid.initialize({startOnLoad:true});</script>
</body>
</html>
```

## Diagram Type Selection

| Problem Type | Mermaid Diagram | Example |
|---|---|---|
| Process/flow | `flowchart` | Data pipeline steps |
| Interactions/sequence | `sequenceDiagram` | API call ordering |
| Dependencies | `graph` (LR/TB) | Module dependency map |
| Decision/choices | `flowchart` with diamond nodes | A vs B decision tree |
| Comparison | Markdown table + `flowchart` per option | Two architecture options side-by-side |

## Adding Interactivity (when complexity warrants)

### Tab-based option comparison

```html
<style>
  .tabs { display: flex; gap: 0.5rem; margin-bottom: 0.5rem; }
  .tab { padding: 6px 16px; border: 1px solid #ccc; border-radius: 4px; cursor: pointer; }
  .tab.active { background: #333; color: #fff; }
  .panel { display: none; }
  .panel.active { display: block; }
</style>

<div class="tabs">
  <div class="tab active" onclick="show('opt-a')">方案 A</div>
  <div class="tab" onclick="show('opt-b')">方案 B</div>
</div>

<div id="opt-a" class="panel active">
  <pre class="mermaid">{diagram A}</pre>
</div>
<div id="opt-b" class="panel">
  <pre class="mermaid">{diagram B}</pre>
</div>

<script>
function show(id) {
  document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  event.target.classList.add('active');
  mermaid.run();
}
</script>
```

### Collapsible details

```html
<details>
  <summary>展开: {brief label}</summary>
  <pre class="mermaid">{sub-diagram}</pre>
</details>
```

## Opening Browser

After writing the HTML file, run:

```bash
open docs/tmp/idea-{topic}-{date}.html
```