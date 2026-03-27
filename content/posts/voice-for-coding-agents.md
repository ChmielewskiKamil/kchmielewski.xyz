+++
title = 'I type at 100 WPM and I switched to voice for coding agents'
date = 2026-03-26
draft = true
tags = ["linux", "agents", "local-llm"]
author = "Kamil Chmielewski"
description = "I type at 100 WPM on a hand-soldered Corne keyboard, but speaking to coding agents turned out to be a different game entirely. Here's how I built system-wide voice-to-text with whisper.cpp and a shell script."
+++

A few colleagues of mine mentioned they've been using voice to talk to their coding agents instead of typing. The argument is simple: you can capture intent faster, give more context, and explain what you actually want in the way you'd explain it to another person. Sounded worth trying.

Look, I have a [Corne](https://github.com/foostan/crkbd) keyboard that I soldered myself and I type at over 100 WPM. I use Neovim. I *like* typing - there's something about getting your hands dirty with precise word editing that's hard to let go of. So I was hesitant to even try voice input. But speaking to an agent turns out to be a fundamentally different thing, and I'd recommend that everyone at least give it a go.

When you type a prompt, you think first, then type. You edit, you refine, the agent sees the polished result. When you speak, you can't delete what you said. Your stumbles, your corrections, your entire thought process from start to finish lands in the prompt. This is a double-edged sword - sometimes the raw stream of consciousness is noise, but if your model is smart enough (and for Claude Code, it is), the agent gets something valuable: not just your conclusion, but how you got there. That extra context genuinely helps.

There's another thing I noticed. I'm a strong believer that if you're not writing things down, you're not really thinking. Speaking out loud works the same way. When you try to express something in words, any gaps in your understanding show up immediately. It's different from silently thinking about a problem in your head where you can convince yourself you understand something you actually don't.

If you haven't tried voice input with a coding agent, I'd recommend giving it a go even without a permanent setup. There's something about speaking to your machine that feels like we're living in the future - Tony Stark talking to Jarvis. It sounds silly until you try it and realize you don't want to go back.

Naturally, I tried using the built-in voice mode in Claude Code and failed miserably:

```
/voice
  ⎿  Voice mode requires SoX for audio recording. Install SoX manually:
       macOS: brew install sox
       Ubuntu/Debian: sudo apt-get install sox
```

Cool. I installed SoX. However, I run Claude Code inside headless, isolated VMs over SSH - there's no microphone in a VM. Anthropic's [voice dictation docs](https://docs.anthropic.com/en/docs/claude-code/voice-dictation) confirm it: "Voice dictation does not work in remote environments such as SSH sessions."

And that's how the rabbit hole started. Instead of fighting audio forwarding through PulseAudio SSH tunnels into a headless QEMU VM (I looked into it - it's as fun as it sounds), I built my own voice-to-text that works system-wide.

<!--more-->

## What We're Building

{{< video src="/videos/demo_web.mp4" alt="Demo: pressing a keybind to record speech, whisper.cpp transcribes it locally, and the text appears in the clipboard and focused window" >}}

A single keybind that:

1. Starts recording from your microphone
2. Stops recording and transcribes locally with [whisper.cpp](https://github.com/ggml-org/whisper.cpp) on the next press
3. Copies the result to your clipboard (and optionally types it into the focused window)

Works everywhere - browser, terminal, Slack, Claude on the web, your text editor. It's basically [SuperWhisper](https://superwhisper.com/) but free, open source, and it's a shell script.

## What is whisper.cpp?

[Whisper](https://github.com/openai/whisper) is OpenAI's speech recognition model. It was trained on 680,000 hours of multilingual audio data and it's remarkably good at transcribing speech - including technical jargon, accented English, and mixed-language input.

[whisper.cpp](https://github.com/ggml-org/whisper.cpp) is a C/C++ port of that model by Georgi Gerganov (the same person behind [llama.cpp](https://github.com/ggml-org/llama.cpp)). I'm not an ML engineer, but as far as I understand it: OpenAI released Whisper as a Python program that relies on PyTorch - a large ML framework. Gerganov rewrote the inference code (the part that *runs* the model, not trains it) in pure C/C++. The model weights - the actual "brain" - stay the same. He just wrote a new, minimal program that loads those weights and runs them without Python or PyTorch, with hand-optimized code for specific CPU instructions.

The result is a single binary and a model file. No dependencies. It runs on everything from a Raspberry Pi to a Mac with Apple Silicon. And because Whisper is a small, specialized model (74 million parameters), it runs almost instantly on any modern CPU. For context, this is the same idea behind llama.cpp which I use to run a local Qwen 3.5 model for coding - same C/C++ port approach, but coding models are 1000x larger so the speed difference is night and day.

Apps like [SuperWhisper](https://superwhisper.com/) and [Wispr Flow](https://wisprflow.ai/) offer polished UIs on top of this, and some colleagues told me they have fully local and offline options too. I just find tinkering with this stuff interesting, so I went with a shell script.

## Step 1: Verify It Works

Before writing any scripts, let's test whisper.cpp on your machine. You need two things: something to record audio, and whisper.cpp to transcribe it.

For recording, there are a few options depending on your platform:
- **pw-record** - comes with PipeWire, available on most modern Linux distros. This is what I use.
- **SoX** (`rec` command) - cross-platform, works on Linux and macOS. Available via `brew`, `apt`, `nix`, etc.
- **arecord** - ALSA tool, available on most Linux systems out of the box.

For transcription, you need `whisper-cpp`. On NixOS you can grab everything with `nix-shell -p whisper-cpp pipewire`. On other platforms, check the [whisper.cpp build instructions](https://github.com/ggml-org/whisper.cpp#build) or your package manager. Homebrew has it as `whisper-cpp`.

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
pw-record --rate 16000 --channels 1 --format s16 /tmp/test.wav

# Transcribe
whisper-cli -m ~/.local/share/whisper/ggml-base.en.bin -f /tmp/test.wav
```

If you see your words in the terminal, you're good. On my machine (Ryzen 7 7840U), 16 seconds of audio transcribes in 1.2 seconds. Even 60-second recordings come back almost instantly. Transcription speed is not a bottleneck here.

**Note**: The package is called `whisper-cpp`, but the binary is `whisper-cli`. Welcome to open source naming conventions.

## Step 2: The Script

The idea is simple: a toggle script. Run it once to start recording, run it again to stop and transcribe. Bind it to a key, and you've got system-wide voice-to-text.

The trick is a PID file. When you run the script, it checks if a PID file exists. If it doesn't, we're not recording yet, so start. If it does, a recording is in progress, so stop it and transcribe.

```bash
#!/usr/bin/env bash
set -euo pipefail

PIDFILE="/tmp/voice-to-text.pid"
WAVFILE="/tmp/voice-to-text-recording.wav"
MODEL="$HOME/.local/share/whisper/ggml-base.en.bin"

# PID file exists = recording in progress, so stop and transcribe
if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE")
    rm -f "$PIDFILE"

    # Send SIGINT (not SIGTERM) so pw-record flushes the audio buffer to disk
    kill -INT "$PID" 2>/dev/null
    wait "$PID" 2>/dev/null

    # Transcribe the recording
    # --no-timestamps: plain text output, no [00:00:00.000 --> ...] prefixes
    # 2>/dev/null: suppress whisper's progress bar and debug output
    # whisper outputs one line per segment, so we join them into a single string
    RESULT=$(whisper-cli \
        -m "$MODEL" \
        -f "$WAVFILE" \
        --no-timestamps \
        2>/dev/null | tr '\n' ' ' | xargs)

    rm -f "$WAVFILE"

    if [ -n "$RESULT" ]; then
        echo -n "$RESULT" | wl-copy    # Wayland
        # echo -n "$RESULT" | pbcopy    # macOS
    fi

# No PID file = not recording, so start
else
    rm -f "$WAVFILE"

    # Record mono 16kHz WAV - the format whisper expects
    # Runs in the background (&) so the script can exit while recording continues
    pw-record --rate 16000 --channels 1 --format s16 "$WAVFILE" &

    # Save the background process ID so we can stop it on the next run
    echo $! > "$PIDFILE"
fi
```

Save it as `voice-to-text` somewhere in your `$PATH`, make it executable, and you're done.

The clipboard command depends on your platform: `wl-copy` (from `wl-clipboard`) on Wayland, `pbcopy` on macOS.

### Optional: Auto-Type Into Focused Window

The demo video above uses [wtype](https://github.com/atx/wtype) to simulate keyboard input, which gives you that nice typing animation. You can add this after the clipboard copy:

```bash
wtype -d 10 -- "$RESULT" 2>/dev/null
```

Full disclosure: wtype is a low-star GitHub repo with its last commit from 2022. It works well for this use case, but I'm not fully comfortable recommending it from a supply chain perspective. Honestly, just pasting from clipboard with Ctrl+V is faster and more reliable. The typing animation looks cool in a demo but in practice you just want the text in your window as fast as possible.

## Step 3: Bind It to a Key

Any window manager or desktop environment lets you bind a shell command to a hotkey. I use Hyprland:

```
bind = SUPER, V, exec, voice-to-text
```

On macOS, you can use Automator, Shortcuts, Keyboard Maestro, or skhd to bind a hotkey to the script.

Now press your keybind once to start recording, and again to stop. The transcription lands in your clipboard.

## Model Options

whisper.cpp supports several model sizes. All are available from the [Hugging Face repo](https://huggingface.co/ggerganov/whisper.cpp):

| Model | Size | Speed | Use Case |
|-------|------|-------|----------|
| `ggml-tiny.en` | 75MB | Fastest | Quick commands, lower accuracy |
| `ggml-base.en` | 142MB | Fast | Good balance - what I use |
| `ggml-small.en` | 466MB | Medium | Better accuracy for accents |
| `ggml-medium.en` | 1.5GB | Slower | Near-perfect transcription |

The `.en` suffix means English-only. Drop it for multilingual support (e.g., `ggml-base.bin`).

For dictating with a decent microphone, `base.en` is all you need. It handles technical vocabulary surprisingly well - regex, OAuth, JSON, NixOS, Ethereum - all transcribed correctly without any custom configuration. If you have a bad microphone or a heavy accent, the larger models might help. If you want to go the other direction, `tiny.en` might work just as well - I haven't bothered testing it because base is already fast enough that the transcription feels instant.

One thing I noticed: whisper occasionally misspells proper nouns. For example, "Claude Code" consistently becomes "cloud code". If that bothers you, whisper.cpp has an `--initial-prompt` flag that lets you prime the model with words you expect:

```bash
whisper-cli --initial-prompt "Claude Code, NixOS, Hyprland" ...
```

This biases the model toward specific spellings. I haven't set this up myself because it feels like overkill for dictation that goes into an LLM anyway, but if there's a word that matters to you, it's there.

## What About Headless VMs?

This was the original problem that sent me down this rabbit hole. The answer is: don't try to forward your microphone to a headless VM. I researched PulseAudio SSH reverse tunnels, PipeWire network discovery, and RTP multicast. All theoretically possible, all practically miserable.

Run voice-to-text on your host machine (where the microphone is) and paste into your SSH session. The transcription lands in your clipboard, the audio never leaves your machine, and no network-accessible audio service is exposed. If you can paste text somewhere, this approach works.

I use this for Claude Code over SSH, but it works for literally any text field - ChatGPT in the browser, Slack, note-taking apps, even writing this blog post.

If you're feeling lazy, you could even have the script press Enter automatically after pasting. I decided against it because the use case varies - sometimes you want to review the transcription before sending, sometimes you want to edit a word or two. Keeping it as a clipboard paste gives you that flexibility.

## References

- [whisper.cpp](https://github.com/ggml-org/whisper.cpp) - The C/C++ port of OpenAI's Whisper
- [OpenAI Whisper](https://github.com/openai/whisper) - The original model
- [Whisper model files](https://huggingface.co/ggerganov/whisper.cpp) - Pre-converted GGML models
- [wtype](https://github.com/atx/wtype) - Wayland keyboard input simulator
- [PipeWire](https://pipewire.org/) - Modern Linux audio system
- [SuperWhisper](https://superwhisper.com/) - The paid macOS app that wraps whisper.cpp
- [Wispr Flow](https://wisprflow.ai/) - Cloud-based alternative with AI text cleanup
