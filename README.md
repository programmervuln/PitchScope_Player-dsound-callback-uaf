# PitchScope Player – Potential Heap Use-After-Free in Audio Player Lifecycle (CWE-416)

## 1. Overview

**Product:** PitchScope Player  
**Vulnerability Type:** Heap Use-After-Free  
**CWE:** CWE-416  
**Investigated Component:** Audio Player Lifecycle / DirectSound Notification Processing  
**Upstream Repository:** https://github.com/CreativeDetectors/PitchScope_Player

This repository documents an investigation into a potential heap use-after-free involving audio player object lifetime management in PitchScope Player.

The investigation focuses on the interaction between `EventMan`, `SoundHelper`, and the audio player during DirectSound notification processing and object destruction.

## 2. Affected Component and Source Code Analysis

The upstream implementation contains the following relevant components:

- `StreamingAudioplayer`: manages audio playback and associated resources.
- `EventMan::RegisterAudioPlayer()`: registers the active audio player.
- `EventMan::UNregisterAudioPlayer()`: clears the registered audio player pointer.
- `EventMan::Process_Event_Notification_PPlayer()`: processes playback notification events.
- `SoundHelper::Release_AudioPlayer()`: releases the audio player object.
- `PitchPlayerDlg::Player_Messaging()`: processes DirectSound notification events.

### 2.1 Audio Player Registration

In `EventMan.cpp`:

```cpp
void EventMan::RegisterAudioPlayer(
    StreamingAudioplayer* buffr)
{
    m_curAudioPlayer = buffr;
}

void EventMan::UNregisterAudioPlayer()
{
    m_curAudioPlayer = NULL;
}
```

The active audio player is maintained through a raw pointer.

### 2.2 Audio Player Access

In `EventMan::Process_Event_Notification_PPlayer()`, the registered player is accessed during notification processing:

```cpp
if (m_curAudioPlayer == NULL ||
    !m_curAudioPlayer->IsOK() ||
    bitSource == NULL)
{
    return;
}

SPitchCalc* sPitchCalcer =
    m_curAudioPlayer->Get_SPitchCalc();

double playSpeedFlt =
    m_curAudioPlayer->m_playSpeedFlt;
```

The notification-processing function accesses player members through `m_curAudioPlayer`.

### 2.3 Audio Player Destruction

In `SoundHelper.cpp`:

```cpp
void SoundHelper::Release_AudioPlayer()
{
    if (m_audioPlayer == NULL)
        return;

    if (m_audioPlayer ==
        Get_EventMan().m_curAudioPlayer)
    {
        Get_EventMan().UNregisterAudioPlayer();
    }

    delete m_audioPlayer;
    m_audioPlayer = NULL;
}
```

The player is unregistered and subsequently destroyed.

### 2.4 Potential Object-Lifetime Issue

The registration, notification-processing, and destruction paths involve shared raw-pointer references to the audio player.

If notification processing retains a pointer to the player while another execution path destroys the object, subsequent access through the retained pointer could result in a heap use-after-free.

The exact concurrency conditions and reachability of this scenario in the original application require further validation.

## 3. Proof of Concept

The repository includes a standalone C++ reproduction harness:

`poc_dsound_callback_uaf.cpp`

The harness demonstrates a heap use-after-free caused by concurrent object destruction and access.

The reproduction follows these steps:

1. Allocate an `AudioStaticPlayer` object using `new`.
2. Create a worker thread that repeatedly invokes `DsoundCallback()`.
3. Delete the player object from the main thread while the worker thread is still running.
4. The worker thread subsequently accesses the object's audio buffer through the dangling pointer.

The PoC models the suspected audio-player object-lifetime race. It does not implement the original Windows DirectSound notification mechanism.

## 4. AddressSanitizer Evidence

The included AddressSanitizer report confirms a heap-use-after-free in the standalone reproduction.

**Reported error:**

```text
ERROR: AddressSanitizer: heap-use-after-free
READ of size 1 at 0x6060000000e0
```

**Invalid access:**

```text
AudioStaticPlayer::DsoundCallback()
poc_dsound_callback_uaf.cpp:19
```

**Allocation:**

```text
operator new(unsigned long)
main
poc_dsound_callback_uaf.cpp:39
```

**Deallocation:**

```text
operator delete(void*)
main
poc_dsound_callback_uaf.cpp:45
```

**Accessing thread:**

```text
Thread T1
callback_thread(void*)
poc_dsound_callback_uaf.cpp:31
```

**Memory allocation size:**

```text
64-byte heap region
```

The report demonstrates that the worker thread accesses memory belonging to an object that has already been deallocated by the main thread.

The complete report is available in:

`asan_dsound_callback_uaf.log`

## 5. Reproduction Environment

| Item | Configuration |
|---|---|
| Operating System | Ubuntu x86_64 |
| Compiler | Clang++ |
| Instrumentation | AddressSanitizer |
| Threading | POSIX pthreads |
| Vulnerability Class | Heap Use-After-Free |

### Compilation

```bash
clang++ -g \
-fsanitize=address \
-fno-omit-frame-pointer \
-pthread \
poc_dsound_callback_uaf.cpp \
-o poc_dsound_callback_uaf
```

### Execution

```bash
./poc_dsound_callback_uaf
```

AddressSanitizer reports a heap-use-after-free when the worker thread accesses the deleted object.

## 6. Root Cause of the Reproduction

The standalone reproduction does not synchronize object destruction with concurrent access.

The main thread deletes the heap-allocated object while the worker thread retains a raw pointer to it.

The worker thread subsequently invokes a member function that reads the object's memory.

This results in a heap-use-after-free.

## 7. Potential Security Impact

A use-after-free in the original application, if reachable through its actual playback and notification lifecycle, could potentially result in:

- Application crashes.
- Unexpected memory access.
- Memory corruption, depending on the allocation and reuse conditions.

The practical security impact and attacker-controlled reachability have not yet been established.

No claim of remote code execution or arbitrary code execution is made.

## 8. Suggested Mitigation

Potential safeguards for the audio player lifecycle include:

- Stop playback before destroying the player.
- Unregister the player before releasing its resources.
- Ensure pending notification processing has completed.
- Synchronize object destruction with all execution paths that access the player.
- Prevent access through previously retained raw pointers after object destruction.

## 9. Investigation Status

The upstream source analysis identifies the audio player registration, notification-processing, and destruction paths.

The standalone PoC confirms a heap-use-after-free under the constructed concurrent execution conditions, supported by AddressSanitizer.

Further validation is required to establish whether the same object-lifetime violation is reachable in the original PitchScope Player application.

## 10. Repository Artifacts

| File | Description |
|---|---|
| `poc_dsound_callback_uaf.cpp` | Standalone C++ UAF reproduction |
| `asan_dsound_callback_uaf.log` | AddressSanitizer report |
| `Screenshot from 2026-10-12 10-45-13.png` | Supporting debugging evidence |
| `Screenshot from 2026-10-12 10-45-22.png` | Supporting debugging evidence |

## References

- PitchScope Player: https://github.com/CreativeDetectors/PitchScope_Player
- CWE-416: https://cwe.mitre.org/data/definitions/416.html

---

**Disclaimer:** This repository documents memory-safety research and an investigation into a potential vulnerability in PitchScope Player. The standalone reproduction demonstrates the described UAF pattern but does not independently establish exploitability in the original application.
