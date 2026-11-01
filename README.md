# PitchScope Player – DirectSound-Related Use-After-Free (CWE-416)

## Overview

This repository documents an investigation into a potential heap use-after-free (UAF) involving the lifetime management of `AudioStaticPlayer` in PitchScope Player.

- **Product:** PitchScope Player
- **Vulnerability Class:** Heap Use-After-Free
- **CWE:** CWE-416
- **Investigated Component:** `AudioStaticPlayer`
- **Related Source Files:**
  - `Src/comnAudio/AudioPlayer.cpp`
  - `Src/comnAudio/AudioPlayer.h`
  - `Src/comnAudio/EventMan.cpp`
  - `Src/comnAudio/EventMan.h`

## Technical Background

PitchScope Player uses DirectSound notification events to manage audio playback.

The upstream implementation includes:

- `AudioStaticPlayer`, which manages static audio playback.
- `EventMan::RegisterAudioPlayer()`, which registers the current audio player.
- `EventMan::UNregisterAudioPlayer()`, which clears the registered player pointer.
- DirectSound notification event handles used for playback event processing.

The lifetime relationship between the audio player, notification processing, and object destruction requires further investigation.

## Proof of Concept

The included PoC is a standalone C++ reproduction harness demonstrating a heap use-after-free caused by concurrent object destruction and access.

The test creates an `AudioStaticPlayer` object, starts a worker thread that repeatedly accesses its member data, and deletes the object while the worker thread is still running.

The resulting access can occur after the object has been freed.

**Important:** The current PoC simulates the object-lifetime race. It does not implement the actual Windows DirectSound callback mechanism or independently prove that the same race occurs in the original PitchScope Player application.

## AddressSanitizer Evidence

AddressSanitizer reports:

```text
ERROR: AddressSanitizer: heap-use-after-free

READ of size 1 at 0x6060000000e0

freed by thread T0:
    operator delete(void*)
    main

previously allocated by thread T0:
    operator new(unsigned long)
    main

Thread T1 created by T0:
    pthread_create

SUMMARY:
AddressSanitizer: heap-use-after-free
```

The report identifies:

- A 64-byte heap allocation.
- Deallocation by the main thread.
- Subsequent access by a worker thread.
- A read operation inside `AudioStaticPlayer::DsoundCallback()`.

The complete report is available in `asan_dsound_callback_uaf.log`.

## Reproduction

### Environment

- OS: Ubuntu x86_64
- Compiler: Clang++
- Instrumentation: AddressSanitizer
- Threading: POSIX pthreads

### Build

Rename `poc.cpp.txt` to `poc.cpp`, then compile:

```bash
clang++ -g -fsanitize=address -fno-omit-frame-pointer -pthread poc.cpp -o poc_dsound_callback_uaf
```

### Run

```bash
./poc_dsound_callback_uaf
```

AddressSanitizer may report a heap-use-after-free when the worker thread accesses the object after deallocation.

## Root Cause of the Reproduction

The standalone reproduction does not synchronize object destruction with concurrent access.

The main thread deletes the object while the worker thread retains a raw pointer to it.

The worker thread subsequently accesses a member of the freed object, resulting in a heap-use-after-free.

## Suggested Mitigation

For the corresponding application lifecycle, investigate the following safeguards:

1. Stop audio playback before destroying the player.
2. Unregister the audio player before releasing its resources.
3. Ensure pending notification events have completed.
4. Synchronize object destruction with all threads or callbacks that may access the object.
5. Release the object only after its users can no longer access it.

## Investigation Status

The standalone PoC confirms a heap-use-after-free under the constructed test conditions.

Further investigation is required to establish whether the same object-lifetime violation is reachable in the original PitchScope Player implementation and whether it has a practical security impact.

No claim of remote code execution or arbitrary code execution is made.

## References

- [PitchScope Player upstream repository](https://github.com/CreativeDetectors/PitchScope_Player)
- [CWE-416: Use After Free](https://cwe.mitre.org/data/definitions/416.html)

## Repository Artifacts

| File | Description |
|---|---|
| `poc.cpp` | Standalone C++ UAF reproduction |
| `asan_dsound_callback_uaf.log` | AddressSanitizer report |
| Screenshot files | Supporting debugging evidence |

## Disclaimer

This repository is intended for memory-safety research and vulnerability investigation.
