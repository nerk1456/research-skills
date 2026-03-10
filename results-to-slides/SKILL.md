---
name: results-to-slides
description: Automate the grunt work of making research presentations — discovers experiments from git/output folders, collects images and metrics, and organizes them into slides. The research narrative and story are yours to tell; this skill just saves you from manually copying images and tabulating results. Creates a slide-by-slide script for user approval, then generates slide markdown and editable PPTX.
argument-hint: [start_date end_date]
disable-model-invocation: true
---

# Results to Slides

Automate the tedious parts of research presentations: discover experiments, collect images/metrics, organize into slides. You provide the story — this skill handles the image-copying, metric-tabulating grunt work. Output: slide markdown + editable PPTX.

## Parallel Execution

Maximize use of the Agent tool. Whenever you have 2+ independent tasks, launch parallel agents.

- **Phase 1**: Launch agents in parallel to read CLAUDE.md, README.md, memory files, and other docs
- **Phase 2 (biggest win)**: Launch separate agents for git log analysis, output folder discovery,
  script/code discovery, and media discovery — these are fully independent
- **Phase 3**: Launch agents in parallel to read different experiment scripts and output folders
- **Phase 5+6**: Sequential (markdown must be written before converting)

Do not serialize what can be parallelized. The researcher's time is valuable.

---

## Important Paths

- Skill directory: `${CLAUDE_SKILL_DIR}`
- Converter: `${CLAUDE_SKILL_DIR}/md_to_pptx.py`
- Backgrounds: `${CLAUDE_SKILL_DIR}/backgrounds/`
- Theme CSS: `${CLAUDE_SKILL_DIR}/theme.css`
- Slide element reference: [slide_reference.md](slide_reference.md)

---

## Phase 0: Parse Arguments & Setup

The user provides: `/results-to-slides START_DATE END_DATE`

Arguments come as `$ARGUMENTS` containing two MMDD date strings (e.g., `0301 0308`).

**Parse the dates:**
- `$0` = start date (MMDD), e.g., `0301` → March 1st
- `$1` = end date (MMDD), e.g., `0308` → March 8th
- Infer the year from the current system date
- The end date is INCLUSIVE — include experiments from that day
- For `find` commands, use end_date + 1 day as the upper bound

**If no arguments provided**, use the `AskUserQuestion` tool:
- header: "Date range"
- question: "What date range should this presentation cover? Use MMDD MMDD format (e.g., 0301 0308)."
- options: ["Last week", "Last 2 weeks", "Last month"]
- The user can also type a custom range via "Other"

**Output directory**: `presentation/YYYY_MM_DD/` using the end date.

```bash
mkdir -p presentation/YYYY_MM_DD
```

### Step 0.1: Ask Presentation Preferences

Ask all upfront preferences in a single sequence before doing any discovery work.

**Slide count** — Use the `AskUserQuestion` tool:
- header: "Presentation length"
- question: "How many content slides? (excluding the title slide)"
- options:
  - label: "5 slides", description: "Quick summary — key highlights only"
  - label: "10 slides", description: "Standard update — covers main experiments"
  - label: "20 slides", description: "Detailed walkthrough — full experiment coverage"

Store the chosen slide count. This determines how much detail to include:
- **5 slides**: Only the most important findings, heavily visual, skip intermediate steps
- **10 slides**: Cover each major experiment phase, balance text and visuals
- **20 slides**: Full coverage including failed experiments, intermediate results, and methodology details

**Background theme** — Use the `AskUserQuestion` tool:
- header: "Background theme"
- question: "Which background theme?"
- options:
  - label: "Light (Recommended)", description: "Clean near-white with very subtle teal/coral accents"
  - label: "Warm", description: "Warm paper gradient with teal/coral accents (classic)"
  - label: "Dark", description: "Near-black dark grey (#1C1C1E) with muted accent bar"

**Background-to-file and theme mapping**:

| Choice | Background file | `--theme` flag |
|--------|----------------|----------------|
| Light  | `slide_bg_light.png` | `light` |
| Warm   | `slide_bg_warm.png`  | `light` |
| Dark   | `slide_bg_dark.png`  | `dark`  |

Store the chosen background filename and theme flag for use in Phase 6 (copy) and Phase 7 (convert).

---

## Phase 1: Research Context Discovery

### Step 1.1: Find Research Goal

Search for the research goal by scanning available project documentation:

1. **CLAUDE.md** in project root — look for sections named "Project Goal", "Scientific Goal", "Research Goal", "Objective", or similar
2. **README.md** — look for project description or goals
3. **Auto-memory files** — check `.claude/projects/*/memory/MEMORY.md` for research context
4. **Any other docs** — glob for `*.md` in root and `docs/` directory, skim for goal/objective sections

If no clear research goal is found, use the `AskUserQuestion` tool:
- header: "Research context"
- question: "Brief description of what this project does? (helps me label experiments correctly, not for building a story)"
- options: ["Let me describe it", "Use README description"]
- The user will typically pick "Other" and type their context in free text

### Step 1.2: Gather Background Context

Read available project documentation to build a deep understanding of the research:
- What is the underlying research question or direction
- What methods/approaches have been tried (historically, not just this date range)
- What the key metrics are and what "good" vs "bad" looks like
- Key terminology, model names, dataset names
- What output folder naming conventions are used
- Known failure modes, baselines, and prior results

Sources: CLAUDE.md, README.md, any other markdown docs in the project root or `docs/` directory, and memory files.

**Why this matters**: Understanding the research direction is always essential — even with a handful of experiments. Without it, folder names and script names are opaque labels. With it, you can group related experiments, skip debug noise, and pick the right images.

**How to use this context**: Use it internally for smart organization — grouping, filtering, prioritization. Do NOT let it leak into slide text as editorial interpretation. The slides themselves stay factual.

---

## Phase 2: Experiment Discovery

Find all experiments conducted within the date range using multiple signals. Use ALL of these methods — each catches things the others miss.

### Step 2.1: Git Log Analysis

```bash
git log --after="YYYY-MM-DD_START" --before="YYYY-MM-DD_END+1" --oneline --stat
```

Extract from each commit:
- **Commit message** — often describes what was tried and results (e.g., "v2 score-guided velocity steering: VLM=0.769")
- **Files changed** — identifies which scripts were created/modified
- **Date** — chronological ordering

### Step 2.2: Output Folder Discovery

Find output/results folders modified in the date range using file system modification times:

```bash
find . -maxdepth 2 -type d -newermt "YYYY-MM-DD_START" ! -newermt "YYYY-MM-DD_END+1" 2>/dev/null | sort
```

Also check common output directory names: `outputs/`, `results/`, `experiments/`, `runs/`, `logs/`, `checkpoints/`.

**IMPORTANT**: Do NOT assume MMDD_ folder naming. Many projects use different conventions. File modification time is the most reliable signal.

For folders WITH date prefixes (like `MMDD_`), you can also filter by prefix.

For each output folder found:
- Note the folder name — it often contains the experiment description
- List contents to understand what was saved
- Look for result files: `metrics.json`, `*.pkl`, `scores.txt`, `summary.txt`, `*.log`
- Look for images: `*.png`, `*.jpg`, `*.gif`

### Step 2.3: Script/Code Discovery

Find experiment scripts modified in the date range:

```bash
find . -name "*.py" -newermt "YYYY-MM-DD_START" ! -newermt "YYYY-MM-DD_END+1" 2>/dev/null | grep -v __pycache__ | sort
```

Also check for shell scripts (`.sh`) and any run/launch scripts that call Python scripts — these often contain the actual experiment configuration (arguments, hyperparameters, ablation flags).

For each relevant script, **read the code** to understand:
- **What the experiment actually does** — the core logic, not just the filename. What model is being trained/evaluated? What loss function? What data?
- **Output directory paths** — where results were saved
- **Key parameters and configurations** — argparse defaults, config files loaded, hardcoded values
- **What is being compared** — does it run multiple configs? Does it load a baseline for comparison?
- **How results are computed and saved** — metric computation, print statements, logging calls
- **Comments describing purpose** — researchers often leave notes about what they're testing

This code reading is critical for accurate slide content. The folder name `0305_exp3_v2` tells you nothing; the script that generated it tells you everything.

### Step 2.4: Media Discovery Strategy

Output folders typically contain three kinds of visual content:
- **Charts/plots** (.png): generated by matplotlib/plotting code — scatter plots, confusion matrices, heatmaps
- **Photos** (.jpg/.jpeg): sample images, dataset examples, real-world captures
- **Videos** (.mp4): recordings, training progressions, demonstrations

When scanning output folders for media to include in slides:

1. **Read the experiment script first** to find how images are saved. Look for:
   - `plt.savefig(...)`, `Image.save(...)`, `cv2.imwrite(...)`
   - Variable names like `best_path`, `baseline_path`, `output_path`
   - Filename patterns in f-strings

2. **Common naming patterns** (check if they exist):
   - `baseline.*`, `base.*`, `original.*` → before/reference images
   - `best_*.*`, `top_*.*` → best results
   - `*_score*.*`, `*_vlm*.*`, `*_loss*.*` → images with metrics in name
   - `eval*.*`, `step*.*`, `iter*.*`, `epoch*.*` → progression images
   - `comparison.*`, `grid.*`, `montage.*` → summary images

3. **If no clear naming**: List all image files, sort by name, and pick:
   - First image (often baseline/initial)
   - Last image (often final/best result)
   - Any image with the highest number in filename

4. **Image grid sizing**:
   - 1 image → full width (`cols-2` with single `figure-card`)
   - 2 images → side-by-side (`cols-2`) — ideal for before/after
   - 3 images → three columns (`cols-3`) — progression or comparison
   - 4 images → 2x2 grid (`cols-4`) — multiple comparisons

### Step 2.5: Cross-Reference & Build Timeline

Build a unified timeline by cross-referencing:
- Git commits ↔ output folders (commit messages often reference folder names)
- Scripts ↔ output folders (scripts define output paths)
- Chronological ordering by modification time

Create a structured experiment list:
```
DATE | EXPERIMENT_NAME | SCRIPT | OUTPUT_FOLDER | KEY_RESULT | IMAGES
```

---

## Phase 3: Experiment Organization

**Philosophy**: Use your understanding of the research context to intelligently organize experiments — group related ones, skip noise, pick the right images. But the slides themselves present only what was done and what the result was. The researcher adds the story during their talk.

### Step 3.1: Understand, Then Organize

Use the research context from Phase 1 to:
- **Group** experiments that are variations of the same idea (e.g., 5 runs with different hyperparameters → one summary slide with a comparison table)
- **Filter** trivial runs: debug scripts, parameter typos, one-off tests that produced nothing
- **Order** experiments in a way that makes sense given the research direction (usually chronological, but group related experiments even if they span days)
- **Prioritize** when there are more experiments than slide slots: keep the ones that produced meaningful results or interesting failures, drop intermediate reruns

Research understanding is always essential for smart organization — it tells you which experiments are variations of the same idea, which are unrelated, and what the project-specific terminology means. This understanding stays internal; it informs organization, not slide text.

### Step 3.2: Map Experiments to Slides

For each experiment (or group of closely related experiments), create one slide containing:

1. **What was done** — the method, approach, or configuration tested (factual description from script/commit)
2. **What was the result** — key metrics, scores, numbers (from metrics files, logs, or commit messages)
3. **Visual evidence** — images from the output folder (before/after, comparisons, grids)

**On slide text, do NOT**:
- Editorialize about significance ("breakthrough", "key insight", "this suggests")
- Speculate about the researcher's reasoning or hypotheses
- Add transition language between experiment slides ("Building on the previous result...")
- Reinterpret or rephrase results beyond what the data shows

### Step 3.3: Design Slide Structure

Structure for the presentation:

1. **Title slide** — Project name, date range, best metric achieved (factual)
2. **Context slide** (optional, only if research goal was found/provided) — 2-3 bullets on what this project does
3. **Experiment slides** — one per experiment or group, chronological order
4. **Summary table** (optional) — if there are comparable metrics across experiments, put them in a table

**Do NOT add**: timeline slides, next steps slides, takeaway slides, or any slide that requires editorial interpretation. Stick to what was actually done and measured.

### Step 3.4: Select Representative Images

For each experiment, find images that show the result:
- **Baseline/before** image (if available)
- **Best result** image
- **Comparison** images — side-by-side if the experiment has before/after
- **Failure** images — if the experiment failed, show what happened

Limit to 2-4 images per slide. If you're unsure which images are most relevant, include the baseline and the best result — the user can swap during review.

---

## Phase 4: Script Generation & User Feedback

### Step 4.1: Generate Slide Script

Create a structured outline for each proposed slide. Headings must be factual descriptions of the experiment and its result — not editorial claims.

Format:

```
SLIDE SCRIPT
============

Slide 1: Title
  Type: Title (lead)
  Heading: [Project Name — Weekly Update]
  Subtitle: [Date range, N experiments]
  Chips: [N slides], [date range], [key topics]

Slide 2: [Experiment Name]
  Type: Content
  Heading: [What was done: key metric]  (e.g., "Learning Rate Sweep (1e-3 to 1e-5): Best at 3e-4")
  Bullets:
    - [Configuration/method detail]
    - [Result metric]
  Images:
    - [path/to/image.png]: [caption]
  Grid: cols-[2/3/4]

...
```

### Step 4.2: Ask User Feedback Preference

Use the `AskUserQuestion` tool:
- header: "Review mode"
- question: "How would you like to review the presentation script?"
- options:
  - label: "Show each slide", description: "Review and approve one slide at a time"
  - label: "Show all slides", description: "See the full script at once, then give feedback"
  - label: "Skip review", description: "Generate directly without preview"

### Step 4.3a: Per-Slide Review ("Show each slide")

For each slide:
1. Present the slide script as text output
2. Use `AskUserQuestion` tool:
   - header: "Slide N"
   - question: "Approve this slide, or describe changes?"
   - options:
     - label: "Approve", description: "Keep this slide as-is"
     - label: "Edit", description: "I'll describe changes needed"
     - label: "Remove", description: "Drop this slide"
3. If "Edit" selected, the user's notes will contain their requested changes — incorporate them before moving to the next slide

### Step 4.3b: Full Script Review ("Show all slides")

1. Present the complete script (all slides) as text output
2. Use `AskUserQuestion` tool:
   - header: "Feedback"
   - question: "Which slides need changes? (e.g., 'Slide 3: add more detail about X', 'Remove slide 5', 'Combine slides 4 and 5')"
   - options:
     - label: "Looks good", description: "No changes needed, proceed to generation"
     - label: "Changes needed", description: "I'll describe what to change"
3. If "Changes needed", incorporate the user's feedback in one pass

### Step 4.3c: Skip Review ("Skip review")

Proceed directly to slide markdown generation.

---

## Phase 5: Generate Slide Markdown

### Step 6.1: Set Up Output Directory

```bash
mkdir -p presentation/YYYY_MM_DD
cp ${CLAUDE_SKILL_DIR}/theme.css presentation/YYYY_MM_DD/
cp ${CLAUDE_SKILL_DIR}/backgrounds/CHOSEN_BG.png presentation/YYYY_MM_DD/slide_bg.png
```

### Step 6.2: Write Slide Markdown

Write the markdown file to `presentation/YYYY_MM_DD/slides.md`.

**File must start with this header:**

```markdown
---
theme: notebook-status
paginate: true
size: 16:9
---
```

**Slide separator**: `---` on its own line between slides.

**Use ONLY the element vocabulary from [slide_reference.md](slide_reference.md)**. The PPTX converter only understands these specific HTML elements.

### Quick Element Reference

**Title slide:**
```markdown
<!-- _class: lead -->

# Main Title

Summary sentence here.

<div class="chips">
  <span class="chip">tag1</span>
  <span class="chip">tag2</span>
</div>
```

**Content slide:**
```markdown
## Slide Heading

- Bullet with **bold** and `code`.
- Another bullet point.
```

**Images:**
```html
<div class="cols-3">
  <figure class="figure-card"><img src="outputs/folder/image.png"><figcaption>Caption</figcaption></figure>
  <figure class="figure-card"><img src="outputs/folder/image2.png"><figcaption>Caption 2</figcaption></figure>
  <figure class="figure-card"><img src="outputs/folder/image3.png"><figcaption>Caption 3</figcaption></figure>
</div>
```

**Table:**
```html
<div class="table-box">
<table>
<tr><th>Method</th><th>Score</th><th>Notes</th></tr>
<tr><td>Approach A</td><td><strong>0.94</strong></td><td>Best</td></tr>
</table>
</div>
```

**Chips (TITLE SLIDE ONLY):**
```html
<div class="chips">
  <span class="chip">Tag 1</span>
  <span class="chip">Tag 2</span>
</div>
```

**IMPORTANT: Chips are ONLY allowed on the title (lead) slide. Do NOT use chips on any content slide.**

### Banned Elements

Do NOT use ANY of the following elements on content slides:
- **Chips** (`<div class="chips">`) — only allowed on title slide
- **Split layout** (`<div class="split">`) — cluttered, use simple bullets instead
- **Arch-box** (`<div class="arch-box">`) — use bullets or tables instead
- **Stat cards** (`<div class="stats">`) — put key numbers in bullets
- **Eyebrow** (`<span class="eyebrow">`) — unnecessary label
- **Decision card** (`<div class="card">`) — no editorial content
- **Timeline** (`<div class="timeline">`) — no timeline slides

### Image Path Rules

- All image paths in the markdown must be **relative to the project root directory**
- The converter resolves them via `--base-dir`
- Example: `outputs/0308_experiment/best_result.png`
- Missing images render as grey placeholder rectangles (non-fatal)

### Content Density Limits

Keep slides readable — if content overflows, split into multiple slides:

| Slide Type | Maximum |
|------------|---------|
| Title | 1 heading + 1 subtitle + chips |
| Content | 1 heading + 4 bullets + optional images |
| Comparison | 1 heading + 2-4 images |
| Table | 1 heading + 1 context line + table (max 5 rows, NO bullets) |

**Do NOT generate**: timeline slides, next steps slides, decision cards, chips on content slides, split/arch-box layouts, or any speculative/editorial content. Only present what was actually done and measured. Never editorialize about significance or connections between experiments.

---

## Phase 6: Convert to PPTX

Run the converter using the Python environment available on the system:

```bash
python ${CLAUDE_SKILL_DIR}/md_to_pptx.py \
  --input presentation/YYYY_MM_DD/slides.md \
  --output presentation/YYYY_MM_DD/slides.pptx \
  --bg-image presentation/YYYY_MM_DD/slide_bg.png \
  --theme light \
  --base-dir .
```

Use `--theme dark` when the dark background was selected. Use `--theme light` for both the light and off-white backgrounds.

**Note**: If the default `python` doesn't have `python-pptx` installed, check for:
- A conda/mamba environment with python-pptx
- The path specified in CLAUDE.md or memory files
- Fall back to asking the user which Python to use

### Report Output

After successful conversion, report:

```
Presentation generated!

  Markdown: presentation/YYYY_MM_DD/slides.md
  PPTX:     presentation/YYYY_MM_DD/slides.pptx
  Theme:    presentation/YYYY_MM_DD/theme.css
  Slides:   [count]

The PPTX has editable text, tables, and shapes on top of the theme background.
Open in PowerPoint or LibreOffice Impress to edit further.
```

---

## Slide Content Style Rules

These rules are distilled from a well-received research presentation. Follow them strictly.

### Element Ordering (Strict)

Every slide must follow this top-to-bottom order. Skip elements that aren't needed, but NEVER reorder them:

1. **Heading** (h2) — always first
2. **Bullets** (max 4) — after heading. **Omit entirely on table slides** — use a single context line instead.
3. **Table** — after context line (no bullets on the same slide)
4. **Images** — **ALWAYS LAST on the slide, never before text/bullets**

### Banned Elements (STRICT — never use these on content slides)

- **Chips** (`<div class="chips">`) — ONLY allowed on title (lead) slide, NEVER on content slides
- **Split layout** (`<div class="split">`) — NEVER use, replace with simple bullets
- **Arch-box** (`<div class="arch-box">`) — NEVER use, replace with bullets or tables
- **Stat cards** (`<div class="stats">`) — NEVER use, put numbers in bullets
- **Eyebrow** (`<span class="eyebrow">`) — NEVER use
- **Decision card** (`<div class="card">`) — NEVER use
- **Timeline** (`<div class="timeline">`) — NEVER use

### Writing Style

- **Headings describe what was done + the result.** Write "ResNet-50 on ImageNet: 76.1% Top-1" not "ResNet-50 Is the Clear Winner". Write "Learning Rate Sweep (1e-3 to 1e-5): Best at 3e-4" not "Learning Rate Was the Missing Piece". The heading is a factual label, not an editorial claim — the user adds interpretation during their talk.
- **Bullets state facts, not interpretations.** Write "Adam, lr=3e-4, batch=64 → 76.1% top-1" not "This confirms Adam is the best optimizer". Report what was configured and what the metric was. Do NOT claim significance, do NOT say why something matters.
- **Bold for key numbers**: `**76.1%**`, `**+2.3**`, `**64-d**`
- **Bold for key terms/method names** used in the project
- **Code formatting for technical identifiers**: `` `learning_rate` ``, `` `batch_size=64` ``
- **One visual concept per slide** — never mix grid types or have multiple unrelated sections
- **Never use these words in headings or bullets**: "breakthrough", "key insight", "importantly", "this suggests", "this shows that", "the critical finding". Just state the experiment and the number.

### Content Density

- **Bullets**: max **4** per slide, each a single line — if you need more, split into two slides
- **Table slides**: NO bullets. Use only a **single context line** (plain text, not a bullet) between the heading and the table to explain what the table shows. The table speaks for itself.
- **Tables**: max 5 rows
- **Images**: 1-4 per slide, using `cols-2` (2 images), `cols-3` (3 images), or `cols-4` (4 images as 2x2)
- **One heading per slide** — never two headings
- **Maximize visuals**: Every experiment slide should include images if ANY exist in the output folder. Prefer before/after comparisons, result grids, and comparison images. A slide with only text is a missed opportunity — always look harder for visual evidence before giving up.

### Image Captions

- **Terse identifiers**, not sentences: `baseline`, `eval70`, `scale 0.25`, `donor14`
- Use `<code>` tags for file paths or parameter values
- Use `<strong>` for scores: `Per-step: <strong>0.0495</strong>`
- Short enough to fit one line under the image

### Title Slide Pattern

```markdown
<!-- _class: lead -->

# Project Title — Weekly Update

Date range and number of experiments covered.

<div class="chips">
  <span class="chip">N slides</span>
  <span class="chip">date range</span>
  <span class="chip">key method</span>
</div>

<div class="cols-2">
  <figure class="figure-card"><img src="hero_image.png"><figcaption>caption</figcaption></figure>
</div>
```

The title slide should use the project name, not an editorial headline. The user can change the title during review.

### Common Slide Patterns

**Explanation + evidence slide:**
```
heading → bullets(2-4) → images(cols-2 or cols-3)
```

**Table slide (NO bullets):**
```
heading → one context line → table(results) → images(optional visual comparison)
```

**Visual comparison slide:**
```
heading → bullets(1-2 for context) → images(cols-2, cols-3, or cols-4)
```


---

## Error Handling

- **No experiments found in date range**: Tell the user, suggest expanding the range or checking if experiments are tracked in git
- **No images in output folders**: Generate text-only slides, note which slides could benefit from images
- **Missing python-pptx**: Report the error, suggest installation, still deliver the slide markdown file
- **Background image missing**: Warn but continue — slides will have white background
- **Git not available**: Fall back to file modification times only

---

## Tips for Generating Slides

1. **State what was done and what happened** — method, config, metric. Don't editorialize.
2. **Show before/after images** wherever possible — visual evidence saves the user from having to find and copy them manually
3. **Group failed experiments** into one slide with a comparison table rather than individual slides
4. **Keep bullets short** — one line each, max **4** per slide
5. **Maximize visuals** — every experiment slide should have images if any exist. Look in subdirectories (`eval/`, `comparisons/`, `baselines/`) not just the folder root.
6. **Tables replace bullets** — when a slide has a table, do NOT add bullets. Use one plain context line above the table instead.
7. **When uncertain about an experiment's purpose**, present the observable facts (script name, parameters, output metrics, images) and let the user interpret. Do NOT guess the intent.
