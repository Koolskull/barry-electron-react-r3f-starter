# FMV Video Game Guide for Barry

## Congratulations Barry!

Dude, building your own Full Motion Video (FMV) game in Three.js / React Three Fiber is seriously cool! FMV games like the classic 90s ones (Night Trap, Phantasmagoria) but in modern 3D with interactive elements — that's next-level creativity. Turning video footage into an immersive 3D experience shows real vision.

This guide helps integrate your existing FMV project into the Electron desktop app.

## Playing Videos in R3F

Use `VideoTexture` from Three.js.

Example code:
```tsx
import { useVideoTexture } from '@react-three/drei';
// or manual HTMLVideoElement + Texture
```

## Desktop Enhancements
- Fullscreen video playback
- Native menus for loading FMV clips
- Combine with Genesis emulator for hybrid retro/FMV game

Keep crushing it!