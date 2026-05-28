# Multi-Scene Whiteboard Skill

This skill automates the creation of a multi-segment whiteboard explainer video using a multi-agent orchestration pattern.

## Workflow

### 1. Project Scaffolding
- Initialize a Remotion project using the template from /home/anant/.gemini/skills/whiteboard-explainer/assets/template/.
- Ensure @remotion/transitions and @remotion/google-fonts are installed.
- **Create src/components.tsx** with the following content:

```tsx
import React from "react";
import { interpolate, useCurrentFrame, Easing } from "remotion";
import { loadFont } from "@remotion/google-fonts/ArchitectsDaughter";

const { fontFamily } = loadFont();

export const SketchyPath: React.FC<{
  d: string;
  startFrame: number;
  duration: number;
  stroke?: string;
  strokeWidth?: number;
  fill?: string;
  opacity?: number;
}> = ({
  d,
  startFrame,
  duration,
  stroke = "#000",
  strokeWidth = 2,
  fill = "none",
  opacity = 1,
}) => {
  const frame = useCurrentFrame();
  const progress = interpolate(frame, [startFrame, startFrame + duration], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
    easing: Easing.inOut(Easing.quad),
  });

  const pathRef = React.useRef<SVGPathElement>(null);
  const [length, setLength] = React.useState(0);

  React.useEffect(() => {
    if (pathRef.current) setLength(pathRef.current.getTotalLength());
  }, []);

  return (
    <path
      ref={pathRef}
      d={d}
      fill={fill}
      stroke={stroke}
      strokeWidth={strokeWidth}
      strokeDasharray={length || 1000}
      strokeDashoffset={length * (1 - progress)}
      strokeLinecap="round"
      strokeLinejoin="round"
      style={{ opacity }}
    />
  );
};

export const HandDrawnText: React.FC<{
  text: string;
  x: number;
  y: number;
  startFrame: number;
  fontSize?: number;
  color?: string;
  align?: "start" | "middle" | "end";
}> = ({ text, x, y, startFrame, fontSize = 24, color = "#000", align = "middle" }) => {
  const frame = useCurrentFrame();
  const progress = interpolate(frame, [startFrame, startFrame + text.length * 2], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });

  return (
    <text
      x={x}
      y={y}
      fontFamily={fontFamily}
      fontSize={fontSize}
      fill={color}
      textAnchor={align}
      style={{ filter: "url(#rough-paper)" }}
    >
      {text.slice(0, Math.floor(text.length * progress))}
    </text>
  );
};
```

### 2. Master Audio Generation (Crucial)
- **Do not generate audio scene-by-scene.**
- Concatenate all scene scripts into one master_script.txt. **Ensure double newlines separate each scene's text**, as the TTS script detects scenes based on paragraphs.
- Run the TTS script on the entire text: `node /home/anant/.gemini/skills/whiteboard-explainer/scripts/generate_tts.mjs public/master "PASTE_TEXT_HERE"`.
- Use public/master/timestamps.json as the master clock for the entire project.

### 3. Multi-Agent Delegation (Visuals Only)
For every [Image Prompt] and [Voiceover] pair, invoke a generalist sub-agent.

**Sub-Agent Prompt Template:**
> Task: Create whiteboard explainer segment for Segment {N}.
> 
> Visual Prompt: {Prompt}
> 
> Instructions:
> 1. Create src/Segment{N}.tsx.
> 2. Use a **0.25x speed factor** for all animations (4x speed) to ensure elements "snap" into place.
> 3. Map **100% of SVG paths** and details (faces, decorations) to SketchyPath.
> 4. Synchronize triggers with the master timestamps.json.
> 5. Ensure all animations end at least **1.5 seconds before the segment ends**.
> 6. **Text Integrity:** Ensure all text is precisely contained within its respective box or visual boundary. Use manual line breaks (\n) or adjust fontSize dynamically to prevent any overflow or "text bleed" outside of containers.

### 4. Cinematic Assembly (src/Main.tsx)
Stitch segments using TransitionSeries.

- **Sequence Duration:** To ensure perfect audio synchronization, the durationInFrames for Sequence N must exactly match the length of the audio scene. Calculate this by setting it to (audio_start_frame of Scene N+1) - (audio_start_frame of Scene N). Do NOT use an arbitrary visual buffer, as this causes desynchronization. The visual transition to the next scene must begin exactly when the audio for that next scene starts.
- **Final Scene Dwell:** For the **last segment**, add a **2-second (60 frames at 30fps)** "dwell time" after the audio ends to allow the final visual to settle before the video finishes.
- **Transition:** Use slide({ direction: 'from-right' }).
- **Timing:** Use linearTiming({ durationInFrames: 15 }).

### 5. Final Render
- **Synchronization Check:** Calculate total duration in src/Root.tsx to precisely match (total_audio_duration + final_dwell_time). This prevents audio cutoff or "moov atom" errors caused by mismatched stream lengths.
- Render using safe software encoding settings to avoid MP4 container corruption: npx remotion render FinalExplainer out.mp4 --concurrency=4 --timeout=120000. Do NOT use experimental GPU flags like --gl=egl unless hardware stability is fully confirmed on the host machine.

## Shared Resources
- **TTS Script:** /home/anant/.gemini/skills/whiteboard-explainer/scripts/generate_tts.mjs
- **Template:** /home/anant/.gemini/skills/whiteboard-explainer/assets/template/
