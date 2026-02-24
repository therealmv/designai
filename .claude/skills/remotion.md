# Remotion Skill

Use this skill when the user wants to build or modify videos programmatically using Remotion (React-based video framework). Trigger if the request:
- Mentions Remotion, `@remotion/*`, `remotion`, or creating videos with React
- Involves rendering video frames, compositions, sequences, or animations
- References Remotion-specific APIs: `useCurrentFrame`, `useVideoConfig`, `interpolate`, `spring`, `Composition`, `Sequence`, etc.

---

## What Is Remotion

Remotion lets you build videos using React components. A video is a React component that receives the current frame number as context and renders accordingly. You run a dev server to preview, then render to MP4/GIF/WebM via CLI or Node.js API.

**Current stable version:** 4.0.x
**Peer deps:** React ≥ 16.8, React DOM ≥ 16.8
**Package manager:** Bun (in the monorepo itself); user projects use npm/yarn/pnpm/bun

---

## Getting Started

```bash
npx create-video@latest
```

This scaffolds a new project. Choose a template (blank, hello-world, three.js, TikTok, etc.).

**Dev server:**
```bash
npx remotion studio
# or
npx remotion preview   # alias
```

**Render:**
```bash
npx remotion render <entry-file> <composition-id> <output-file>
npx remotion still  <entry-file> <composition-id> <output-file>
```

---

## Core Package: `remotion`

Install: `npm install remotion react react-dom`

### Entry Point

Every Remotion project needs a root file that calls `registerRoot`:

```tsx
// src/index.ts (or Root.tsx)
import { registerRoot } from 'remotion';
import { RemotionRoot } from './Root';

registerRoot(RemotionRoot);
```

```tsx
// src/Root.tsx
import { Composition } from 'remotion';
import { MyVideo } from './MyVideo';

export const RemotionRoot: React.FC = () => {
  return (
    <Composition
      id="MyVideo"
      component={MyVideo}
      durationInFrames={150}
      fps={30}
      width={1920}
      height={1080}
      defaultProps={{ text: 'Hello' }}
    />
  );
};
```

---

## Core Components

### `<Composition>`
Registers a video/still for rendering. Appears in the Studio sidebar.

```tsx
<Composition
  id="MyVideo"              // required, URL-safe string
  component={MyVideo}       // React component to render
  durationInFrames={150}    // required (omit for stills)
  fps={30}                  // required
  width={1920}              // required (or use calculateMetadata)
  height={1080}             // required
  defaultProps={{ ... }}    // typed via schema or generics
  schema={myZodSchema}      // optional Zod schema for props
  calculateMetadata={async ({ props, defaultProps, abortSignal }) => ({
    durationInFrames: 200,
    width: 1280,
    height: 720,
    props: { ...props },
  })}
/>
```

> Cannot be nested inside other Compositions or `<Player>`.

### `<AbsoluteFill>`
Shorthand for `position: absolute; top: 0; left: 0; width: 100%; height: 100%`.

```tsx
import { AbsoluteFill } from 'remotion';

export const MyScene: React.FC = () => (
  <AbsoluteFill style={{ backgroundColor: 'blue' }}>
    <h1>Hello</h1>
  </AbsoluteFill>
);
```

### `<Sequence>`
Time-shifts children — they only render when the current frame is within the sequence's window.

```tsx
import { Sequence } from 'remotion';

// Renders from frame 30, for 60 frames
<Sequence from={30} durationInFrames={60}>
  <MyScene />
</Sequence>

// Layout options
<Sequence from={0} layout="none">   // no AbsoluteFill wrapper
  <span>inline content</span>
</Sequence>
```

Props:
- `from` (number, default 0): first frame to render
- `durationInFrames` (number, default Infinity): how long to show
- `layout`: `"absolute-fill"` (default) | `"none"`
- `name`: display name in timeline
- `premountFor` / `postmountFor`: frames to mount before/after window

### `<Series>`
Convenience wrapper that stacks Sequences back-to-back.

```tsx
import { Series } from 'remotion';

<Series>
  <Series.Sequence durationInFrames={40}><SceneA /></Series.Sequence>
  <Series.Sequence durationInFrames={60}><SceneB /></Series.Sequence>
</Series>
```

### `<Loop>`
Repeats children for a given number of times or duration.

```tsx
import { Loop } from 'remotion';
<Loop durationInFrames={30} times={3}>
  <Blink />
</Loop>
```

### `<Still>`
Registers a still image composition (no `durationInFrames` / `fps`).

```tsx
<Still id="Thumbnail" component={Thumbnail} width={1200} height={630} />
```

### `<Video>` / `<OffthreadVideo>`
Renders a video asset. `OffthreadVideo` renders via FFmpeg (recommended for rendering; `Video` is for the Player).

```tsx
import { Video, OffthreadVideo } from 'remotion';

<Video src={staticFile('video.mp4')} startFrom={0} endAt={90} />
<OffthreadVideo src="https://example.com/clip.mp4" />
```

Props: `src`, `startFrom`, `endAt`, `volume`, `muted`, `playbackRate`, `style`

### `<Audio>`
Embeds audio in a composition.

```tsx
import { Audio } from 'remotion';
<Audio src={staticFile('music.mp3')} volume={0.8} startFrom={0} endAt={150} />
```

### `<Img>`
Drop-in for `<img>` that integrates with Remotion's rendering pipeline (waits for load).

```tsx
import { Img } from 'remotion';
<Img src={staticFile('logo.png')} style={{ width: 200 }} />
```

### `<IFrame>`
Embeds an iframe and waits for load during rendering.

---

## Core Hooks

### `useCurrentFrame()`
Returns the current frame number (0-indexed integer).

```tsx
import { useCurrentFrame } from 'remotion';

const MyComp: React.FC = () => {
  const frame = useCurrentFrame(); // e.g. 0, 1, 2 ... durationInFrames-1
  return <div>Frame: {frame}</div>;
};
```

### `useVideoConfig()`
Returns the composition's metadata.

```tsx
import { useVideoConfig } from 'remotion';

const MyComp: React.FC = () => {
  const { width, height, fps, durationInFrames, id, defaultProps, props } = useVideoConfig();
  // width/height: pixels
  // fps: frames per second
  // durationInFrames: total frame count
  return <div>{width}×{height} @ {fps}fps</div>;
};
```

### `useDelayRender()` / `delayRender()` / `continueRender()`
Pauses rendering until async work completes (e.g. data fetching).

```tsx
import { useDelayRender, continueRender, cancelRender } from 'remotion';

const MyComp: React.FC = () => {
  const [data, setData] = useState(null);
  const handle = useDelayRender();   // or: const handle = delayRender('Loading data')

  useEffect(() => {
    fetch('/api/data')
      .then(r => r.json())
      .then(d => { setData(d); continueRender(handle); })
      .catch(e => cancelRender(e));
  }, []);

  if (!data) return null;
  return <div>{data.title}</div>;
};
```

---

## Animation Utilities

### `interpolate(input, inputRange, outputRange, options?)`
Maps a number from one range to another (similar to React Native's `Animated.interpolate`).

```tsx
import { interpolate } from 'remotion';

const opacity = interpolate(frame, [0, 30], [0, 1]);
// frame=0 → 0, frame=30 → 1

// With clamping (prevent values outside 0–1)
const opacity = interpolate(frame, [0, 30], [0, 1], {
  extrapolateLeft: 'clamp',
  extrapolateRight: 'clamp',
});

// Custom easing
import { Easing } from 'remotion';
const y = interpolate(frame, [0, 60], [0, 300], {
  easing: Easing.bezier(0.8, 0, 0.2, 1),
  extrapolateRight: 'clamp',
});
```

**Extrapolation modes:** `'extend'` (default), `'clamp'`, `'identity'`, `'wrap'`

### `spring({ frame, fps, config?, from?, to?, durationInFrames?, delay?, reverse? })`
Physics-based spring animation.

```tsx
import { spring, useCurrentFrame, useVideoConfig } from 'remotion';

const MyComp: React.FC = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  const scale = spring({
    frame,
    fps,
    config: { damping: 10, stiffness: 100, mass: 1, overshootClamping: false },
    from: 0,
    to: 1,
  });

  return <div style={{ transform: `scale(${scale})` }}>Hello</div>;
};
```

`config` presets via `SpringConfig`:
- `SpringConfig.gentle` — soft, slow
- `SpringConfig.wobbly` — bouncy
- `SpringConfig.stiff` — quick
- `SpringConfig.molasses` — very slow

### `interpolateColors(input, inputRange, outputRange)`
Like `interpolate` but interpolates between CSS color strings.

```tsx
import { interpolateColors } from 'remotion';

const color = interpolateColors(frame, [0, 60], ['#ff0000', '#0000ff']);
```

### `random(seed)`
Deterministic pseudo-random number (same seed always gives same result across frames).

```tsx
import { random } from 'remotion';
const x = random('particle-1'); // 0..1
```

---

## Static Assets

### `staticFile(path)`
Resolves a path relative to the `public/` directory.

```tsx
import { staticFile } from 'remotion';
<Video src={staticFile('intro.mp4')} />
<Img  src={staticFile('logo.png')} />
<Audio src={staticFile('music.mp3')} />
```

### `getStaticFiles()`
Returns all files in `public/` as an array of `StaticFile` objects.

---

## Input Props

Pass data into compositions at render time:

```bash
npx remotion render src/index.ts MyVideo output.mp4 --props='{"title":"Hello"}'
```

```tsx
import { getInputProps } from 'remotion';

const inputProps = getInputProps(); // typed as unknown; cast or use schema
```

Or use `defaultProps` + `schema` on `<Composition>` for typed, validated props.

---

## `@remotion/player`

Embeds a Remotion composition in a React web app for interactive preview.

```bash
npm install @remotion/player
```

```tsx
import { Player } from '@remotion/player';
import { MyVideo } from './MyVideo';

export const App: React.FC = () => (
  <Player
    component={MyVideo}
    durationInFrames={150}
    fps={30}
    compositionWidth={1920}
    compositionHeight={1080}
    style={{ width: '100%' }}
    controls
    inputProps={{ text: 'Hello' }}
  />
);
```

**Player ref methods:** `play()`, `pause()`, `toggle()`, `seekTo(frame)`, `getCurrentFrame()`, `getContainerNode()`, `mute()`, `unmute()`, `setVolume(v)`, `getVolume()`

**Events:** `play`, `pause`, `seeked`, `ended`, `error`, `fullscreenchange`, `scalechange`, `ratechange`

**Hooks:**
- `usePlayer()` — access player state/methods inside a component tree under `<Player>`
- `usePlayback({ compositionDurationInFrames })` — custom playback loop

---

## `@remotion/renderer` (Node.js API)

Server-side rendering without the CLI.

```bash
npm install @remotion/renderer
```

```ts
import { bundle } from '@remotion/bundler';
import { renderMedia, selectComposition } from '@remotion/renderer';

// 1. Bundle
const bundleLocation = await bundle({
  entryPoint: './src/index.ts',
  webpackOverride: (config) => config,
});

// 2. Select composition
const composition = await selectComposition({
  serveUrl: bundleLocation,
  id: 'MyVideo',
  inputProps: { title: 'Hello' },
});

// 3. Render
await renderMedia({
  composition,
  serveUrl: bundleLocation,
  codec: 'h264',
  outputLocation: 'out/video.mp4',
  inputProps: { title: 'Hello' },
  onProgress: ({ progress }) => console.log(`${Math.round(progress * 100)}%`),
});
```

**Key functions:**
| Function | Purpose |
|---|---|
| `renderMedia` | Render to video/audio file |
| `renderFrames` | Render individual frames as images |
| `renderStill` | Render a single still frame |
| `getCompositions` | List all compositions in a bundle |
| `selectComposition` | Get one composition by ID |
| `stitchFramesToVideo` | Combine frames into video |
| `extractAudio` | Extract audio track |
| `getVideoMetadata` | Probe a video file |
| `ensureBrowser` | Download/prepare headless Chrome |

---

## `@remotion/bundler`

Wraps Webpack to bundle your Remotion project for rendering.

```ts
import { bundle } from '@remotion/bundler';

const bundleLocation = await bundle({
  entryPoint: './src/index.ts',
  onProgress: (progress) => console.log(`Bundling: ${progress}%`),
  webpackOverride: (config) => {
    // modify webpack config
    return config;
  },
});
```

---

## `@remotion/lambda`

Render videos at scale on AWS Lambda.

```bash
npm install @remotion/lambda
```

**Setup:**
```bash
npx remotion lambda policies user    # get required IAM policy
npx remotion lambda policies role
npx remotion lambda functions deploy --memory=2048 --timeout=120 --disk=2048
npx remotion lambda sites create src/index.ts --site-name=my-video
```

**Render:**
```ts
import { renderMediaOnLambda, getRenderProgress } from '@remotion/lambda';

const { renderId, bucketName } = await renderMediaOnLambda({
  region: 'us-east-1',
  functionName: 'remotion-render-4-0-0-mem2048mb-disk2048mb-120sec',
  serveUrl: 'https://remotion-render-my-video.s3.amazonaws.com/sites/my-video/index.html',
  composition: 'MyVideo',
  inputProps: { title: 'Hello' },
  codec: 'h264',
  imageFormat: 'jpeg',
});

// Poll for progress
const progress = await getRenderProgress({ renderId, bucketName, region: 'us-east-1', functionName: '...' });
```

---

## CLI Reference

```bash
npx remotion studio [entry]           # Dev server (default: src/index.ts)
npx remotion render [entry] [comp] [out]  # Render video
npx remotion still  [entry] [comp] [out]  # Render still
npx remotion bundle [entry]           # Bundle without rendering
npx remotion compositions [entry]     # List all compositions
npx remotion upgrade                  # Upgrade all @remotion/* packages
npx remotion add                      # Add a Remotion package
npx remotion lambda <subcommand>      # Lambda operations
npx remotion cloudrun <subcommand>    # Cloud Run operations
```

Common render flags:
```bash
--props='{"key":"value"}'  # Input props (JSON)
--codec=h264               # h264 | h265 | vp8 | vp9 | gif | mp3 | aac | wav | prores
--image-format=jpeg        # jpeg | png
--crf=18                   # Quality (lower = better)
--frames=0-100             # Render subset of frames
--concurrency=4            # Parallel browser tabs
--output=out/video.mp4     # Output path
--log=verbose              # Log level: error | warn | info | verbose
```

---

## Project Structure (typical)

```
my-video/
├── public/                 # Static assets (videos, images, audio)
├── src/
│   ├── index.ts            # Entry: calls registerRoot
│   ├── Root.tsx            # Composition declarations
│   ├── compositions/
│   │   └── MyVideo.tsx     # Composition components
│   └── components/         # Shared components
├── remotion.config.ts      # Optional Remotion config
├── package.json
└── tsconfig.json
```

---

## `remotion.config.ts`

```ts
import { Config } from '@remotion/cli/config';

Config.setVideoImageFormat('jpeg');
Config.setOverwriteOutput(true);
Config.setCodec('h264');
Config.setScale(2);           // 2× resolution rendering
Config.overrideWebpackConfig((config) => {
  return config; // modify webpack
});
```

---

## Patterns & Conventions

### Fade in/out
```tsx
const opacity = interpolate(frame, [0, 20, durationInFrames - 20, durationInFrames], [0, 1, 1, 0], {
  extrapolateLeft: 'clamp',
  extrapolateRight: 'clamp',
});
```

### Enter animation with spring
```tsx
const translateY = spring({ frame, fps, from: 50, to: 0, config: { damping: 12 } });
const style = { transform: `translateY(${translateY}px)` };
```

### Staggered animations
```tsx
{items.map((item, i) => {
  const delay = i * 5;  // 5-frame stagger
  const opacity = interpolate(frame - delay, [0, 20], [0, 1], { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' });
  return <div key={i} style={{ opacity }}>{item}</div>;
})}
```

### Data-driven video
```tsx
// Composition with schema
const schema = z.object({ title: z.string(), items: z.array(z.string()) });

<Composition
  id="DataVideo"
  component={DataVideo}
  schema={schema}
  defaultProps={{ title: 'My Title', items: ['A', 'B', 'C'] }}
  durationInFrames={150}
  fps={30}
  width={1920}
  height={1080}
/>
```

### Waiting for async data before rendering
```tsx
const handle = useDelayRender('Fetching data');
useEffect(() => {
  fetchData().then(d => { setData(d); continueRender(handle); });
}, []);
```

---

## Common Gotchas

- **`useCurrentFrame` / `useVideoConfig` only work inside Remotion compositions** — not in plain React apps or outside a `<Composition>`.
- **`interpolate` input range must be strictly ascending** — otherwise it throws.
- **`staticFile()` resolves against `public/`** — put assets there, not `src/`.
- **Avoid side-effects in render** — each frame renders independently; don't mutate external state.
- **`OffthreadVideo` vs `Video`**: use `OffthreadVideo` for rendering (frame-accurate); use `Video` for the Player (browser-native playback).
- **Compositions cannot be nested** — use `<Sequence>` or `<Series>` to compose timing.
- **CSS transforms must be strings**: `transform: \`scale(${scale})\`` — not individual properties in WebKit.
- **All Remotion packages must be on the same version** — run `npx remotion upgrade` to sync.

---

## Licensing

Remotion uses a dual license:
- **Free** for individuals and small companies (revenue < $1M/year or fewer than 3 employees)
- **Paid company license** required beyond those thresholds

Always check `LICENSE.md` in the repo and https://remotion.dev/license for current terms.
