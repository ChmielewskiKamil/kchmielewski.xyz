+++
title = 'Build Your Own Voice-to-Text With whisper.cpp'
date = 2026-03-26
draft = true
tags = ["whisper", "voice", "linux"]
author = "Kamil Chmielewski"
description = "How to build a system-wide voice-to-text tool with whisper.cpp and a shell script. No cloud, no subscription, no app store. Works on any Linux and probably macOS."
+++

A few colleagues mentioned they've been using voice to talk to their coding agents instead of typing. The argument is simple: you can capture intent faster, give more context, and explain what you actually want in the way you'd explain it to another person. Sounded worth trying.

I'll admit this wasn't an easy choice. I have a [Corne](https://github.com/foostan/crkbd) keyboard that I soldered myself and I type at over 100 WPM. I *like* typing. But it turns out that speaking your intent to an agent is a different thing entirely - you don't need to be precise with syntax, you just explain what you want. Now that I have this set up, I use it more than I expected.

So I tried using voice mode in Claude Code:

```
/voice
  ⎿  Voice mode requires SoX for audio recording. Install SoX manually:
       macOS: brew install sox
       Ubuntu/Debian: sudo apt-get install sox
```

Cool. I installed SoX. However, I run Claude Code inside headless, isolated VMs over SSH - there's no microphone in a VM. Anthropic's [voice dictation docs](https://docs.anthropic.com/en/docs/claude-code/voice-dictation) confirm it: "Voice dictation does not work in remote environments such as SSH sessions."

So instead of fighting audio forwarding through PulseAudio SSH tunnels into a headless QEMU VM (I looked into it - it's as fun as it sounds), I built my own voice-to-text that works system-wide.

<!--more-->

## What We're Building

A single keybind that:

1. Starts recording from your microphone
2. Press again - stops recording, transcribes locally with [whisper.cpp](https://github.com/ggml-org/whisper.cpp)
3. Copies the result to your clipboard (and optionally types it into the focused window)

Works everywhere - browser, terminal, Slack, Claude on the web, your text editor. It's basically [SuperWhisper](https://superwhisper.com/) but free, open source, and it's a shell script.

## What Is whisper.cpp?

[Whisper](https://github.com/openai/whisper) is OpenAI's speech recognition model. It was trained on 680,000 hours of multilingual audio data and it's remarkably good at transcribing speech - including technical jargon, accented English, and mixed-language input.

[whisper.cpp](https://github.com/ggml-org/whisper.cpp) is a C/C++ port of that model by Georgi Gerganov (the same person behind [llama.cpp](https://github.com/ggml-org/llama.cpp)). It runs the model locally on your CPU with no dependencies, no Python, no PyTorch. Just a single binary and a model file. It runs on everything from a Raspberry Pi to a Mac with Apple Silicon.

The reason this matters: apps like SuperWhisper literally use whisper.cpp under the hood. You're paying for a GUI wrapper around a C++ binary. [Wispr Flow](https://wisprflow.ai/) takes a different approach - it sends your audio to the cloud for transcription and then runs AI on top to clean up your speech. For our purposes, local is better. No internet required, no subscription, your audio never leaves your machine.

**What this post is not**: a comprehensive guide to speech recognition. We're building one specific thing: press a key, talk, get text. If you want real-time streaming transcription or custom language models, look elsewhere.

## Step 1: Verify It Works

Before writing any scripts, let's test whisper.cpp on your machine. You need two things: something to record audio, and whisper.cpp to transcribe it.

For recording, there are a few options depending on your platform:
- **pw-record** - comes with PipeWire, available on most modern Linux distros. This is what I ended up using (more on why [later](#the-sox-buffer-trap)).
- **SoX** (`rec` command) - cross-platform, works on Linux and macOS. Available via `brew`, `apt`, `nix`, etc.
- **arecord** - ALSA tool, available on most Linux systems out of the box.

For transcription, you need `whisper-cpp`. On NixOS you can grab everything with `nix-shell -p whisper-cpp sox`. On other platforms, check the [whisper.cpp build instructions](https://github.com/ggml-org/whisper.cpp#build) or your package manager. Homebrew has it as `whisper-cpp`.

### Download a Model

```bash
mkdir -p ~/.local/share/whisper
curl -L -o ~/.local/share/whisper/ggml-base.en.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

This is the base English model (~142MB). Good enough for dictation. By "good enough" I mean it handles everything I've thrown at it - coding discussions, writing, technical jargon - without issues. I haven't bothered testing the larger models. There are [other sizes](#model-options) if you need them.

### Record and Transcribe

```bash
# Record a few seconds, Ctrl+C to stop
rec /tmp/test.wav rate 16k channels 1

# Transcribe
whisper-cli -m ~/.local/share/whisper/ggml-base.en.bin -f /tmp/test.wav
```

If you see your words in the terminal, you're good. On my machine (Ryzen 7 7840U), 16 seconds of audio transcribes in 1.2 seconds. Even 60-second recordings come back almost instantly. Transcription speed is not a bottleneck here.

**Note**: The package is called `whisper-cpp`, but the binary is `whisper-cli`. Welcome to open source naming conventions.

## Step 2: The Script

The idea is simple: a toggle script. Run it once to start recording, run it again to stop and transcribe. Bind it to a key, and you've got system-wide voice-to-text.

```bash
#!/usr/bin/env bash
set -euo pipefail

PIDFILE="/tmp/voice-to-text.pid"
WAVFILE="/tmp/voice-to-text-recording.wav"
MODEL="$HOME/.local/share/whisper/ggml-base.en.bin"

if [ -f "$PIDFILE" ]; then
    # === STOP: kill recording, transcribe, copy to clipboard ===
    PID=$(cat "$PIDFILE")
    rm -f "$PIDFILE"

    kill -INT "$PID" 2>/dev/null
    wait "$PID" 2>/dev/null

    # Transcribe and join lines into a single string
    RESULT=$(whisper-cli \
        -m "$MODEL" \
        -f "$WAVFILE" \
        --no-timestamps \
        -t 4 \
        2>/dev/null | tr '\n' ' ' | sed 's/^[[:space:]]*//;s/[[:space:]]*$//;s/  */ /g')

    rm -f "$WAVFILE"

    if [ -n "$RESULT" ]; then
        # Copy to clipboard
        echo -n "$RESULT" | wl-copy    # Wayland
        # echo -n "$RESULT" | pbcopy    # macOS
    fi
else
    # === START: begin recording ===
    rm -f "$WAVFILE"

    pw-record --rate 16000 --channels 1 --format s16 "$WAVFILE" &
    # rec -q "$WAVFILE" rate 16k channels 1 &  # SoX alternative (see caveat below)

    echo $! > "$PIDFILE"
fi
```

Save it as `voice-to-text` somewhere in your `$PATH`, make it executable, and you're done.

The clipboard command depends on your platform:
- **Wayland**: `wl-copy` (from `wl-clipboard`)
- **macOS**: `pbcopy`

### Optional: Auto-Type Into Focused Window

On Wayland, you can use [wtype](https://github.com/atx/wtype) to inject the text directly into whatever window is focused - as if you typed it. Add this after the clipboard copy:

```bash
wtype -d 10 -- "$RESULT" 2>/dev/null
```

This is nice for browser text fields and chat apps, but can be flaky in terminals. The clipboard copy is the reliable fallback.

## Step 3: Bind It to a Key

How you do this depends on your desktop environment.

**Hyprland** (what I use):
```
bind = SUPER, V, exec, voice-to-text
```

**GNOME:**
Settings → Keyboard → Custom Shortcuts → add `voice-to-text`

**KDE:**
System Settings → Shortcuts → Custom Shortcuts → add `voice-to-text`

**Sway:**
```
bindsym $mod+v exec voice-to-text
```

**macOS:**
You could use Automator or Shortcuts to bind a hotkey to the script. I haven't tried this myself.

Now press your keybind once to start recording, and again to stop. The transcription lands in your clipboard.

## The SoX Buffer Trap

My first version used `sox rec` for recording. It worked, except the last 1-2 seconds of speech were consistently cut off. Every time.

I tried longer sleep values. Didn't help. The problem isn't timing.

SoX uses PulseAudio's "simple API" which calls `pa_simple_read()` in a blocking loop. When you send SIGINT, this blocking call gets interrupted, and any audio sitting in PulseAudio/PipeWire's server-side buffer (up to ~2 seconds worth) is silently abandoned. There's no `pa_simple_drain()` for recording - that function only exists for playback.

**The fix**: use `pw-record` instead if you're on PipeWire (most modern Linux distros). It talks directly to PipeWire with no intermediate buffer to lose. When you `kill -INT` it, the process finishes writing the file and exits cleanly.

If you're stuck with SoX (e.g., on macOS or a system without PipeWire), it might work fine there since the audio stack is different. I only hit this issue on Linux with PipeWire's PulseAudio compatibility layer. If you experience cut-off audio, try `parecord` or `arecord` as alternatives:

```bash
# PulseAudio native
parecord --rate=16000 --channels=1 --format=s16le --file-format=wav "$WAVFILE" &

# ALSA direct (bypasses audio server entirely)
arecord -f S16_LE -r 16000 -c 1 -t wav "$WAVFILE" &
```

## Model Options

whisper.cpp supports several model sizes. All are available from the [Hugging Face repo](https://huggingface.co/ggerganov/whisper.cpp):

| Model | Size | Speed | Use Case |
|-------|------|-------|----------|
| `ggml-tiny.en` | 75MB | Fastest | Quick commands, lower accuracy |
| `ggml-base.en` | 142MB | Fast | Good balance - what I use |
| `ggml-small.en` | 466MB | Medium | Better accuracy for accents |
| `ggml-medium.en` | 1.5GB | Slower | Near-perfect transcription |

The `.en` suffix means English-only. Drop it for multilingual support (e.g., `ggml-base.bin`).

For dictating prompts to an LLM or typing messages, `base.en` is more than enough. It handles technical vocabulary surprisingly well - regex, OAuth, JSON, NixOS, Ethereum - all transcribed correctly without any custom configuration.

## NixOS: The Declarative Version

If you're on NixOS, you can wrap this whole thing into a Home Manager module instead of managing a loose script. Here's what I use:

```nix
{pkgs, ...}: let
  whisperModel = "ggml-base.en.bin";
  whisperModelUrl = "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/${whisperModel}";

  voice-to-text = pkgs.writeShellScriptBin "voice-to-text" ''
    PIDFILE="/tmp/voice-to-text.pid"
    WAVFILE="/tmp/voice-to-text-recording.wav"
    MODEL="$HOME/.local/share/whisper/${whisperModel}"

    if [ ! -f "$MODEL" ]; then
      mkdir -p "$(dirname "$MODEL")"
      ${pkgs.curl}/bin/curl -L -o "$MODEL" "${whisperModelUrl}"
    fi

    if [ -f "$PIDFILE" ]; then
      PID=$(cat "$PIDFILE")
      rm -f "$PIDFILE"

      kill -INT "$PID" 2>/dev/null
      wait "$PID" 2>/dev/null

      RESULT=$(${pkgs.whisper-cpp}/bin/whisper-cli \
        -m "$MODEL" -f "$WAVFILE" --no-timestamps -t 4 \
        2>/dev/null | tr '\n' ' ' | sed 's/^[[:space:]]*//;s/[[:space:]]*$//;s/  */ /g')

      rm -f "$WAVFILE"

      if [ -n "$RESULT" ]; then
        echo -n "$RESULT" | ${pkgs.wl-clipboard}/bin/wl-copy
        ${pkgs.wtype}/bin/wtype -d 10 -- "$RESULT" 2>/dev/null
      fi
    else
      rm -f "$WAVFILE"
      ${pkgs.pipewire}/bin/pw-record --rate 16000 --channels 1 --format s16 "$WAVFILE" &
      echo $! > "$PIDFILE"
    fi
  '';
in {
  home.packages = [
    voice-to-text
    pkgs.whisper-cpp
    pkgs.wtype
  ];
}
```

Import it, add the Hyprland keybind, rebuild. The model auto-downloads on first use.

## What About Headless VMs?

This was the original problem that sent me down this rabbit hole. The answer is: don't try to forward your microphone to a headless VM. I researched PulseAudio SSH reverse tunnels, PipeWire network discovery, and RTP multicast. All theoretically possible, all practically miserable.

Run voice-to-text on your host machine (where the microphone is) and paste into the SSH session. The transcription lands in your clipboard. Sometimes the simplest solution is the right one.

## References

- [whisper.cpp](https://github.com/ggml-org/whisper.cpp) - The C/C++ port of OpenAI's Whisper
- [OpenAI Whisper](https://github.com/openai/whisper) - The original model
- [Whisper model files](https://huggingface.co/ggerganov/whisper.cpp) - Pre-converted GGML models
- [wtype](https://github.com/atx/wtype) - Wayland keyboard input simulator
- [PipeWire](https://pipewire.org/) - The audio system that actually works
- [SuperWhisper](https://superwhisper.com/) - The paid macOS app that wraps whisper.cpp
- [Wispr Flow](https://wisprflow.ai/) - Cloud-based alternative with AI text cleanup
