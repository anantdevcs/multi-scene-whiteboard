# Multi-Scene Whiteboard Skill

This skill automates the creation of a multi-segment whiteboard explainer video using a multi-agent orchestration pattern.

## Workflow

### 1. Project Scaffolding
Initialize a Remotion project by creating the following file structure:

- **tsconfig.json**:
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "lib": ["DOM", "DOM.Iterable", "ESNext"]
  },
  "include": ["remotion"]
}
```

- **remotion/index.ts**:
```tsx
import { registerRoot } from "remotion";
import { RemotionRoot } from "./Root";

registerRoot(RemotionRoot);
```

- **remotion/Root.tsx**:
```tsx
import { Composition } from "remotion";
import { FunnelVideo } from "./FunnelVideo";

export const RemotionRoot: React.FC = () => {
  return (
    <>
      <Composition
        id="FinalExplainer"
        component={FunnelVideo}
        durationInFrames={1800} // Update based on (audio_duration + 2s)
        fps={30}
        width={1080}
        height={1920}
      />
    </>
  );
};
```

- **src/components.tsx**:
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

Ensure `@remotion/transitions`, `@remotion/google-fonts`, `axios`, and `dotenv` are installed.

### 2. Master Audio Generation (Crucial)
1. Concatenate all scene scripts into one `master_script.txt`. **Ensure double newlines separate each scene's text**.
2. Create **generate_tts.mjs** to handle the Inworld API:
```javascript
import axios from 'axios';
import fs from 'fs';
import dotenv from 'dotenv';
import path from 'path';
import { fileURLToPath } from 'url';

dotenv.config();
const API_KEY = process.env.INWORLD_API_KEY;
const OUTPUT_DIR = process.argv[2] || './public';
const TEXT = process.argv[3];

const VOICE_ID = 'Evelyn';
const MODEL_ID = 'inworld-tts-1.5-max';

async function generateTTS() {
  if (!fs.existsSync(OUTPUT_DIR)) fs.mkdirSync(OUTPUT_DIR, { recursive: true });
  try {
    const response = await axios.post('https://api.inworld.ai/tts/v1/voice', {
      text: TEXT, voiceId: VOICE_ID, modelId: MODEL_ID, timestampType: 'WORD',
      audioConfig: { speakingRate: 1 }, temperature: 1
    }, { headers: { Authorization: `Basic ${API_KEY}`, 'Content-Type': 'application/json' } });

    const { audioContent, timestampInfo } = response.data;
    const words = [];
    const wordAlignment = timestampInfo.wordAlignment;
    const rawTextWords = TEXT.trim().split(/\s+/).filter(w => w.length > 0);
    let inworldWordIdx = 0;

    for (let i = 0; i < rawTextWords.length; i++) {
      const originalWord = rawTextWords[i];
      while (inworldWordIdx < wordAlignment.words.length) {
        const inworldWord = wordAlignment.words[inworldWordIdx].trim();
        if (inworldWord && (originalWord.toLowerCase().includes(inworldWord.toLowerCase()) || inworldWord.toLowerCase().includes(originalWord.toLowerCase().replace(/[^\w]/g, '')))) break;
        inworldWordIdx++;
      }
      if (inworldWordIdx < wordAlignment.words.length) {
        words.push({ word: originalWord, start: wordAlignment.wordStartTimeSeconds[inworldWordIdx], end: wordAlignment.wordEndTimeSeconds[inworldWordIdx] });
        inworldWordIdx++;
      }
    }

    const paragraphs = TEXT.trim().split(/\n\n+/);
    const scenes = [];
    let currentWordIdx = 0;
    paragraphs.forEach(para => {
      const paraWords = para.trim().split(/\s+/).filter(w => w.length > 0);
      const startWord = words[currentWordIdx];
      currentWordIdx += paraWords.length;
      const endWord = words[Math.min(currentWordIdx - 1, words.length - 1)];
      if (startWord && endWord) scenes.push({ start: startWord.start, end: endWord.end });
    });

    fs.writeFileSync(path.join(OUTPUT_DIR, 'voiceover.mp3'), Buffer.from(audioContent, 'base64'));
    fs.writeFileSync(path.join(OUTPUT_DIR, 'timestamps.json'), JSON.stringify({ words, scenes, duration: words.length > 0 ? words[words.length - 1].end + 0.5 : 0 }, null, 2));
  } catch (error) { console.error('Error:', error.message); process.exit(1); }
}
generateTTS();
```
3. Run the script: `node generate_tts.mjs public/master "$(cat master_script.txt)"`.
4. Use `public/master/timestamps.json` as the master clock.

### 3. Multi-Agent Delegation (Visuals Only)
For every scene, invoke a sub-agent with this prompt:
> Task: Create whiteboard explainer segment for Segment {N}.
> 
> Visual Prompt: {Prompt}
> 
> Instructions:
> 1. Create `src/Segment{N}.tsx`.
> 2. Use a **0.25x speed factor** for all animations (4x speed) to ensure elements "snap" into place.
> 3. Map **100% of SVG paths** and details to `SketchyPath`.
> 4. Synchronize triggers with the master `timestamps.json`.
> 5. Ensure all animations end at least **1.5 seconds before the segment ends**.
> 6. **Text Integrity:** Ensure all text is precisely contained within its respective box or visual boundary. Use manual line breaks (`\n`) or adjust `fontSize` dynamically to prevent any overflow.

### 4. Cinematic Assembly (`remotion/FunnelVideo.tsx`)
Stitch segments using `TransitionSeries`.

- **Sequence Duration:** Set `durationInFrames` for Sequence N to `Math.round((Scene N+1 start - Scene N start) * fps)`.
- **Final Scene Dwell:** For the **last segment**, add a **2-second (60 frames)** "dwell time" after the audio ends.
- **Transition:** Use `slide({ direction: 'from-right' })` with `linearTiming({ durationInFrames: 15 })`.

### 5. Final Render
- **Synchronization Check:** Set total duration in `Root.tsx` to `Math.round((audio_duration + 2s) * fps)`.
- Render: `npx remotion render FinalExplainer out.mp4 --concurrency=4 --timeout=120000`.
