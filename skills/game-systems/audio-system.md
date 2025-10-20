---
name: audio-system
description: Audio system for games including sound effects, music, spatial audio, and audio management with Web Audio API
---

# Audio System

## When to Use

Use this skill when:
- Playing sound effects
- Managing background music
- Implementing spatial 3D audio
- Creating audio feedback for actions
- Building audio mixing system
- Handling audio asset loading

## Core Principles

1. **Web Audio API**: Use modern audio capabilities
2. **Spatial Audio**: 3D positioned sounds
3. **Audio Pooling**: Reuse audio sources
4. **Volume Control**: Master, music, and SFX volumes
5. **Performance-Aware**: Limit concurrent sounds
6. **Mobile-Friendly**: Handle autoplay restrictions

## Audio System Implementation

### 1. Audio Manager

```typescript
// audio/AudioManager.ts
export interface AudioConfig {
  masterVolume?: number;
  musicVolume?: number;
  sfxVolume?: number;
  maxConcurrentSounds?: number;
}

export class AudioManager {
  private context: AudioContext;
  private masterGain: GainNode;
  private musicGain: GainNode;
  private sfxGain: GainNode;
  private sounds = new Map<string, AudioBuffer>();
  private activeSounds: AudioBufferSourceNode[] = [];
  private maxConcurrentSounds: number;
  private listener: AudioListener | null = null;

  constructor(config: AudioConfig = {}) {
    this.context = new AudioContext();

    // Create gain nodes
    this.masterGain = this.context.createGain();
    this.musicGain = this.context.createGain();
    this.sfxGain = this.context.createGain();

    // Connect gain hierarchy
    this.musicGain.connect(this.masterGain);
    this.sfxGain.connect(this.masterGain);
    this.masterGain.connect(this.context.destination);

    // Set volumes
    this.masterGain.gain.value = config.masterVolume ?? 1;
    this.musicGain.gain.value = config.musicVolume ?? 0.7;
    this.sfxGain.gain.value = config.sfxVolume ?? 1;

    this.maxConcurrentSounds = config.maxConcurrentSounds ?? 32;
  }

  async loadSound(name: string, url: string): Promise<void> {
    try {
      const response = await fetch(url);
      const arrayBuffer = await response.arrayBuffer();
      const audioBuffer = await this.context.decodeAudioData(arrayBuffer);
      this.sounds.set(name, audioBuffer);
    } catch (error) {
      console.error(`Failed to load sound: ${name}`, error);
    }
  }

  async loadSounds(sounds: Record<string, string>): Promise<void> {
    const promises = Object.entries(sounds).map(([name, url]) =>
      this.loadSound(name, url)
    );
    await Promise.all(promises);
  }

  playSound(
    name: string,
    options: {
      volume?: number;
      loop?: boolean;
      playbackRate?: number;
      destination?: GainNode;
    } = {}
  ): AudioBufferSourceNode | null {
    const buffer = this.sounds.get(name);
    if (!buffer) {
      console.warn(`Sound not found: ${name}`);
      return null;
    }

    // Limit concurrent sounds
    if (this.activeSounds.length >= this.maxConcurrentSounds) {
      this.stopOldestSound();
    }

    // Resume audio context (required after user interaction)
    if (this.context.state === 'suspended') {
      this.context.resume();
    }

    // Create source
    const source = this.context.createBufferSource();
    source.buffer = buffer;
    source.loop = options.loop ?? false;
    source.playbackRate.value = options.playbackRate ?? 1;

    // Create gain node for individual sound volume
    const gainNode = this.context.createGain();
    gainNode.gain.value = options.volume ?? 1;

    // Connect to destination (music or sfx)
    source.connect(gainNode);
    gainNode.connect(options.destination ?? this.sfxGain);

    // Track active sound
    this.activeSounds.push(source);

    source.onended = () => {
      const index = this.activeSounds.indexOf(source);
      if (index !== -1) {
        this.activeSounds.splice(index, 1);
      }
    };

    source.start(0);

    return source;
  }

  playSoundAt(
    name: string,
    position: Vector3,
    options: {
      volume?: number;
      loop?: boolean;
      refDistance?: number;
      maxDistance?: number;
      rolloffFactor?: number;
    } = {}
  ): AudioBufferSourceNode | null {
    const buffer = this.sounds.get(name);
    if (!buffer) {
      console.warn(`Sound not found: ${name}`);
      return null;
    }

    if (this.activeSounds.length >= this.maxConcurrentSounds) {
      this.stopOldestSound();
    }

    if (this.context.state === 'suspended') {
      this.context.resume();
    }

    // Create source
    const source = this.context.createBufferSource();
    source.buffer = buffer;
    source.loop = options.loop ?? false;

    // Create panner for 3D positioning
    const panner = this.context.createPanner();
    panner.panningModel = 'HRTF';
    panner.distanceModel = 'inverse';
    panner.refDistance = options.refDistance ?? 1;
    panner.maxDistance = options.maxDistance ?? 10000;
    panner.rolloffFactor = options.rolloffFactor ?? 1;
    panner.coneInnerAngle = 360;
    panner.coneOuterAngle = 0;
    panner.coneOuterGain = 0;

    // Set position
    panner.positionX.value = position.x;
    panner.positionY.value = position.y;
    panner.positionZ.value = position.z;

    // Create gain node
    const gainNode = this.context.createGain();
    gainNode.gain.value = options.volume ?? 1;

    // Connect: source -> panner -> gain -> destination
    source.connect(panner);
    panner.connect(gainNode);
    gainNode.connect(this.sfxGain);

    this.activeSounds.push(source);

    source.onended = () => {
      const index = this.activeSounds.indexOf(source);
      if (index !== -1) {
        this.activeSounds.splice(index, 1);
      }
    };

    source.start(0);

    return source;
  }

  stopSound(source: AudioBufferSourceNode): void {
    try {
      source.stop();
    } catch (error) {
      // Already stopped
    }
  }

  stopAllSounds(): void {
    for (const source of this.activeSounds) {
      this.stopSound(source);
    }
    this.activeSounds = [];
  }

  private stopOldestSound(): void {
    if (this.activeSounds.length > 0) {
      const oldest = this.activeSounds.shift()!;
      this.stopSound(oldest);
    }
  }

  setMasterVolume(volume: number): void {
    this.masterGain.gain.value = Math.max(0, Math.min(1, volume));
  }

  setMusicVolume(volume: number): void {
    this.musicGain.gain.value = Math.max(0, Math.min(1, volume));
  }

  setSFXVolume(volume: number): void {
    this.sfxGain.gain.value = Math.max(0, Math.min(1, volume));
  }

  getMasterVolume(): number {
    return this.masterGain.gain.value;
  }

  getMusicVolume(): number {
    return this.musicGain.gain.value;
  }

  getSFXVolume(): number {
    return this.sfxGain.gain.value;
  }

  setListener(position: Vector3, forward: Vector3, up: Vector3): void {
    if (!this.context.listener) return;

    this.context.listener.positionX.value = position.x;
    this.context.listener.positionY.value = position.y;
    this.context.listener.positionZ.value = position.z;

    this.context.listener.forwardX.value = forward.x;
    this.context.listener.forwardY.value = forward.y;
    this.context.listener.forwardZ.value = forward.z;

    this.context.listener.upX.value = up.x;
    this.context.listener.upY.value = up.y;
    this.context.listener.upZ.value = up.z;
  }

  dispose(): void {
    this.stopAllSounds();
    this.context.close();
  }
}
```

### 2. Audio Components

```typescript
// components/AudioSource.ts
export class AudioSource {
  soundName: string;
  volume: number = 1;
  loop: boolean = false;
  playOnStart: boolean = false;
  spatial: boolean = true;
  refDistance: number = 1;
  maxDistance: number = 50;
  rolloffFactor: number = 1;

  private source: AudioBufferSourceNode | null = null;

  constructor(soundName: string, volume: number = 1, loop: boolean = false) {
    this.soundName = soundName;
    this.volume = volume;
    this.loop = loop;
  }

  setSource(source: AudioBufferSourceNode): void {
    this.source = source;
  }

  getSource(): AudioBufferSourceNode | null {
    return this.source;
  }

  isPlaying(): boolean {
    return this.source !== null;
  }
}

// components/AudioListener.ts
export class AudioListener {
  // Marks the entity as the audio listener (usually the camera)
}
```

### 3. Audio System

```typescript
// systems/AudioSystem.ts
import { AudioSource } from '../components/AudioSource';
import { AudioListener } from '../components/AudioListener';
import { Transform } from '../components/Transform';

export class AudioSystem extends UpdateSystem {
  priority = 60; // Late update
  private audioManager: AudioManager;

  constructor(audioManager: AudioManager) {
    super();
    this.audioManager = audioManager;
  }

  update(world: World, deltaTime: number): void {
    // Update listener position
    this.updateListener(world);

    // Handle audio sources
    const sources = world.query<[Transform, AudioSource]>([Transform, AudioSource]);

    sources.iterate((entity, [transform, audioSource]) => {
      // Play on start
      if (audioSource.playOnStart && !audioSource.isPlaying()) {
        this.playAudioSource(audioSource, transform);
        audioSource.playOnStart = false;
      }
    });
  }

  private updateListener(world: World): void {
    const listenerEntity = world
      .query<[Transform, AudioListener]>([Transform, AudioListener])
      .first();

    if (listenerEntity) {
      const transform = listenerEntity.getComponent(Transform);
      const forward = new Vector3(0, 0, -1).applyQuaternion(transform.rotation);
      const up = new Vector3(0, 1, 0).applyQuaternion(transform.rotation);

      this.audioManager.setListener(transform.position, forward, up);
    }
  }

  private playAudioSource(audioSource: AudioSource, transform: Transform): void {
    if (audioSource.spatial) {
      const source = this.audioManager.playSoundAt(
        audioSource.soundName,
        transform.position,
        {
          volume: audioSource.volume,
          loop: audioSource.loop,
          refDistance: audioSource.refDistance,
          maxDistance: audioSource.maxDistance,
          rolloffFactor: audioSource.rolloffFactor,
        }
      );

      if (source) {
        audioSource.setSource(source);
      }
    } else {
      const source = this.audioManager.playSound(audioSource.soundName, {
        volume: audioSource.volume,
        loop: audioSource.loop,
      });

      if (source) {
        audioSource.setSource(source);
      }
    }
  }

  playSound(name: string, volume: number = 1): void {
    this.audioManager.playSound(name, { volume });
  }

  playSoundAt(name: string, position: Vector3, volume: number = 1): void {
    this.audioManager.playSoundAt(name, position, { volume });
  }
}
```

### 4. Music Manager

```typescript
// audio/MusicManager.ts
export class MusicManager {
  private audioManager: AudioManager;
  private currentTrack: AudioBufferSourceNode | null = null;
  private currentTrackName: string | null = null;
  private fadeTime: number = 1; // seconds
  private playlist: string[] = [];
  private playlistIndex: number = 0;
  private shuffled: boolean = false;

  constructor(audioManager: AudioManager) {
    this.audioManager = audioManager;
  }

  play(trackName: string, fadeIn: boolean = true): void {
    if (this.currentTrackName === trackName && this.currentTrack) {
      return; // Already playing
    }

    // Fade out current track
    if (this.currentTrack && fadeIn) {
      this.fadeOut(this.currentTrack, this.fadeTime);
    } else if (this.currentTrack) {
      this.audioManager.stopSound(this.currentTrack);
    }

    // Play new track
    const source = this.audioManager.playSound(trackName, {
      loop: true,
      destination: this.audioManager['musicGain'],
      volume: fadeIn ? 0 : 1,
    });

    if (source) {
      this.currentTrack = source;
      this.currentTrackName = trackName;

      if (fadeIn) {
        this.fadeIn(source, this.fadeTime);
      }
    }
  }

  stop(fadeOut: boolean = true): void {
    if (this.currentTrack) {
      if (fadeOut) {
        this.fadeOut(this.currentTrack, this.fadeTime);
      } else {
        this.audioManager.stopSound(this.currentTrack);
      }

      this.currentTrack = null;
      this.currentTrackName = null;
    }
  }

  setPlaylist(tracks: string[], shuffle: boolean = false): void {
    this.playlist = shuffle ? this.shuffle([...tracks]) : tracks;
    this.playlistIndex = 0;
    this.shuffled = shuffle;
  }

  playNext(): void {
    if (this.playlist.length === 0) return;

    this.playlistIndex = (this.playlistIndex + 1) % this.playlist.length;
    this.play(this.playlist[this.playlistIndex]);
  }

  playPrevious(): void {
    if (this.playlist.length === 0) return;

    this.playlistIndex =
      (this.playlistIndex - 1 + this.playlist.length) % this.playlist.length;
    this.play(this.playlist[this.playlistIndex]);
  }

  private fadeIn(source: AudioBufferSourceNode, duration: number): void {
    const context = this.audioManager['context'];
    const gainNode = context.createGain();
    gainNode.gain.value = 0;

    // Reconnect through gain node
    source.disconnect();
    source.connect(gainNode);
    gainNode.connect(this.audioManager['musicGain']);

    // Fade in
    gainNode.gain.linearRampToValueAtTime(
      1,
      context.currentTime + duration
    );
  }

  private fadeOut(source: AudioBufferSourceNode, duration: number): void {
    const context = this.audioManager['context'];
    const gainNode = context.createGain();
    gainNode.gain.value = 1;

    // Reconnect through gain node
    source.disconnect();
    source.connect(gainNode);
    gainNode.connect(this.audioManager['musicGain']);

    // Fade out
    gainNode.gain.linearRampToValueAtTime(
      0,
      context.currentTime + duration
    );

    // Stop after fade
    setTimeout(() => {
      this.audioManager.stopSound(source);
    }, duration * 1000);
  }

  private shuffle(array: string[]): string[] {
    const shuffled = [...array];
    for (let i = shuffled.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
    }
    return shuffled;
  }

  setFadeTime(seconds: number): void {
    this.fadeTime = seconds;
  }
}
```

### 5. Audio Event System

```typescript
// audio/AudioEventSystem.ts
import { DamageEvent } from '../events/DamageEvent';
import { DeathEvent } from '../events/DeathEvent';
import { JumpEvent } from '../events/JumpEvent';

export class AudioEventSystem extends EventDrivenSystem {
  private audioSystem: AudioSystem;

  constructor(eventBus: EventBus, audioSystem: AudioSystem) {
    super(eventBus);
    this.audioSystem = audioSystem;

    // Subscribe to game events
    this.subscribe(DamageEvent, (event) => this.onDamage(event));
    this.subscribe(DeathEvent, (event) => this.onDeath(event));
    this.subscribe(JumpEvent, (event) => this.onJump(event));
    this.subscribe(CollisionEvent, (event) => this.onCollision(event));
  }

  update(world: World, deltaTime: number): void {
    // Event-driven, no update needed
  }

  private onDamage(event: DamageEvent): void {
    const transform = event.target.getComponent(Transform);
    if (transform) {
      this.audioSystem.playSoundAt('hit', transform.position, 0.5);
    }
  }

  private onDeath(event: DeathEvent): void {
    const transform = event.entity.getComponent(Transform);
    if (transform) {
      this.audioSystem.playSoundAt('death', transform.position, 1);
    }
  }

  private onJump(event: JumpEvent): void {
    const transform = event.entity.getComponent(Transform);
    if (transform) {
      this.audioSystem.playSoundAt('jump', transform.position, 0.7);
    }
  }

  private onCollision(event: CollisionEvent): void {
    const transform = event.entityA.getComponent(Transform);
    if (transform) {
      // Play different sounds based on collision intensity
      const impact = event.relativeVelocity?.length() ?? 0;
      if (impact > 10) {
        this.audioSystem.playSoundAt('impact_heavy', transform.position, 1);
      } else if (impact > 5) {
        this.audioSystem.playSoundAt('impact_medium', transform.position, 0.7);
      } else {
        this.audioSystem.playSoundAt('impact_light', transform.position, 0.4);
      }
    }
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic setup
const audioManager = new AudioManager({
  masterVolume: 1,
  musicVolume: 0.7,
  sfxVolume: 1,
  maxConcurrentSounds: 32,
});

// Load sounds
await audioManager.loadSounds({
  jump: '/sounds/jump.mp3',
  hit: '/sounds/hit.mp3',
  explosion: '/sounds/explosion.mp3',
  music_menu: '/music/menu.mp3',
  music_game: '/music/game.mp3',
});

// Add audio system
const audioSystem = new AudioSystem(audioManager);
systemManager.add(audioSystem);

// Example 2: Play sound effect
audioManager.playSound('jump', { volume: 0.7 });

// Example 3: Spatial audio
const enemyPos = new Vector3(10, 0, 5);
audioManager.playSoundAt('explosion', enemyPos, {
  volume: 1,
  refDistance: 5,
  maxDistance: 50,
});

// Example 4: Audio source component
const enemy = world.createEntity();
enemy.addComponent(new Transform(new Vector3(10, 0, 5)));
const audioSource = enemy.addComponent(new AudioSource('enemy_idle', 0.5, true));
audioSource.playOnStart = true;
audioSource.spatial = true;

// Example 5: Music manager
const musicManager = new MusicManager(audioManager);

// Play track
musicManager.play('music_menu');

// Set up playlist
musicManager.setPlaylist(['track1', 'track2', 'track3'], true);
musicManager.playNext();

// Example 6: Audio listener (camera)
const camera = world.createEntity();
camera.addComponent(new Transform());
camera.addComponent(new AudioListener());

// Example 7: Volume controls
function createVolumeControls(): void {
  const masterSlider = document.getElementById('master-volume') as HTMLInputElement;
  masterSlider.addEventListener('input', (e) => {
    audioManager.setMasterVolume(parseFloat((e.target as HTMLInputElement).value));
  });

  const musicSlider = document.getElementById('music-volume') as HTMLInputElement;
  musicSlider.addEventListener('input', (e) => {
    audioManager.setMusicVolume(parseFloat((e.target as HTMLInputElement).value));
  });

  const sfxSlider = document.getElementById('sfx-volume') as HTMLInputElement;
  sfxSlider.addEventListener('input', (e) => {
    audioManager.setSFXVolume(parseFloat((e.target as HTMLInputElement).value));
  });
}

// Example 8: Handle autoplay restrictions
document.addEventListener('click', () => {
  if (audioManager['context'].state === 'suspended') {
    audioManager['context'].resume();
  }
}, { once: true });
```

## Checklist

- [ ] Create AudioManager instance
- [ ] Load all sound assets
- [ ] Set up audio system
- [ ] Configure volume controls
- [ ] Add audio listener to camera
- [ ] Implement spatial audio for 3D sounds
- [ ] Set up music manager
- [ ] Handle autoplay restrictions
- [ ] Test audio on mobile devices
- [ ] Profile audio performance

## Common Pitfalls

1. **Autoplay blocking**: Audio won't play without user interaction
2. **Too many sounds**: Performance issues
3. **No audio pooling**: Creating too many audio nodes
4. **Forgetting listener**: 3D audio doesn't work
5. **Wrong formats**: Use MP3/OGG for compatibility
6. **Large files**: Long load times
7. **No volume controls**: Users can't adjust audio

## Performance Tips

### Audio Optimization
- Limit concurrent sounds (16-32)
- Use shorter sound files
- Compress audio files
- Pool audio sources
- Stop distant sounds

### Memory Optimization
- Load sounds on demand
- Unload unused sounds
- Use compressed formats (MP3, OGG)
- Share audio buffers

### Mobile Considerations
- Lower quality audio
- Fewer concurrent sounds (<16)
- Simpler spatial audio
- Handle autoplay restrictions
- Provide mute option

## Related Skills

- `ecs-events` - Audio events
- `ecs-system-patterns` - System implementation
- `input-system` - User interaction for autoplay
- `mobile-performance` - Mobile optimization
- `threejs-scene-setup` - Camera for audio listener

## References

- Web Audio API: https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API
- Spatial audio concepts
- Audio compression best practices
- Mobile audio handling
