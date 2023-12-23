# flipper-zero-project

## Code for detecting BAD USB via Flipper Zero

```c
#include <windows.h>
#include <stdio.h>
#include <time.h>

HHOOK hKeyboardHook;
KBDLLHOOKSTRUCT hooked_key;

#define KEYBUFF_SIZE 10
KBDLLHOOKSTRUCT keyBuffer[KEYBUFF_SIZE];
int keyBufferIndex = 0;
int rate;

// Reporter function for logging
void reporter() {
    printf("Rapid keystroke sequence detected. Possible BadUSB.\n");
    // Optionally, add file logging or other reporting mechanisms here
}

// Function to handle the keyboard input
void bufHandle() {
    DWORD currentTime = GetTickCount();

    // Remove old keystrokes
    while ((keyBufferIndex > 0) && (keyBuffer[0].time < (currentTime - 1000))) {
        // Shift everything left
        for (int i = 0; i < keyBufferIndex - 1; i++) {
            keyBuffer[i] = keyBuffer[i + 1];
        }
        keyBufferIndex--;
    }

    // Add new keystroke
    if (keyBufferIndex < KEYBUFF_SIZE) {
        keyBuffer[keyBufferIndex++] = hooked_key;
    }

    // Check the rate
    if (keyBufferIndex == KEYBUFF_SIZE) {
        float keyRate = (float)(keyBuffer[KEYBUFF_SIZE - 1].time - keyBuffer[0].time) / KEYBUFF_SIZE;
        if (keyRate < rate) {
            reporter();
        }
    }
}

// Hook procedure
LRESULT CALLBACK Keylog(int nCode, WPARAM wParam, LPARAM lParam) {
    if ((nCode == HC_ACTION) && ((wParam == WM_SYSKEYDOWN) || (wParam == WM_KEYDOWN))) {
        hooked_key = *((KBDLLHOOKSTRUCT*)lParam);
        bufHandle();
    }
    return CallNextHookEx(hKeyboardHook, nCode, wParam, lParam);
}

// Setting the hook
DWORD WINAPI Hooker(LPVOID lpParm) {
    HINSTANCE hInstance = GetModuleHandle(NULL);
    hKeyboardHook = SetWindowsHookEx(WH_KEYBOARD_LL, Keylog, hInstance, 0);

    MSG message;
    while (GetMessage(&message, NULL, 0, 0)) {
        TranslateMessage(&message);
        DispatchMessage(&message);
    }
    UnhookWindowsHookEx(hKeyboardHook);
    return 0;
}

int main(int argc, char *argv[]) {
    if (argc == 2) {
        rate = atoi(argv[1]);
    } else {
        printf("Default rate (35ms) is being used.\n");
        rate = 35;
    }

    HANDLE hThread;
    hThread = CreateThread(NULL, 0, Hooker, NULL, 0, NULL);
    if (hThread) {
        WaitForSingleObject(hThread, INFINITE);
        CloseHandle(hThread);
    }
    return 0;
}
```

## Output/ Images:

