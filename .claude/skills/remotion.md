# Remotion Skill

Use this skill when the user wants to build or modify videos programmatically using Remotion (React-based video framework). Trigger if the request:
- Mentions Remotion, `@remotion/*`, `remotion`, or creating videos with React
- Involves rendering video frames, compositions, sequences, or animations
- References Remotion-specific APIs: `useCurrentFrame`, `useVideoConfig`, `interpolate`, `spring`, `Composition`, `Sequence`, etc.

---

## What Is Remotion

Remotion lets you build videos using React components. A video is a React component that receives the current frame number as context and renders accordingly. The rendering engine is Chromium-based, so CSS, Canvas, SVG, and WebGL all work. You run a dev server to preview, then render to MP4/GIF/WebM via CLI or Node.js API.

**Core model:** Remotion gives you a frame number and a blank canvas. Your React component uses that frame number to compute what to render. Time-based state is derived, not stored — animations must be driven by `useCurrentFrame()`. CSS transitions and imperative animations cause flickering during rendering because Remotion cannot control their timing.

**Renders are deterministic:** the same frame always produces the same output.

**Current stable version:** 4.0.x
**Peer deps:** React ≥ 16.8, React DOM ≥ 16.8

---

## Getting Started

```bash
npx create-video@latest
```

Scaffolds a new project. Choose a template (blank, hello-world, three.js, TikTok, etc.).

**Dev server:**
```bash
npx remotion studio    # or: npx remotion preview
```

**Render:**
```bash
npx remotion render <entry-file> <composition-id> <output-file>
npx remotion still  <entry-file> <composition-id> <output-file>
```

---

## Entry Point

Every Remotion project needs a root file that calls `registerRoot`:

```tsx
// src/index.ts
import { registerRoot } from 'remotion';
import { RemotionRoot } from './Root';
registerRoot(RemotionRoot);
```

```tsx
// src/Root.tsx
import { Composition } from 'remotion';
import { MyVideo } from './MyVideo';

export const RemotionRoot: React.FC = () => (
  <>
    <Composition
      id="MyVideo"
      component={MyVideo}
      durationInFrames={150}
      fps={30}
      width={1920}
      height={1080}
      defaultProps={{ text: 'Hello' }}
    />
  </>
);
```

---

## Core Components

### `<Composition>`
Registers a renderable video or still. Appears in the Studio sidebar. Cannot be nested inside another `<Composition>` or inside `<Player>`.

```tsx
<Composition
  id="MyVideo"              // required, URL-safe string (letters, numbers, hyphens)
  component={MyVideo}       // or lazyComponent={() => import('./MyVideo').then(m => ({default: m.MyVideo}))}
  durationInFrames={150}    // required for videos (omit for stills)
  fps={30}
  width={1920}
  height={1080}
  defaultProps={{ title: 'Hello' }}  // pure JSON-serializable; overridable from CLI/Studio
  schema={myZodSchema}               // optional Zod schema for Studio props editor
  calculateMetadata={async ({ props, abortSignal }) => ({
    durationInFrames: 200,
    width: 1280,
    height: 720,
    props: { ...props },
  })}
/>
```

`getInputProps()` can be called outside component scope to dynamically compute `durationInFrames`.

### `<AbsoluteFill>`
Shorthand for `position: absolute; top: 0; left: 0; width: 100%; height: 100%`. Used to layer scenes.

```tsx
import { AbsoluteFill } from 'remotion';
<AbsoluteFill style={{ backgroundColor: 'blue' }}>
  <MyScene />
</AbsoluteFill>
```

### `<Sequence>`
Time-shifts children — children only render when the global frame is within the sequence's window. Children see frame 0 when the global frame equals `from`. Sequences cascade: a `<Sequence from={60}>` inside `<Sequence from={30}>` starts at absolute frame 90.

```tsx
import { Sequence } from 'remotion';

<Sequence from={30} durationInFrames={60} name="Intro">
  <IntroScene />
</Sequence>

// No wrapping div:
<Sequence from={0} layout="none">
  <span>inline content</span>
</Sequence>
```

| Prop | Default | Description |
|---|---|---|
| `from` | `0` | Frame at which children mount (and see frame 0 inside) |
| `durationInFrames` | `Infinity` | Frames the children stay mounted |
| `layout` | `"absolute-fill"` | `"absolute-fill"` or `"none"` |
| `name` | — | Label in Studio timeline |
| `premountFor` | — | Frames to mount before `from` |
| `showInTimeline` | `true` | Toggle Studio timeline track |

### `<Series>`
Stacks Sequences back-to-back without manual frame arithmetic.

```tsx
import { Series } from 'remotion';
<Series>
  <Series.Sequence durationInFrames={40}><SceneA /></Series.Sequence>
  <Series.Sequence durationInFrames={60}><SceneB /></Series.Sequence>
</Series>
```

### `<Loop>`
Repeats children for a given number of times or indefinitely.

```tsx
import { Loop } from 'remotion';
<Loop durationInFrames={30} times={5}>
  <AnimatedDot />
</Loop>
```

### `<Freeze>`
Freezes the timeline at a specific frame for its children.

```tsx
import { Freeze } from 'remotion';
<Freeze frame={0}>
  <StaticBackground />
</Freeze>
```

### `<Still>`
Defines a still image composition (single frame). Rendered with `npx remotion still`.

```tsx
<Still id="Thumbnail" component={Thumbnail} width={1200} height={630} />
```

### `<Folder>`
Groups compositions in the Studio sidebar. No effect on rendering.

---

## Media Components

### `<Img>`
Always use `<Img>` instead of `<img>`. Blocks rendering until fully loaded (uses `delayRender` internally).

```tsx
import { Img, staticFile } from 'remotion';
<Img src={staticFile('logo.png')} />
<Img src="https://example.com/remote.png" />
```

### `<Video>` / `<OffthreadVideo>`
- Use `<OffthreadVideo>` for rendering (frame-accurate, extracts frames off main thread via native layer).
- Use `<Video>` for the `<Player>` (browser-native, supports `loop`).

```tsx
import { Video, OffthreadVideo, staticFile } from 'remotion';

<Video src={staticFile('clip.mp4')} trimBefore={30} trimAfter={90} volume={0.5} />
<OffthreadVideo src="https://example.com/video.mp4" />
```

**Key props:** `src`, `volume` (0–1 or per-frame callback `(f) => interpolate(...)`), `muted`, `loop` (Video only), `trimBefore` (frames), `trimAfter` (frames), `playbackRate`, `style`

### `<Audio>`

```tsx
import { Audio, staticFile } from 'remotion';
<Audio src={staticFile('music.mp3')} volume={0.8} trimBefore={0} trimAfter={120} loop />
```

**Key props:** `src`, `volume`, `muted`, `loop`, `trimBefore`, `trimAfter`, `toneFrequency` (pitch shift 0.01–2), `showInTimeline`

### `<IFrame>`
Embeds an iframe and waits for load during rendering.

---

## Core Hooks

### `useCurrentFrame()`
Returns the current frame number (0-indexed integer). Inside a `<Sequence from={10}>`, returns frame relative to sequence start.

```tsx
const frame = useCurrentFrame(); // e.g. 0, 1, 2 ... durationInFrames-1
```

To get the absolute global frame inside a sequence, capture `useCurrentFrame()` above the sequence and pass it as a prop.

### `useVideoConfig()`
Returns the composition's metadata.

```tsx
const { width, height, fps, durationInFrames, id, defaultProps, props } = useVideoConfig();
```

| Property | Description |
|---|---|
| `width` / `height` | Composition dimensions in px |
| `fps` | Frames per second — always pass this to `spring()` |
| `durationInFrames` | Total frames of composition (or enclosing Sequence) |
| `id` | Composition ID string |
| `defaultProps` | The `defaultProps` defined on `<Composition>` |
| `props` | Effective props after runtime overrides |

### `useDelayRender()` / `delayRender()` / `continueRender()` / `cancelRender()`
Pauses rendering until async work completes. Must call `continueRender` within 30 seconds.

```tsx
import { delayRender, continueRender, cancelRender } from 'remotion';

const handle = delayRender('Fetching data');
fetchData()
  .then(d => { setData(d); continueRender(handle); })
  .catch(e => cancelRender(e));
```

---

## Animation APIs

### `interpolate(input, inputRange, outputRange, options?)`
Maps a number from one range to another. Core to all time-based animations. Pure function — usable outside Remotion.

```tsx
import { interpolate } from 'remotion';

// Fade in over 30 frames
const opacity = interpolate(frame, [0, 30], [0, 1], {
  extrapolateRight: 'clamp',
});

// Fade in, hold, fade out
const opacity2 = interpolate(
  frame,
  [0, 20, durationInFrames - 20, durationInFrames],
  [0, 1, 1, 0]
);
```

| Option | Values | Default |
|---|---|---|
| `extrapolateLeft` | `'extend'`, `'clamp'`, `'identity'`, `'wrap'` | `'extend'` |
| `extrapolateRight` | same | `'extend'` |
| `easing` | Easing function | identity |

`inputRange` must be strictly monotonically increasing.

### `spring({ frame, fps, config?, from?, to?, durationInFrames?, delay?, reverse? })`
Physics-based animation. Always pass `fps` from `useVideoConfig()`.

```tsx
import { spring, useCurrentFrame, useVideoConfig } from 'remotion';

const frame = useCurrentFrame();
const { fps } = useVideoConfig();

const scale = spring({ frame, fps, from: 0, to: 1 });

// Delayed spring
const slideIn = spring({ frame: frame - 20, fps, config: { damping: 200 } });
```

| Param | Default | Description |
|---|---|---|
| `frame` | required | Use `frame - N` to delay by N frames |
| `fps` | required | From `useVideoConfig()` |
| `from` | `0` | Start value |
| `to` | `1` | End value (may overshoot before settling) |
| `delay` | `0` | Frames before animation starts |
| `reverse` | `false` | Play backwards |
| `durationInFrames` | — | Stretch spring to exact duration |
| `config.mass` | `1` | Lower = faster |
| `config.damping` | `10` | Higher = less bounce; increase to eliminate overshoot |
| `config.stiffness` | `100` | Affects bounciness |
| `config.overshootClamping` | `false` | Clamp at `to`, no overshoot |

Use the interactive editor at [springs.remotion.dev](https://springs.remotion.dev) to tune config.

### `measureSpring({ fps, config?, threshold? })`
Returns the duration (in frames) of a spring animation. Useful for layout planning without trial-and-error.

```tsx
import { measureSpring } from 'remotion';
const duration = measureSpring({ fps: 30, config: { damping: 200 } });
```

### `Easing`
Mirrors the React Native `Easing` API exactly.

```tsx
import { Easing, interpolate } from 'remotion';

const y = interpolate(frame, [0, 30], [0, 300], {
  easing: Easing.bezier(0.8, 0.22, 0.96, 0.65),
  extrapolateRight: 'clamp',
});
```

Methods: `Easing.linear`, `Easing.quad`, `Easing.cubic`, `Easing.poly(n)`, `Easing.bezier(x1,y1,x2,y2)`, `Easing.back(s)`, `Easing.bounce`, `Easing.elastic(b)`, `Easing.in(fn)`, `Easing.out(fn)`, `Easing.inOut(fn)`

### `interpolateColors(input, inputRange, outputRange)`
Like `interpolate` but between CSS color strings. Returns `rgba(r, g, b, a)`. Supports named colors, hex, rgb, rgba, hsl, hsla. Pure function.

```tsx
import { interpolateColors } from 'remotion';
const color = interpolateColors(frame, [0, 20], ['#ff0000', '#0000ff']);
```

### `random(seed)`
Deterministic pseudo-random number (same seed → same result across all frames). Returns 0–1.

```tsx
import { random } from 'remotion';
const x = random('particle-1');
```

---

## Static Assets

### `staticFile(path)`
Resolves a file in the `public/` directory. Only works for files in `public/` — do not pass relative paths or external URLs.

```tsx
import { staticFile } from 'remotion';
<Video src={staticFile('intro.mp4')} />
<Img   src={staticFile('logo.png')} />
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
const { title } = getInputProps<{ title: string }>();
```

`getInputProps()` can be called outside components (e.g., to compute dynamic `durationInFrames`).

Use `defaultProps` + `schema` (Zod) on `<Composition>` for type-safe, visually editable props in Studio.

---

## `@remotion/player`

Install: `npm install @remotion/player`

Embeds a Remotion composition as an interactive player in any React app.

```tsx
import { Player, PlayerRef } from '@remotion/player';
import { MyVideo } from './MyVideo';

const App: React.FC = () => {
  const playerRef = useRef<PlayerRef>(null);
  return (
    <Player
      ref={playerRef}
      component={MyVideo}
      inputProps={{ title: 'Hello' }}
      durationInFrames={150}
      fps={30}
      compositionWidth={1920}
      compositionHeight={1080}
      controls
      loop
      style={{ width: '100%' }}
    />
  );
};
```

**Key props:** `component`, `inputProps`, `durationInFrames`, `fps`, `compositionWidth`, `compositionHeight`, `controls`, `loop`, `autoPlay`, `clickToPlay`, `doubleClickToFullscreen`, `spaceKeyToPlayOrPause`, `showVolumeControls`, `allowFullscreen`, `playbackRate` (−4 to 4, not 0), `initialFrame`, `alwaysShowControls`, `renderLoading`, `errorFallback`, `bufferStateDelayInMilliseconds`

**`PlayerRef` methods:** `play()`, `pause()`, `toggle()`, `seekTo(frame)`, `getCurrentFrame()`, `isPlaying()`, `getVolume()`, `setVolume(v)`, `mute()`, `unmute()`, `isMuted()`, `isFullscreen()`, `requestFullscreen()`, `exitFullscreen()`, `getContainerNode()`, `getScale()`, `addEventListener(event, cb)`, `removeEventListener(event, cb)`

**Events:** `play`, `pause`, `ended`, `seeked`, `timeupdate` (~250ms throttle), `frameupdate` (every frame), `ratechange`, `volumechange`, `mutechange`, `fullscreenchange`, `scalechange`, `error`, `waiting`, `resume`

**`<Thumbnail>`** — renders a single static frame:
```tsx
import { Thumbnail } from '@remotion/player';
<Thumbnail component={MyVideo} frameToDisplay={30} compositionWidth={1920} compositionHeight={1080} style={{ width: 400 }} />
```

**Best practice:** Keep player controls as siblings of `<Player>`, not parents, to avoid re-renders on every `frameupdate`.

---

## `@remotion/renderer` (Node.js / SSR)

Install: `npm install @remotion/renderer @remotion/bundler`

```ts
import { bundle } from '@remotion/bundler';
import { renderMedia, selectComposition } from '@remotion/renderer';

// 1. Bundle the project
const serveUrl = await bundle({ entryPoint: './src/index.ts' });

// 2. Select the composition
const composition = await selectComposition({
  serveUrl,
  id: 'MyVideo',
  inputProps: { title: 'Hello' },
});

// 3. Render
await renderMedia({
  composition,
  serveUrl,
  codec: 'h264',
  outputLocation: 'out/video.mp4',
  inputProps: { title: 'Hello' },
  onProgress: ({ progress }) => console.log(`${Math.round(progress * 100)}%`),
});
```

| Function | Purpose |
|---|---|
| `bundle()` | Webpack-bundle the project to a serve URL |
| `getCompositions()` | List all compositions in a bundle |
| `selectComposition()` | Get one composition by ID |
| `renderMedia()` | Render to video/audio file |
| `renderFrames()` | Render individual frames as images |
| `renderStill()` | Render a single frame |
| `stitchFramesToVideo()` | Assemble frames into video |
| `extractAudio()` | Extract audio track |
| `getVideoMetadata()` | Probe a video file |
| `ensureBrowser()` | Download/prepare headless Chrome |
| `makeCancelSignal()` | Create cancellation token |

---

## `@remotion/lambda`

Install: `npm install @remotion/lambda`

```bash
# Deploy
npx remotion lambda policies user
npx remotion lambda functions deploy --memory=2048 --timeout=120 --disk=2048
npx remotion lambda sites create src/index.ts --site-name=my-video
```

```ts
import { renderMediaOnLambda, getRenderProgress } from '@remotion/lambda';

const { renderId, bucketName } = await renderMediaOnLambda({
  region: 'us-east-1',
  functionName: 'remotion-render-4-0-0-mem2048mb-disk2048mb-120sec',
  serveUrl: 'https://...',
  composition: 'MyVideo',
  inputProps: { title: 'Hello' },
  codec: 'h264',
});

const progress = await getRenderProgress({ renderId, bucketName, region: 'us-east-1', functionName: '...' });
```

Requires: `REMOTION_AWS_ACCESS_KEY_ID` and `REMOTION_AWS_SECRET_ACCESS_KEY` env vars.

---

## Package Ecosystem

| Package | Role |
|---|---|
| `remotion` | Core: hooks, animation, components, utilities |
| `@remotion/cli` | CLI (`studio`, `render`, `still`, `bundle`, `compositions`) |
| `@remotion/renderer` | Server-side rendering: `renderMedia()`, `renderStill()`, `getCompositions()` |
| `@remotion/bundler` | Webpack bundling: `bundle()` |
| `@remotion/player` | `<Player>` and `<Thumbnail>` components |
| `@remotion/lambda` | AWS Lambda distributed rendering |
| `@remotion/transitions` | `<TransitionSeries>` with timing presets for scene transitions |
| `@remotion/media-utils` | `useAudioData()`, waveform visualization, media metadata |
| `@remotion/google-fonts` | Type-safe Google Fonts loader |
| `@remotion/shapes` | SVG shape components: `<Triangle>`, `<Star>`, `<Pie>`, `makeStar()`, etc. |
| `@remotion/paths` | Pure functions for animating and manipulating SVG paths |
| `@remotion/noise` | Simplex/Perlin noise functions |
| `@remotion/gif` | `<Gif>` component for animated GIFs |
| `@remotion/lottie` | Lottie animation integration |
| `@remotion/skia` | React Native Skia integration |
| `@remotion/three` | React Three Fiber / Three.js integration |
| `@remotion/rive` | Rive animation integration |
| `@remotion/motion-blur` | `<Trail>` and `<CameraMotionBlur>` effects |
| `@remotion/animation-utils` | `interpolateStyles()` and style helpers |

**Version discipline: all `@remotion/*` packages and `remotion` must be pinned to the same version.** Run `npx remotion upgrade` to sync. Mismatched versions cause runtime errors.

---

## CLI Reference

```bash
npx remotion studio [entry]                     # Dev server
npx remotion render [entry] [comp] [out]        # Render video
npx remotion still  [entry] [comp] [out]        # Render still image
npx remotion bundle [entry]                     # Bundle without rendering
npx remotion compositions [entry]               # List compositions
npx remotion upgrade                            # Upgrade all @remotion/* packages
npx remotion add                                # Add a Remotion package
npx remotion lambda <subcommand>                # Lambda operations
npx remotion cloudrun <subcommand>              # Cloud Run operations
```

Common render flags:
```bash
--props='{"key":"value"}'   # Input props (JSON)
--codec=h264                # h264 | h265 | vp8 | vp9 | gif | mp3 | aac | wav | prores
--image-format=jpeg         # jpeg | png
--crf=18                    # Quality (lower = better quality)
--frames=0-100              # Render subset of frames
--concurrency=4             # Parallel browser tabs
--output=out/video.mp4
--log=verbose               # error | warn | info | verbose
--scale=2                   # Render at 2× resolution
```

---

## `remotion.config.ts`

```ts
import { Config } from '@remotion/cli/config';

Config.setVideoImageFormat('jpeg');
Config.setOverwriteOutput(true);
Config.setCodec('h264');
Config.setScale(2);
Config.overrideWebpackConfig((config) => config);
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
│   │   └── MyVideo.tsx
│   └── components/
├── remotion.config.ts      # Optional Remotion config
├── package.json
└── tsconfig.json
```

---

## Patterns & Conventions

### Fade in/out
```tsx
const opacity = interpolate(
  frame,
  [0, 20, durationInFrames - 20, durationInFrames],
  [0, 1, 1, 0],
  { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' }
);
```

### Spring enter animation
```tsx
const translateY = spring({ frame, fps, from: 50, to: 0, config: { damping: 12 } });
<div style={{ transform: `translateY(${translateY}px)` }}>Hello</div>
```

### Staggered animations
```tsx
{items.map((item, i) => {
  const delayedFrame = frame - i * 5;
  const opacity = interpolate(delayedFrame, [0, 20], [0, 1], {
    extrapolateLeft: 'clamp', extrapolateRight: 'clamp',
  });
  return <div key={i} style={{ opacity }}>{item}</div>;
})}
```

### Data-driven video with Zod schema
```tsx
const schema = z.object({ title: z.string(), items: z.array(z.string()) });
<Composition id="DataVideo" component={DataVideo} schema={schema}
  defaultProps={{ title: 'My Title', items: ['A', 'B', 'C'] }}
  durationInFrames={150} fps={30} width={1920} height={1080} />
```

### Async data before rendering
```tsx
const handle = delayRender('Fetching data');
useEffect(() => {
  fetchData().then(d => { setData(d); continueRender(handle); });
}, []);
```

### Volume envelope (fade in/out audio)
```tsx
<Audio src={staticFile('music.mp3')}
  volume={(f) => interpolate(f, [0, 10, durationInFrames - 10, durationInFrames], [0, 1, 1, 0], { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' })}
/>
```

---

## Gotchas

- **Never use `<img>`, `<video>`, `<audio>` directly** — always use `<Img>`, `<Video>`/`<OffthreadVideo>`, `<Audio>`. Never use Next.js `<Image>` inside Remotion components.
- **Never use CSS transitions or CSS animations** — they are not frame-driven and cause flickering. All animation must derive from `useCurrentFrame()`.
- **`useCurrentFrame` / `useVideoConfig` only work inside Remotion compositions** — not in plain React apps outside a `<Composition>`.
- **`interpolate` input range must be strictly ascending** — otherwise it throws.
- **`staticFile()` resolves against `public/`** — put assets there, not `src/`. Do not pass relative paths or external URLs to it.
- **Avoid side-effects in render** — each frame renders independently; don't mutate external state.
- **`OffthreadVideo` vs `Video`**: use `OffthreadVideo` for rendering (frame-accurate); use `Video` for the Player (supports `loop`).
- **Compositions cannot be nested** — use `<Sequence>` or `<Series>` to compose timing.
- **CSS `transform` must be a string**: `` transform: `scale(${scale})` `` — not individual `scaleX`/`scaleY` in all WebKit contexts.
- **All Remotion packages must be on the same version** — run `npx remotion upgrade` to sync.
- **`delayRender` must be cleared within 30 seconds** — otherwise the render times out.
- **`getInputProps()` can be called outside components** — useful for computing dynamic `durationInFrames`.

---

## Licensing

Remotion uses a dual license:
- **Free** for individuals and small companies (revenue < $1M/year or fewer than 3 employees)
- **Paid company license** required beyond those thresholds

Always check `LICENSE.md` and https://remotion.dev/license for current terms.

---

*Sources: [remotion-dev/remotion](https://github.com/remotion-dev/remotion), [remotion.dev/docs](https://remotion.dev/docs), [remotion.dev/api](https://remotion.dev/api)*
