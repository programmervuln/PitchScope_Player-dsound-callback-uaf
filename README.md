# PitchScope_Player-dsound-callback-uaf
# PitchScope_Player Dsound Callback Heap Use-After-Free Vulnerability

PoC for Heap Use-After-Free (CWE-416) in PitchScope_Player `AudioStaticPlayer` DirectSound callback implementation.

## Vulnerability Summary
A race condition exists between the main thread and the DirectSound callback thread:
1. Main thread allocates an `AudioStaticPlayer` object.
2. Main thread spawns a callback worker thread which invokes `AudioStaticPlayer::DsoundCallback()` to read object members.
3. Main thread immediately deletes the `AudioStaticPlayer` instance while the callback thread is still executing and accessing the freed object memory.
4. This results in **heap-use-after-free (UAF)** when the callback thread reads member data on the already freed heap memory.

CWE: [CWE-416: Use After Free](https://cwe.mitre.org/data/definitions/416.html)

## Crash Log (AddressSanitizer)

=================================================================
==3993==ERROR: AddressSanitizer: heap-use-after-free on address 0x6060000000e0 at pc 0x5555ea7e57e5 bp 0x7fee865fed80 sp 0x7fee865fed78
READ of size 1 at 0x6060000000e0 thread T1
#0 0x5555ea7e57e4 in AudioStaticPlayer::DsoundCallback() /home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf.cpp:19:18
#1 0x5555ea7e555f in callback_thread(void*) /home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf.cpp:31:19
#2 0x7fee88a94a82 in start_thread nptl/./nptl/pthread_create.c:442:8
#3 0x7fee88b268df misc/../sysdeps/unix/sysv/linux/x86_64/clone3.S:81
0x6060000000e0 is located 0 bytes inside of 64-byte region [0x6060000000e0,0x606000000120)
freed by thread T0 here:
#0 0x5555ea7e379d in operator delete(void*) (/home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf+0xdc79d) (BuildId: daf2dc51f19a130d5180bb36ba91f93b754b6776)
#1 0x5555ea7e56a8 in main /home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf.cpp:45:5
#2 0x7fee88a29d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
previously allocated by thread T0 here:
#0 0x5555ea7e2f3d in operator new(unsigned long) (/home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf+0xdbf3d) (BuildId: daf2dc51f19a130d5180bb36ba91f93b754b6776)
#1 0x5555ea7e5643 in main /home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf.cpp:39:16
#2 0x7fee88a29d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
Thread T1 created by T0 here:
#0 0x5555ea7915ec in __interceptor_pthread_create (/home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf+0x8a5ec) (BuildId: daf2dc51f19a130d5180bb36ba91f93b754b6776)
#1 0x5555ea7e5680 in main /home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf.cpp:41:5
#2 0x7fee88a29d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
SUMMARY: AddressSanitizer: heap-use-after-free /home/lloyd/Documents/PitchScope_Player-master/poc_dsound_callback_uaf.cpp:19:18 in AudioStaticPlayer::DsoundCallback()



## Reproduction
### Environment
- OS: Ubuntu x86_64
- Compiler: Clang with AddressSanitizer
- Target: `poc_dsound_callback_uaf.cpp`

### Build Command
```bash
clang++ -g -fsanitize=address,undefined -pthread poc_dsound_callback_uaf.cpp -o poc_dsound_callback_uaf

Run
./poc_dsound_callback_uaf

The program will trigger ASan heap-use-after-free crash reliably under race condition.
Root Cause
The program does not synchronize object lifetime between main thread and callback thread:
Main thread deletes the AudioStaticPlayer instance without waiting for the audio callback thread to finish execution.
The callback thread retains a raw pointer to the object and continues accessing its member variables after deletion.
Suggested Fix
Add reference counting (std::shared_ptr) or lifetime guards to AudioStaticPlayer.
Use mutex/condition variable to block object destruction until the callback thread exits completely.
Nullify callback pointers and wait for pending callbacks before releasing object memory.
Disclaimer
This PoC is for security research purposes only. Use only on systems you own or have authorization to test.


simplified version
```markdown
# PitchScope_Player Dsound Callback UAF PoC

CWE-416 Heap Use-After-Free vulnerability in PitchScope_Player AudioStaticPlayer DirectSound callback.

### Vulnerability
Race condition between main thread and callback thread: main thread deletes AudioStaticPlayer object while worker callback thread is still reading its member fields, leading to heap-use-after-free.

### Build
```bash
clang++ -g -fsanitize=address,undefined -pthread poc_dsound_callback_uaf.cpp -o poc_dsound_callback_uaf

Run
./poc_dsound_callback_uaf
ASan crash occurs at AudioStaticPlayer::DsoundCallback() when reading freed heap memory.
Mitigation
Synchronize object destruction; wait for callback thread termination before freeing the object.
For security research only.
