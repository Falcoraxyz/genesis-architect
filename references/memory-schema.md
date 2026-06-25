# Memory System — Supabase Schema & Operations

## Database Setup

Run this SQL in the Supabase SQL editor once. Everything else is managed by the TypeScript layer.

```sql
-- Projects: every build session that resulted in something real
create table if not exists genesis_projects (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  description text,
  tech_stack  jsonb not null default '[]',
  core_decisions jsonb default '[]',
  files_created  jsonb default '[]',
  lessons        jsonb default '[]',
  status      text not null default 'active'
              check (status in ('active', 'archived', 'paused')),
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

-- Component library: reusable patterns extracted across projects
create table if not exists genesis_components (
  id                uuid primary key default gen_random_uuid(),
  name              text not null unique,
  description       text,
  code              text not null,
  language          text not null,
  tags              text[] not null default '{}',
  usage_count       integer not null default 1,
  source_project_id uuid references genesis_projects(id) on delete set null,
  last_used_at      timestamptz,
  created_at        timestamptz not null default now()
);

-- Decision log: every significant architectural fork
create table if not exists genesis_decisions (
  id                 uuid primary key default gen_random_uuid(),
  question           text not null,
  context            text,
  options_considered jsonb not null default '[]',
  chosen_option      text not null,
  reasoning          text not null,
  outcome            text,
  project_id         uuid references genesis_projects(id) on delete set null,
  created_at         timestamptz not null default now()
);

-- Lessons: failures, bottlenecks, and discoveries
create table if not exists genesis_lessons (
  id           uuid primary key default gen_random_uuid(),
  title        text not null,
  problem      text not null,
  root_cause   text,
  solution     text not null,
  prevention   text,
  severity     text not null default 'low'
               check (severity in ('low', 'medium', 'high', 'critical')),
  tags         text[] not null default '{}',
  project_id   uuid references genesis_projects(id) on delete set null,
  created_at   timestamptz not null default now()
);

-- Enable RLS on all tables
alter table genesis_projects  enable row level security;
alter table genesis_components enable row level security;
alter table genesis_decisions  enable row level security;
alter table genesis_lessons    enable row level security;

-- Service role bypass (memory operations use service key server-side)
create policy "service_full_access" on genesis_projects
  for all to service_role using (true) with check (true);
create policy "service_full_access" on genesis_components
  for all to service_role using (true) with check (true);
create policy "service_full_access" on genesis_decisions
  for all to service_role using (true) with check (true);
create policy "service_full_access" on genesis_lessons
  for all to service_role using (true) with check (true);

-- Auto-update updated_at on genesis_projects
create or replace function _genesis_touch_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger genesis_projects_updated_at
  before update on genesis_projects
  for each row execute function _genesis_touch_updated_at();

-- Indexes for common query patterns
create index genesis_projects_status    on genesis_projects(status);
create index genesis_components_tags    on genesis_components using gin(tags);
create index genesis_lessons_severity   on genesis_lessons(severity);
create index genesis_lessons_tags       on genesis_lessons using gin(tags);
```

---

## TypeScript Types

```typescript
// memory/types.ts

export type Project = {
  id: string
  name: string
  description: string | null
  tech_stack: string[]
  core_decisions: string[]
  files_created: string[]
  lessons: string[]
  status: 'active' | 'archived' | 'paused'
  created_at: string
  updated_at: string
}

export type Component = {
  id: string
  name: string
  description: string | null
  code: string
  language: string
  tags: string[]
  usage_count: number
  source_project_id: string | null
  last_used_at: string | null
  created_at: string
}

export type DecisionOption = {
  option: string
  pros: string[]
  cons: string[]
}

export type Decision = {
  id: string
  question: string
  context: string | null
  options_considered: DecisionOption[]
  chosen_option: string
  reasoning: string
  outcome: string | null
  project_id: string | null
  created_at: string
}

export type Lesson = {
  id: string
  title: string
  problem: string
  root_cause: string | null
  solution: string
  prevention: string | null
  severity: 'low' | 'medium' | 'high' | 'critical'
  tags: string[]
  project_id: string | null
  created_at: string
}

export type Brain = {
  projects: Project[]
  components: Component[]
  decisions: Decision[]
  lessons: Lesson[]
  snapshot_at: string
}
```

---

## Supabase Client (server-side only)

```typescript
// memory/client.ts
import { createClient } from '@supabase/supabase-js'
import { env } from '@/config/env'

// Service role key — never expose to the browser
export const memoryClient = createClient(
  env.SUPABASE_URL,
  env.SUPABASE_SERVICE_KEY,
  { auth: { persistSession: false } }
)
```

---

## Memory Operations

```typescript
// memory/operations.ts
import { memoryClient } from './client'
import type { Brain, Project, Component, Decision, Lesson } from './types'

// Pull everything from Supabase into a Brain snapshot
export async function pullBrain(): Promise<Brain> {
  const [projects, components, decisions, lessons] = await Promise.all([
    memoryClient.from('genesis_projects').select('*').order('updated_at', { ascending: false }),
    memoryClient.from('genesis_components').select('*').order('usage_count', { ascending: false }),
    memoryClient.from('genesis_decisions').select('*').order('created_at', { ascending: false }),
    memoryClient.from('genesis_lessons').select('*').order('severity', { ascending: false }),
  ])

  if (projects.error) throw projects.error
  if (components.error) throw components.error
  if (decisions.error) throw decisions.error
  if (lessons.error) throw lessons.error

  return {
    projects: projects.data,
    components: components.data,
    decisions: decisions.data,
    lessons: lessons.data,
    snapshot_at: new Date().toISOString(),
  }
}

// Register a new project at session start
export async function registerProject(
  data: Omit<Project, 'id' | 'created_at' | 'updated_at'>
): Promise<Project> {
  const { data: project, error } = await memoryClient
    .from('genesis_projects')
    .insert(data)
    .select()
    .single()
  if (error) throw error
  return project
}

// Update a project at session end (append lessons, decisions, files)
export async function updateProject(
  id: string,
  patch: Partial<Pick<Project, 'lessons' | 'core_decisions' | 'files_created' | 'status'>>
): Promise<void> {
  const { error } = await memoryClient
    .from('genesis_projects')
    .update(patch)
    .eq('id', id)
  if (error) throw error
}

// Add a component to the library (upsert by name — increment usage if exists)
export async function addComponent(
  data: Omit<Component, 'id' | 'usage_count' | 'created_at'>
): Promise<Component> {
  // Check if it exists first
  const { data: existing } = await memoryClient
    .from('genesis_components')
    .select('id, usage_count')
    .eq('name', data.name)
    .maybeSingle()

  if (existing) {
    const { data: updated, error } = await memoryClient
      .from('genesis_components')
      .update({ usage_count: existing.usage_count + 1, last_used_at: new Date().toISOString() })
      .eq('id', existing.id)
      .select()
      .single()
    if (error) throw error
    return updated
  }

  const { data: created, error } = await memoryClient
    .from('genesis_components')
    .insert({ ...data, last_used_at: new Date().toISOString() })
    .select()
    .single()
  if (error) throw error
  return created
}

// Log an architectural decision
export async function logDecision(data: Omit<Decision, 'id' | 'created_at'>): Promise<Decision> {
  const { data: decision, error } = await memoryClient
    .from('genesis_decisions')
    .insert(data)
    .select()
    .single()
  if (error) throw error
  return decision
}

// Record a lesson learned
export async function recordLesson(data: Omit<Lesson, 'id' | 'created_at'>): Promise<Lesson> {
  const { data: lesson, error } = await memoryClient
    .from('genesis_lessons')
    .insert(data)
    .select()
    .single()
  if (error) throw error
  return lesson
}

// Full-text search across all memory (returns matched items per category)
export async function searchMemory(query: string): Promise<{
  projects: Project[]
  components: Component[]
  lessons: Lesson[]
}> {
  const [projects, components, lessons] = await Promise.all([
    memoryClient
      .from('genesis_projects')
      .select('*')
      .or(`name.ilike.%${query}%,description.ilike.%${query}%`),
    memoryClient
      .from('genesis_components')
      .select('*')
      .or(`name.ilike.%${query}%,description.ilike.%${query}%`),
    memoryClient
      .from('genesis_lessons')
      .select('*')
      .or(`title.ilike.%${query}%,problem.ilike.%${query}%,solution.ilike.%${query}%`),
  ])

  return {
    projects: projects.data ?? [],
    components: components.data ?? [],
    lessons: lessons.data ?? [],
  }
}
```

---

## Sync Script (run at session start and end)

```typescript
// memory/sync.ts
import { writeFileSync, readFileSync, existsSync, mkdirSync } from 'fs'
import { pullBrain } from './operations'
import type { Brain } from './types'

const BRAIN_PATH = './memory/brain.json'

// Pull from Supabase, write to disk
export async function syncBrain(): Promise<Brain> {
  const brain = await pullBrain()

  if (!existsSync('./memory')) mkdirSync('./memory', { recursive: true })
  writeFileSync(BRAIN_PATH, JSON.stringify(brain, null, 2))

  console.log(
    `Brain synced — ${brain.projects.length} projects, ` +
    `${brain.components.length} components, ` +
    `${brain.lessons.length} lessons`
  )

  return brain
}

// Load from disk (no network call — for use inside build tools)
export function loadLocalBrain(): Brain {
  if (!existsSync(BRAIN_PATH)) {
    return { projects: [], components: [], decisions: [], lessons: [], snapshot_at: '' }
  }
  return JSON.parse(readFileSync(BRAIN_PATH, 'utf-8')) as Brain
}

// CLI usage: npx tsx memory/sync.ts [--generate-map]
if (require.main === module) {
  const generateMap = process.argv.includes('--generate-map')
  syncBrain().then(async () => {
    if (generateMap) {
      const { generateBrainMap } = await import('./generate-brainmap')
      generateBrainMap()
    }
  })
}
```

---

## Session-End Extraction Prompt (internal)

Before ending any session, Claude should extract and store these from the conversation:

```
EXTRACT → Component Library:
  Any function, hook, or pattern that appeared more than once
  Any utility that's generic enough to be reused without modification

EXTRACT → Decision Log:
  Any choice between two or more approaches where reasoning mattered
  Any "should we use X or Y" question that was answered with trade-off analysis

EXTRACT → Lessons:
  Anything that broke or was slower than expected
  Any incorrect assumption that had to be corrected mid-build
  Any library or pattern that was swapped out and why

EXTRACT → Project Registry update:
  Final list of files created
  Actual tech stack used (may differ from initial plan)
  Summary of what the project does in one sentence
```
