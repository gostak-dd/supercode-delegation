---
name: supercode-delegation
description: Orchestrate non-trivial feature work, refactors, and bug fixes through isolated product_workflow agents. Use when the user asks for supercode, delegation, orchestration, or a delegated feature workflow.
compatibility: Requires a harness exposing the product_workflow custom tool.
---

# Supercode delegation

Keep the parent session focused on orchestration. Delegate repository research, implementation, review, fixes, and verification to isolated agents through `product_workflow`. The parent may scope with quick file-name or text searches, then plans from compact research digests.

## Prerequisites

This skill requires the custom harness tool `product_workflow`. `subagent_spawn` may handle a single tiny delegation. `product_workflow_save` may persist a stable workflow. Tool installation is separate; this skill supplies instructions only.

## Model routing

The identifiers below are examples from the author's setup. They are not guaranteed to exist in your environment: check your harness's live model catalog and map every identifier (including any provider prefix your deployment requires) before use.

| Role | Model | Effort |
|---|---|---|
| Research | `gpt-6.1-terra` | high |
| Implementation and first fix | `gpt-6.1-sol` | high |
| Ambiguous implementation or failed first fix | `claude-opus-5-5` | high |
| Review and hard escalation | `claude-fable-5-2` | high |
| Optional second review | `gpt-6-astra` | high |
| Large, mechanical scan | `gpt-6-luna` | medium |

Use Terra for focused repository research. Use Luna only for large mechanical scans. Use Sol for clear implementation and the first fix attempt. Route ambiguous implementation and the second fix attempt to Opus. Use Fable for review and hard escalation. Add Astra as a sequential second reviewer when the change warrants another independent quality gate.

Always set `access.models` to the exact model ceiling for each workflow and map the identifiers first.

## Orchestrator rules

1. Delegate execution. Keep implementation file reads, edits, tests, reviews, and fixes inside agents.
2. Run repository research separately. Research agents return compact file paths, existing patterns, constraints, and risks. The parent turns those digests into an approved plan.
3. Pass the approved plan as `args.items` to the implementation workflow. Each item includes a name, specification, owned files, and check command.
4. Give every agent a self-contained prompt with the goal, scope, constraints, check command, and compact report shape.
5. Use schemas for every result that controls a branch. Read domain decisions only from validated structured fields.
6. Check `.ok` before reading `.structured` or `.output`. Treat every unsuccessful call as a failed delivery.
7. Permit at most two fix attempts. Run review after every fix attempt. A failed review or check produces a failed verdict.
8. Avoid concurrent edits to shared files. Parallel work is suitable only for disjoint ownership. Serialize overlapping work and finish with one full quality gate.
9. A failed workflow has no resume path. Re-run a suitably small workflow.

## `product_workflow` contract

- The definition and script are separate inputs.
- Scripts contain no imports or exports.
- Enter declared steps sequentially with `step()` using exact names.
- Call `agent()` only after entering a step.
- Check every agent result's `.ok` field.
- `access.models` is a ceiling; every requested model must appear there.
- `parallel()` has a maximum concurrency of 4. Use it only for disjoint ownership.
- Keep scripts deterministic. Avoid `Date` and `Math.random`.
- The final declared step owns the output slot, and the returned value must match the reported facts.
- Step descriptions state the work to accomplish.

## Standard flow

1. Run a research workflow with Terra, or Luna for a large mechanical scan.
2. Build an orchestrator-approved plan from the research digests.
3. Run the implementation workflow below with that plan in `args.items`.
4. Implement plan items sequentially unless ownership is demonstrably disjoint.
5. Review the complete diff. Apply at most two fixes, reviewing after each one.
6. Run the final full quality gate and return a structured verdict.

Expected arguments:

```json
{
  "items": [
    {
      "name": "short-name",
      "spec": "Complete implementation instructions",
      "files": ["relative/path"],
      "check": "project check command"
    }
  ],
  "qualityGate": "full project quality gate command"
}
```

## Compact implementation workflow

Definition:

```json
{
  "name": "approved-plan-delivery",
  "description": "Implement an approved plan, review corrections, and verify the result.",
  "instruction": "Deliver the researched and orchestrator-approved plan in args.items. Return a compact structured verdict.",
  "access": {
    "models": [
      "gpt-6.1-sol",
      "claude-opus-5-5",
      "claude-fable-5-2"
    ]
  },
  "steps": [
    {
      "name": "Implement",
      "description": "Apply each approved plan item and run its specified check."
    },
    {
      "name": "Review and fix",
      "description": "Review the complete diff, resolve confirmed defects, and confirm the corrected diff."
    },
    {
      "name": "Verify",
      "description": "Run the final full quality gate and return the delivery verdict.",
      "output": { "slot": "verdict" }
    }
  ]
}
```

Script:

```js
const IMPLEMENT = {
  type: 'object',
  properties: {
    status: { type: 'string', enum: ['pass', 'fail'] },
    summary: { type: 'string' },
    checks: { type: 'array', items: { type: 'string' } }
  },
  required: ['status', 'summary', 'checks']
}

const REVIEW = {
  type: 'object',
  properties: {
    status: { type: 'string', enum: ['pass', 'fail'] },
    findings: { type: 'array', items: { type: 'string' } },
    summary: { type: 'string' }
  },
  required: ['status', 'findings', 'summary']
}

const FIX = {
  type: 'object',
  properties: {
    status: { type: 'string', enum: ['pass', 'fail'] },
    summary: { type: 'string' }
  },
  required: ['status', 'summary']
}

const CHECK = {
  type: 'object',
  properties: {
    status: { type: 'string', enum: ['pass', 'fail'] },
    summary: { type: 'string' },
    failures: { type: 'array', items: { type: 'string' } }
  },
  required: ['status', 'summary', 'failures']
}

const implementations = []
const reviews = []
const fixes = []
let verification = { status: 'skipped', summary: 'The workflow has not reached verification.' }
const finish = (status, stage, reason, skipped = {}) => ({
  status,
  stage,
  reason,
  implementations,
  reviews,
  fixes,
  verification,
  skipped
})

step('Implement')
for (let index = 0; index < args.items.length; index += 1) {
  const item = args.items[index]
  const result = await agent(
    `Implement the approved plan item: ${item.name}\nSpecification: ${item.spec}\nOwned files: ${item.files.join(', ')}\nRun this check: ${item.check}\nReturn a compact structured result.`,
    { label: `implement:${item.name}`, schema: IMPLEMENT, model: 'gpt-6.1-sol', effort: 'high' }
  )
  if (!result.ok) {
    implementations.push({ name: item.name, status: 'fail', error: result.error })
    step('Review and fix')
    step('Verify')
    return finish('fail', 'Implement', `Implementation agent failed for ${item.name}.`, {
      items: args.items.slice(index + 1).map((remaining) => remaining.name),
      reviewAndFix: 'Skipped after implementation failure.',
      verify: 'Skipped after implementation failure.'
    })
  }
  implementations.push({ name: item.name, ...result.structured })
  if (result.structured.status === 'fail') {
    step('Review and fix')
    step('Verify')
    return finish('fail', 'Implement', `Implementation reported failure for ${item.name}.`, {
      items: args.items.slice(index + 1).map((remaining) => remaining.name),
      reviewAndFix: 'Skipped after implementation failure.',
      verify: 'Skipped after implementation failure.'
    })
  }
}

step('Review and fix')
let reviewResult = await agent(
  `Review the complete working-tree diff against this approved plan: ${JSON.stringify(args.items)}\nCheck correctness, missed callers, regressions, and required checks. Report actionable defects only in a compact structured result.`,
  { label: 'review:initial', schema: REVIEW, model: 'claude-fable-5-2', effort: 'high' }
)
if (!reviewResult.ok) {
  reviews.push({ status: 'fail', error: reviewResult.error })
  step('Verify')
  return finish('fail', 'Review and fix', 'Initial review agent failed.', {
    verify: 'Skipped after review failure.'
  })
}
reviews.push(reviewResult.structured)
let currentReview = reviewResult.structured

for (let attempt = 0; currentReview.status === 'fail' && attempt < 2; attempt += 1) {
  const model = attempt === 0 ? 'gpt-6.1-sol' : 'claude-opus-5-5'
  const fixResult = await agent(
    `Approved plan and scope: ${JSON.stringify(args.items)}\nApply the smallest complete correction for these review findings:\n${currentReview.findings.join('\n')}\nRun relevant checks and return a compact structured result.`,
    { label: `fix:${attempt + 1}`, schema: FIX, model, effort: 'high' }
  )
  fixes.push(fixResult.ok ? fixResult.structured : { status: 'fail', error: fixResult.error })

  reviewResult = await agent(
    `Review the complete working-tree diff against this approved plan: ${JSON.stringify(args.items)}\nPrior findings: ${currentReview.findings.join('\n')}\nConfirm each finding was resolved and identify remaining actionable defects in a compact structured result.`,
    { label: `review:after-fix-${attempt + 1}`, schema: REVIEW, model: 'claude-fable-5-2', effort: 'high' }
  )
  if (!reviewResult.ok) {
    reviews.push({ status: 'fail', error: reviewResult.error })
    step('Verify')
    return finish('fail', 'Review and fix', 'Follow-up review agent failed.', {
      verify: 'Skipped after review failure.'
    })
  }
  reviews.push(reviewResult.structured)
  currentReview = reviewResult.structured

  if (!fixResult.ok) {
    step('Verify')
    return finish('fail', 'Review and fix', 'Fix agent failed.', {
      verify: 'Skipped after fix failure.'
    })
  }
  if (fixResult.structured.status === 'fail') {
    step('Verify')
    return finish('fail', 'Review and fix', 'Fix agent reported failure.', {
      verify: 'Skipped after fix failure.'
    })
  }
}

if (currentReview.status === 'fail') {
  step('Verify')
  return finish('fail', 'Review and fix', 'Review still failed after two fix attempts.', {
    verify: 'Skipped after unresolved review findings.'
  })
}

step('Verify')
const checkResult = await agent(
  `Run the full project quality gate: ${args.qualityGate}\nReturn a compact structured pass or fail result with every failure.`,
  { label: 'verify:quality-gate', schema: CHECK, model: 'gpt-6.1-sol', effort: 'high' }
)
if (!checkResult.ok) {
  verification = { status: 'fail', error: checkResult.error }
  return finish('fail', 'Verify', 'Verification agent failed.')
}
verification = checkResult.structured
if (checkResult.structured.status === 'fail') {
  return finish('fail', 'Verify', 'The full quality gate failed.')
}
return finish('pass', 'Verify', 'Implementation, review, and full quality gate passed.')
```

Pass the definition, script, and serialized arguments directly to `product_workflow`. Use `product_workflow_save` only when the workflow is stable and persistence is desired.
