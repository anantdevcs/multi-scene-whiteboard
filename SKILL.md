# Multi-Scene Whiteboard Skill

This skill automates the creation of a multi-segment whiteboard explainer video using a multi-agent orchestration pattern.

## Workflow

### 1. Project Scaffolding
- Initialize a Remotion project using the template from `/home/anant/.gemini/skills/whiteboard-explainer/assets/template/`.
- Ensure `@remotion/transitions` is installed.
- Copy `/home/anant/.gemini/skills/whiteboard-explainer/references/components.tsx` to `src/components.tsx`.

### 2. Multi-Agent Delegation
For every `[Image Prompt]` and `[Voiceover]` pair provided by the user, invoke a `generalist` sub-agent to handle the segment generation in parallel.

**Sub-Agent Prompt Template:**
> Task: Create whiteboard explainer segment for Segment {N}.
> 
> Visual Prompt: {Prompt}
> Voiceover: {Voiceover}
> 
> Instructions:
> 1. Create directory `public/segment{N}`.
> 2. Generate `public/segment{N}/diagram.svg` using simple `<path>` and `<text>` elements.
> 3. Run `node /home/anant/.gemini/skills/whiteboard-explainer/scripts/generate_tts.mjs public/segment{N} "{Voiceover}"`.
> 4. Create `src/Segment{N}.tsx` using `SketchyPath` and `HandDrawnText` from `./components`. Sync animations with `timestamps.json`.
> 5. Export the component for use in a Series.

### 3. Cinematic Assembly (`src/Main.tsx`)
Once all sub-agents finish, use `TransitionSeries` from `@remotion/transitions` to stitch segments together.

- **Transition:** Use `slide({ direction: 'from-right' })` or `fade()` between scenes.
- **Timing:** Use `linearTiming({ durationInFrames: 20 })`.
- **Normalization:** Ensure all segments use `backgroundColor: "white"` for seamless transitions.

### 4. Final Render
- Calculate total duration in `src/Root.tsx` (Sum of segment durations minus transition overlaps).
- Run `npx remotion render FinalExplainer out.mp4`.

## Shared Resources
- **TTS Script:** `/home/anant/.gemini/skills/whiteboard-explainer/scripts/generate_tts.mjs`
- **Components:** `/home/anant/.gemini/skills/whiteboard-explainer/references/components.tsx`
- **Template:** `/home/anant/.gemini/skills/whiteboard-explainer/assets/template/`
