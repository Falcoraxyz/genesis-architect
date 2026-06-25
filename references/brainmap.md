# Brain Map — Galaxy Visualization

## What It Is

A standalone HTML file: `docs/brainmap.html`. No build step. No server. One file that opens in any browser and renders the entire accumulated knowledge graph as an interactive galaxy.

It reads from `memory/brain.json` at generation time. The JSON is embedded directly into the HTML — the file works offline.

---

## Node Types and Visual Rules

| Type | Color | Size | Glow |
|---|---|---|---|
| Project | `#00f5ff` (cyan) | 24px radius | Strong, pulsing |
| Component | `#b084ff` (purple) | 14px radius | Soft |
| Decision | `#ffb347` (amber) | 9px radius | Minimal |
| Lesson (low) | `#ff6b6b` (red) | 7px radius | None |
| Lesson (critical) | `#ff0000` (bright red) | 16px radius | Strong |

**Background**: `#020410` — deep space blue-black, not pure black.
**Edges**: `rgba(255, 255, 255, 0.12)` — barely visible connections between nodes.
**Selected node**: white outer ring, side panel opens.
**Font**: `'JetBrains Mono', 'Courier New', monospace` — labels at 11px, panel text at 13px.

---

## Data Model

```typescript
type GraphNode = {
  id: string
  label: string
  type: 'project' | 'component' | 'decision' | 'lesson'
  severity?: 'low' | 'medium' | 'high' | 'critical'  // lessons only
  data: Record<string, unknown>  // raw object from brain.json
  // assigned during layout:
  x: number
  y: number
  vx: number  // velocity for force simulation
  vy: number
}

type GraphEdge = {
  source: string  // node id
  target: string  // node id
  relation: 'uses' | 'derived_from' | 'solved_by' | 'led_to' | 'related'
}
```

---

## Generation Script

```typescript
// memory/generate-brainmap.ts
import { readFileSync, writeFileSync, existsSync, mkdirSync } from 'fs'
import { loadLocalBrain } from './sync'
import type { Brain } from './types'

type GraphNode = {
  id: string
  label: string
  type: string
  severity?: string
  data: Record<string, unknown>
}

type GraphEdge = {
  source: string
  target: string
  relation: string
}

export function generateBrainMap(): void {
  const brain = loadLocalBrain()

  const nodes: GraphNode[] = [
    ...brain.projects.map(p => ({
      id: p.id,
      label: p.name,
      type: 'project',
      data: p as unknown as Record<string, unknown>,
    })),
    ...brain.components.map(c => ({
      id: c.id,
      label: c.name,
      type: 'component',
      data: c as unknown as Record<string, unknown>,
    })),
    ...brain.decisions.map(d => ({
      id: d.id,
      label: d.question.length > 45 ? d.question.slice(0, 42) + '...' : d.question,
      type: 'decision',
      data: d as unknown as Record<string, unknown>,
    })),
    ...brain.lessons.map(l => ({
      id: l.id,
      label: l.title,
      type: 'lesson',
      severity: l.severity,
      data: l as unknown as Record<string, unknown>,
    })),
  ]

  const edges: GraphEdge[] = [
    ...brain.components
      .filter(c => c.source_project_id)
      .map(c => ({
        source: c.source_project_id as string,
        target: c.id,
        relation: 'derived_from',
      })),
    ...brain.decisions
      .filter(d => d.project_id)
      .map(d => ({
        source: d.project_id as string,
        target: d.id,
        relation: 'led_to',
      })),
    ...brain.lessons
      .filter(l => l.project_id)
      .map(l => ({
        source: l.project_id as string,
        target: l.id,
        relation: 'solved_by',
      })),
  ]

  const templatePath = './memory/brainmap-template.html'
  if (!existsSync(templatePath)) {
    console.error('Template not found at memory/brainmap-template.html — copy assets/galaxy-template.html there first')
    process.exit(1)
  }

  const template = readFileSync(templatePath, 'utf-8')
  const graphJson = JSON.stringify({ nodes, edges, meta: { snapshot_at: brain.snapshot_at } })
  const html = template.replace('__GRAPH_DATA__', graphJson)

  if (!existsSync('./docs')) mkdirSync('./docs', { recursive: true })
  writeFileSync('./docs/brainmap.html', html)

  console.log(
    `Brain map generated — ${nodes.length} nodes (${brain.projects.length} projects, ` +
    `${brain.components.length} components, ${brain.decisions.length} decisions, ` +
    `${brain.lessons.length} lessons), ${edges.length} edges`
  )
}

if (require.main === module) {
  generateBrainMap()
}
```

---

## Side Panel Content by Node Type

When a node is clicked, the panel renders the following. No markdown, no bullet soup — clean monospace sections.

**Project node**
```
[PROJECT NAME]
Status: active | archived | paused
Created: 2024-03-15

Tech stack:  Next.js 14 · TypeScript · Supabase · Vercel

Description:
[description text]

Files created:
  src/app/page.tsx
  src/server/db/posts.ts
  ...

Core decisions:
  [linked to decision nodes]

Lessons:
  [inline summary list]
```

**Component node**
```
[COMPONENT NAME]
Language: TypeScript   Used: 7 times
Source: [project name]
Tags: auth · server · middleware

Description:
[description text]

Code:
[syntax-highlighted code block]
```

**Decision node**
```
[QUESTION ASKED]
Project: [project name]

Context:
[context text]

Options considered:
  A: [option] — pros: [...] cons: [...]
  B: [option] — pros: [...] cons: [...]

Chosen: [chosen option]

Reasoning:
[reasoning text]

Outcome:
[outcome or "not yet observed"]
```

**Lesson node**
```
[TITLE]
Severity: CRITICAL | HIGH | MEDIUM | LOW
Project: [project name]
Tags: [tags]

Problem:
[problem description]

Root cause:
[root cause or "unknown"]

Solution:
[what was done]

Prevention:
[how to avoid this next time]
```

---

## Synthesize Button

Bottom-center of the canvas. Clicking it:

1. Aggregates all node data into a structured summary prompt
2. Sends to the Anthropic API (`claude-sonnet-4-6`, streaming)
3. Renders output in a terminal-style overlay panel
4. Output format (enforced via system prompt to the API):

```
KNOWLEDGE SYNTHESIS
Generated: [timestamp]

MOST REUSED COMPONENTS
[list by usage_count descending]

RECURRING PROBLEMS
[problems that appeared across 2+ projects]

STRONGEST PATTERNS
[tech decisions that have appeared consistently]

CRITICAL LESSONS
[all lessons with severity: critical or high]

RECOMMENDED NEXT ACTIONS
[3-5 concrete, specific suggestions based on the data]
```

No generic advice. Every recommendation must reference actual nodes in the graph.

---

## Force-Directed Layout Algorithm

Implemented in vanilla JS inside the HTML:

```javascript
// Parameters
const REPULSION = 8000      // between all node pairs
const ATTRACTION = 0.04     // spring constant for edges
const DAMPING = 0.85        // velocity decay per tick
const ITERATIONS = 400      // run to stability, then stop

// Initial positions: projects in center cluster, others distributed outward
// After layout, positions are frozen — user can drag nodes manually
```

Project nodes start near the center (within 150px of canvas center).
All other nodes start randomly distributed in a 600x600 area around the canvas center.
After 400 iterations, the layout stabilizes and animation switches to interaction-only mode.

---

## Setup Instructions (copy to README)

```bash
# Copy the template to the expected location
cp assets/galaxy-template.html memory/brainmap-template.html

# Sync brain from Supabase and generate map
npx tsx memory/sync.ts --generate-map

# Open the result
open docs/brainmap.html

# Or view it after Vercel deploy at:
# https://yourdomain.com/brainmap.html
```
