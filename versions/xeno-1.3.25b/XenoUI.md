`XenoUI.dll` выполняет инъекцию `Xeno.dll` в целевой процесс. Поскольку `XenoUI` — это .NET сборка, а инъекция - низкоуровневая операция с Windows API, код использует **Platform Invoke (P/Invoke)** для вызова функций из `kernel32.dll` и `user32.dll`.

Я восстановлю логику на основе типичного поведения таких экзекьюторов и функций, которые видны в дизассемблированном коде.

### Общий алгоритм инъекции

Когда вы нажимаете кнопку "Attach" (`buttonAttach_Click`), происходит следующая последовательность действий:

#### Шаг 1: Поиск целевого процесса (Roblox)

Прежде чем внедрять DLL, нужно найти окно или процесс Roblox. Судя по именам функций в коде, используется несколько методов:

*   **Поиск по имени окна:** Функция `FindWindowW` (импортируется из `user32.dll`) ищет окно с заголовком, содержащим "Roblox".
    ```asm
    ; Пример вызова FindWindowW (восстановлено)
    call    cs:FindWindowW
    test    rax, rax
    jz      short loc_... ; Если окно не найдено, выйти
    mov     hWnd, rax
    ```
*   **Получение ID процесса (PID):** После получения дескриптора окна (`hWnd`), вызывается `GetWindowThreadProcessId`, чтобы узнать PID процесса, которому принадлежит это окно.
    ```asm
    ; Получение PID по HWND
    lea     rdx, [rsp+28h+ProcessId] ; Адрес для сохранения PID
    mov     rcx, hWnd                 ; Дескриптор окна
    call    cs:GetWindowThreadProcessId
    ```
*   **(Альтернатива) Поиск по имени процесса:** В коде также видны обращения к .NET классу `System.Diagnostics.Process`. Это более высокоуровневый способ найти процесс по имени (например, "RobloxPlayerBeta").

#### Шаг 2: Открытие процесса с необходимыми правами

Когда PID найден, код должен открыть процесс с правами, достаточными для инъекции. Для этого используется функция `OpenProcess` с флагами доступа.

```asm
; Открытие процесса для инъекции
mov     rcx, 1F0FFFh  ; Флаг PROCESS_ALL_ACCESS (максимальные права)
mov     rdx, 0         ; bInheritHandle = FALSE
mov     r8d, [rsp+28h+ProcessId] ; PID целевого процесса
call    cs:OpenProcess
mov     [rsp+28h+hProcess], rax ; Сохраняем дескриптор процесса
test    rax, rax
jz      short loc_... ; Если не удалось открыть, выйти с ошибкой
```

#### Шаг 3: Выделение памяти в целевом процессе

Необходимо место для записи пути к `Xeno.dll`. Используется `VirtualAllocEx`, чтобы выделить память **внутри адресного пространства Roblox**.

```asm
; Выделение памяти в удаленном процессе
mov     rcx, hProcess               ; Дескриптор процесса
xor     rdx, rdx                     ; lpAddress = NULL (пусть система выберет адрес)
mov     r8d, 100h                    ; dwSize = 256 байт (достаточно для пути)
mov     r9d, 3000h                   ; flAllocationType = MEM_COMMIT | MEM_RESERVE
push    40h ; '@'                     ; flProtect = PAGE_EXECUTE_READWRITE
call    cs:VirtualAllocEx
mov     [rsp+28h+lpRemoteMem], rax   ; Сохраняем адрес выделенной памяти
test    rax, rax
jz      short loc_cleanup            ; Если не удалось выделить, идти на очистку
```

#### Шаг 4: Запись пути к DLL в удаленную память

Теперь нужно записать строку с полным путем к `Xeno.dll` (например, `"C:\Users\...\Xeno.dll"`) в только что выделенную память. Это делается с помощью `WriteProcessMemory`.

```asm
; Запись строки в память удаленного процесса
mov     rcx, hProcess                ; Дескриптор процесса
mov     rdx, lpRemoteMem              ; Адрес в удаленном процессе
lea     r8, [rsp+28h+szDllPath]       ; Адрес строки с путем в памяти XenoUI
mov     r9d, [rsp+28h+szDllPathLength] ; Длина строки (включая нуль-терминатор)
call    cs:WriteProcessMemory
test    rax, rax
jz      short loc_cleanup             ; Если не удалось записать, идти на очистку
```

#### Шаг 5: Создание удаленного потока для загрузки DLL

Самый важный шаг. Код заставляет целевой процесс выполнить функцию `LoadLibraryW` (или `LoadLibraryA`), передав ей адрес пути к DLL. Это делается с помощью `CreateRemoteThread`. Функция `LoadLibraryW` является стандартной экспортируемой функцией всех Windows-приложений (из `kernel32.dll`).

```asm
; Получение адреса функции LoadLibraryW
mov     rcx, offset aKernel32_dll    ; "kernel32.dll"
call    cs:GetModuleHandleW           ; Получаем базовый адрес kernel32.dll
mov     rcx, rax                       ; hModule
mov     rdx, offset aLoadLibraryW     ; "LoadLibraryW"
call    cs:GetProcAddress              ; Получаем адрес функции LoadLibraryW
mov     rbx, rax                       ; Сохраняем адрес LoadLibraryW

; Создание удаленного потока
mov     rcx, hProcess                 ; Дескриптор процесса
xor     rdx, rdx                       ; lpThreadAttributes = NULL
mov     r8d, 0                          ; dwStackSize = 0 (использовать размер по умолчанию)
mov     r9, rbx                          ; lpStartAddress = Адрес LoadLibraryW
push    lpRemoteMem                     ; lpParameter = Адрес пути к DLL в удаленной памяти
push    0                                ; dwCreationFlags = 0 (поток запустится сразу)
push    0                                ; lpThreadId = NULL (не интересует ID)
call    cs:CreateRemoteThread
mov     [rsp+28h+hRemoteThread], rax
test    rax, rax
jz      short loc_cleanup             ; Если не удалось, идти на очистку
```

В этот момент `Xeno.dll` загружается в процесс Roblox, и его функция `DllMain` начинает выполняться.

#### Шаг 6: Ожидание и очистка

После успешного создания потока, UI обычно ожидает его завершения с помощью `WaitForSingleObject`, а затем закрывает все открытые дескрипторы (`CloseHandle`), чтобы не оставлять мусор.

```asm
; Ожидание завершения работы загруженной DLL (необязательно)
mov     rcx, hRemoteThread
mov     edx, 0FFFFFFFFh               ; dwMilliseconds = INFINITE
call    cs:WaitForSingleObject

; Очистка дескрипторов
mov     rcx, hRemoteThread
call    cs:CloseHandle
mov     rcx, hProcess
call    cs:CloseHandle
```

### Итог

Весь процесс инъекции строится на классической технике **DLL Injection через удаленный поток (CreateRemoteThread)**. Код выполняет следующие WinAPI функции в строгом порядке:

1.  **FindWindowW / GetWindowThreadProcessId** — чтобы найти процесс.
2.  **OpenProcess** — чтобы получить доступ к процессу.
3.  **VirtualAllocEx** — чтобы выделить память внутри процесса.
4.  **WriteProcessMemory** — чтобы записать путь к `Xeno.dll` в эту память.
5.  **GetModuleHandleW / GetProcAddress** — чтобы найти адрес `LoadLibraryW`.
6.  **CreateRemoteThread** — чтобы заставить процесс выполнить `LoadLibraryW` и загрузить вашу DLL.

Имена функций и строк, которые вы привели в листинге (особенно в секции `.text`), полностью соответствуют этому алгоритму.
