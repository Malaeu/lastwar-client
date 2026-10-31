# Сверка ctf-skills с реверсом мобильной игры (Last War: Survival, Android/Unity/xLua)

Источник: локальный клон `/Users/emalam/GitHub/prowka-bot/Projects/ctf-skills/` (197 .md, 14 разделов).
Проверялось под задачу: APK-реверс, Unity/xLua, обход ACE/ShuMei, .NET, разбор трафика, анти-анализ и обход детекта Frida/эмулятора.
Метка «применимо прямо сейчас» = можно взять сегодня и получить результат на реальном клиенте, без переписывания под Android.

Счётчик `new_count` = число пунктов в разделе 1 (техники, забираемые как есть) = **14**.

---

## 1. Применимо прямо сейчас

**С1-1. Правило «не обрезать вывод триажа».** `ctf-reverse/SKILL.md:114-134`. `strings | head` и `sed -n 'N,Mp'` дают ложный вывод «здесь ничего нет». Для 200-МБ L2Dump/`libil2cpp.so` это ровно тот случай, когда обрезанный вывод = неверный вывод. Фильтруй `grep`, не режь объём.

**С1-2. Порядок псевдокод → smali/ассемблер, и правило «Java-слой это роутер».** `ctf-reverse/tools-mcp.md:596-624`. Читай 2-3 не-`R` класса целиком: они укажут ровно на одно из трёх — константа в DEX, ресурс в `strings.xml` (`get_strings`, ловушка-обманки `:555-570`), либо `native` + `System.loadLibrary` (`:572-576`). Для нашей игры это карта: где живёт проверка, там и патч.

**С1-3. Хост-харнесс на JNI: `dlopen` .so и вызов экспорта, без устройства и эмулятора.** `ctf-reverse/tools-mcp.md:626-734`. Ключевое: `unzip app.apk 'lib/*'`, взять `x86_64` ABI, `objdump -T | grep Java_`, подделать `JNIEnv` (в C, не C++ — `:695-697`), стабы bionic и version-node `LIBC` (`:701-721`). Вызов функции-генератора даёт её выход за минуты вместо часа декомпиляции.

**С1-4. Таблица «симптом → куда идти» для харнесса.** `tools-mcp.md:743-762`: `dlsym: undefined symbol Java_...` → `RegisterNatives` (декомпилируй `JNI_OnLoad`); SIGSEGV после `mov 0x<N>(%rdx),%rdx` → пустой слот `JNIEnv`, индекс = `offset/8` (`:764-780`); «возвращает фиксированную строку» → состояние из `JNI_OnLoad` не поднято, иди во Frida; «abort на любом вводе» → это валидатор, харнесс не поможет.

**С1-5. `RegisterNatives`-обфускация: как её распознать и обойти статически.** `ctf-reverse/languages-platforms.md:89-125`. Признак: в Java `native`-метод есть, символа `Java_pkg_Class_method` в .so нет. Дальше — `JNI_OnLoad` → массив `JNINativeMethod{name, sig, fn}` → реальный хендлер. ACE-обвязка в игре ровно так и прячет точки входа.

**С1-6. Frida-детект и его обход хуком самого детекта.** `ctf-reverse/anti-analysis.md:413-462`. Пять векторов: `/proc/self/maps` (`frida`/`gadget`), порт 27042, первые байты `strcmp` (`0xE9`/`0xFF` = inline-хук), имена потоков (`gmain`/`gdbus`), named pipes. Обход — `Interceptor.attach` на `strstr` с `retval.replace(ptr(0))` (`:446-458`). Это базовый слой против ShuMei; ранняя загрузка gadget — `:460-462`.

**С1-7. Android-анти-дебаг в native: `TracerPid`, `su`, `__system_property_get` — и обход статикой.** `languages-platforms.md:413-424`. Не запускать отладчик вообще: по графу найти happy-path и посчитать, какие значения должны лежать в регистрах на каждой ветке.

**С1-8. Целостность кода / self-hash и пять обходов.** `anti-analysis.md:477-511`. Порядок: аппаратные точки останова (не пишут в код), патч сравнения, хук хеш-функции, эмуляция вместо отладки, snapshot+diff. Отдельно — watchdog-поток с CRC в цикле (`:497-510`): убить поток или растянуть `usleep`.

**С1-9. Патч проверок окружения по таблице опкодов.** `ctf-reverse/patterns-runtime.md:19-47`. `JNZ`(0x75)↔`JZ`(0x74), `JLE`→`JGE`, `cmp -1`→`cmp 0`, sleep 150→1. Плюс правило: если патченный клиент шлёт отчёт о системе на сервер — патчить и данные, не только ветку (`:43-46`). Прямо про нашу задачу «обойти условие в клиенте, но не спалиться».

**С1-10. LIEF для правки ELF без дизассемблера.** `ctf-reverse/tools-advanced-2.md:233-257`. Добавление/правка секций `.so`, патч импортов, кросс-формат (ELF/PE/Mach-O). Для вставки трамплина в `libil2cpp.so` — то, что нужно, вместо ручной правки заголовков.

**С1-11. Unity Mono: правка `Assembly-CSharp.dll` → repack → resign.** `ctf-reverse/platforms.md:635-647`. Если сборка Mono (проверить: есть `assets/bin/Data/Managed/*.dll`), то `apktool d` → dnSpy → правка `Update()`/`Start()` → `apktool b` → `jarsigner` → `adb install`. `PlayerPrefs`/`SceneManager` как ориентиры — `:474-479`.

**С1-12. Lua-байткод: `unluac`/`luadec` для 5.1-5.3.** `platforms.md:525-540`. xLua возит `luac`-чанки в `assets`; строки-маркеры `lua_`/`luaL_`/`LUA_` в .so подтверждают наличие Lua-рантайма. Хук `lua_pcall` — точка перехвата выполнения скрипта (`:540`).

**С1-13. Сетевой триаж: `tshark`-фильтры, follow stream, экспорт объектов, статистика диалогов.** `ctf-forensics/network.md:28-70,127-155`. `-Y "tcp.stream eq N"`, `--export-objects http,dir`, `-z conv,ip`. Плюс keylog-файл для TLS (`:74-123`) — при перехвате прод-трафика игры это единственный способ увидеть открытый текст без реверса шифра.

**С1-14. Идентификация шифра по константам и работа с поточным шифром.** `ctf-malware/c2-and-protocols.md:111-123` (AES S-box `0x637c777b`, RC4-инициализация S-box, TEA `0x9E3779B9`, ChaCha20 `expand 32-byte k`) и `:27-33` (склейка всех payload'ов по времени перед расшифровкой; keystream из null-байтов; ключ в `.rodata`). Наш XOR-провод (`lastwar-traffic`) ловится этой же логикой: константы, длина, повторяемость.

---

## 2. Применимо с оговоркой (что адаптировать под Android/Unity)

**A. `.NET`-реверс.** `ctf-reverse/tools.md:355-395`, `ctf-malware/pe-and-dotnet.md:70-92`, `anti-analysis-ctf.md:189-202`.
Оговорка: всё написано под Windows-EXE и dnSpy-GUI. Под нашу задачу переносится так: Unity-Mono DLL открывается ILSpy/`ilspycmd` (кроссплатформенно, `dotnet tool install -g ilspycmd`), патч метода — правка IL в `dnlib`/ILSpy-обратно-в-dll, а не F5-дебаг. Dump-техника ConfuserEx (`<Module> .cctor` + Save Module) валидна и для Unity-сборок, но требует Windows-VM или `dotnet`-рантайма с отладчиком. Шаблон «XOR → Base64 → AES-CBC с key==IV» (`tools.md:367-395`) — типовой для конфигов и клиентских «секретов».

**B. `global-metadata.dat` под IL2CPP.** `ctf-reverse/languages.md:126-136`. Оговорка: при удачном запуске Il2CppDumper — почти чистая победа (имена методов и строковые литералы), но у нас metadata-версия прибита к версии Unity, а сам файл может быть зашифрован. Инструкция `:129` верная: при провале дампера искать путь загрузки metadata в нативном бинаре и точку кастомной расшифровки, а не долбить дампер.

**C. Анти-VM / анти-эмулятор.** `anti-analysis.md:338-407`, `ctf-malware/scripts-and-obfuscation.md:285-321`. Всё про CPUID/`/sys/class/dmi`/реестр. Android-эквиваленты, которых в файлах нет: `ro.kernel.qemu`, `ro.hardware=goldfish/ranchu`, `Build.FINGERPRINT`/`MODEL`, `/dev/qemu_pipe`, `ro.product.manufacturer=Genymotion`. Байпас — не патч, а маскировка: Magisk DenyList + Shamiko, спавн Frida под переименованным сервером и портом, gadget вместо сервера.

**D. Frida на устройстве.** `ctf-reverse/tools-dynamic.md:205-231`. Оговорка: нужен root или gadget в APK; `frida-apk` уже стоит в `~/.local/bin` — это и есть путь без рута, но пересборка ломает подпись → нужен наш же `apksigner`-цикл из `apk-reverse`. Готовый сниппет хука — `:216-227`, мемоизация дорогих рекурсивных вызовов — `:233-281` (полезно на Lua/C# hot-path, где счёт идёт на секунды).

**E. Патч DEX в памяти через `/proc/self/maps`.** `languages-platforms.md:129-152`. Оговорка автора: только Dalvik (API<21), у нас ART — оффлайн-восстановление патчей работает, но сама техника «бинарь патчит DEX на лету» на ART выглядит иначе (нужен hook на `OpenMemory`/`mmap`+`mprotect`, а не сверка maps).

**F. «Новый проект с тем же package/классом → вызвать native напрямую».** `languages-platforms.md:156-178`. Оговорка: ломается на сплитах (base + config-сплиты), на подписи и на Play Integrity — приложение поднимет `ClassNotFoundException` или отвалится на аттестации. Для Last War дешевле JNI-харнесс (С1-3) или Frida, чем новый Android-проект.

**G. Античит-раздел.** `platforms.md:495-523`. Оговорка: EAC/BattlEye/VAC — PC, у нас ACE (Tencent) и ShuMei. Переносим только рамку «идентифицируй систему → обходи конкретную проверку, а не античит целиком» (`:518-522`); конкретика по ACE/ShuMei — в наших `gaming/lastwar-anticheat`.

**H. Эмуляция ARM-кода вместо устройства.** `tools-advanced.md:224-266` (Qiling), `unicorn-emulation.md:1054-1066` (таблица решений). Оговорка: Qiling умеет Android-rootfs, но `libil2cpp.so` тянет bionic + JNI + вызовы в Java — на практике упирается; Unicorn годится для одиночной чистой функции (крипто/скремблер), где нет syscall'ов. Правило выбора из `:1066` («нужны `open`/`socket`/`ptrace` → Qiling») для нашей игры надо читать как «нужен реальный `JNIEnv` → устройство».

**I. `angr`/Z3 по чекеру.** `tools-dynamic.md:286-320`. Оговорка: на 60-МБ `libil2cpp.so` символьное исполнение захлёбывается; работает только точечно — вырезанная функция на 50 инструкций с известными адресами, а не «решить всю игру».

---

## 3. Неприменимо и почему

**N-1. VMProtect / Themida / WinLicense.** `tools-advanced.md:32-75`. x86/x64-протекторы PE. У нас нативная ARM-сборка Unity (IL2CPP), виртуализатора нет; из файла полезна только общая идея «трассируй диспетчер, а не девиртуализируй целиком».

**N-2. Windows-анти-дебаг целиком.** `ctf-reverse/anti-analysis.md:203-335` — PEB/`NtQueryInformationProcess`/heap flags/TLS-callbacks/`NtSetInformationThread`. В Android нет PEB и NTAPI; есть `TracerPid` (см. С1-7).

**N-3. UPX / PyInstaller / PyArmor / кастомные упаковщики PE.** `tools.md:399-416`. В APK другие контейнеры: `apktool`/`jadx`/`unzip`. UPX в `.so` встречается, но редко и не у ACE.

**N-4. WASM, HarmonyOS HAP/ABC, RISC-V/MIPS, UEFI, DOS-стабы.** `tools.md:281-352,430-470`, `languages.md:140-190`. Другой байткод/ISA, к Unity-APK не относятся.

**N-5. GDB и PIE-техника `b *main+0xca`.** `tools.md:54-92`, `SKILL.md:152-160`. На устройстве GDB нет; для `libil2cpp.so` адреса берутся из `Il2CppDumper` + `module base`, а не из символа `main`.

**N-6. `linux-privesc.md` целиком.** `ctf-misc/linux-privesc.md:1-333`. Про подъём с www-data до root на HTB-машинах. К клиенту игры неприменимо, кроме одного совпадения по названию: tcpdump-патч `-Z root -r/-w` (`:27-43`) — если снимаем трафик на рутованом устройстве и пишем в привилегированный путь.

**N-7. Стеганография, дисковая и память-форензика, крипто-задачи.** `ctf-forensics/*` кроме `network*.md`, `ctf-crypto/*`. Другая постановка: у нас нет «найди флаг в образе», есть «пойми протокол и подделай пакет».

**N-8. Сигнальные трюки SIGILL/SIGFPE и parent-patched child dump.** `anti-analysis-ctf.md:15-186`. Это x86-специфика CTF-кракми; в ARM64-игре сигналов и `process_vm_writev` в таком виде нет.

**N-9. `call-less function chaining`, MBA-упрощение, opaque predicates.** `anti-analysis-ctf.md:116-145`, `mba-simplification.md`, `anti-analysis.md:514-560`. IL2CPP-код генерирует компилятор C++ (clang), MBA-мусора в нём почти нет — это оружие против обфускаторов, а не против Unity.

---

## 4. Инструменты, которых нет (существование проверено)

Проверено 2026-10-31: GitHub API (`archived`, `pushed_at`, наличие тегов/ассетов) и PyPI/Homebrew.

**Ставить в первую очередь (закрывают разделы 1-2):**

| Инструмент | Проверка | Зачем |
|---|---|---|
| `objection` | PyPI 1.12.5; `sensepost/objection` живой, push 2026-10-27 | `objection -g pkg explore` → `android sslpinning disable`, `android root disable`, хук классов без написания JS. Быстрый вход в Frida-слой. |
| `jnitrace` | PyPI 3.3.1 (`>=3.0,<4`); репо живое, но push 2023-07-18 | Трасса всех JNI-вызовов с аргументами и стеком. Ровно то, что нужно для `RegisterNatives`-обвязки ACE. Старое, но работает на arm64. |
| `unidbg` | репо `zhkl0228/unidbg` живой, push 2026-10-30, тег v0.9.9 (ассетов нет — сборка maven'ом) | Эмуляция `.so` с фейковым JNI на хосте, без устройства. Тяжёлый, но обходит и анти-дебаг, и анти-Frida, потому что Frida там нет. |
| `Il2CppDumper` v6.7.46 | репо живое, 5 ассетов, тег 2024-07-06 | Символы + offsets из `libil2cpp.so` + `global-metadata.dat`. Без него IL2CPP-путь закрыт. |
| `AssetRipper` 2.0.0 | репо живое, push 2026-10-25, 6 ассетов | Вытащить из бандлов текстовые ассеты, Lua-чанки, ScriptableObject'ы; искать конфиги/строки вне кода. |
| `ilspycmd` | `dotnet` стоит (`/opt/homebrew/bin/dotnet`), `icsharpcode/ILSpy` живой, push 2026-10-29 | Кроссплатформенная декомпиляция Unity-Mono DLL: `dotnet tool install -g ilspycmd`. |
| `de4dot` | репо `de4dot/de4dot` **archived** (2020), 7.4k★, форки мертвы | Снятие обфускации символов после дампа .NET. Архивный, но альтернативы с той же зрелостью нет — берём как есть. |
| `APKiD` 3.2.0 | PyPI 3.2.0; `rednaga/APKiD` живой, push 2026-10-21 | Опознать протектор/упаковщик/анти-тампер по сигнатурам — до того, как тратить день на декомпиляцию. |
| `Androguard` 4.1.4 | PyPI, репо живое | Программный разбор DEX/манифеста, свои проверки по тысячам классов. |
| `ghidra` 12.2 + `ghidra-mcp` | Homebrew `ghidra 12.2`; `bethington/ghidra-mcp` живой, push 2026-10-30 | Головной декомпилятор без GUI (headless) + MCP для агентной работы по `libil2cpp.so`. У нас Ghidra не стоит вообще. |
| `rizin` 0.9.1 + `rz-ghidra` | Homebrew 0.9.1; `rizinorg/rz-ghidra` живой, push 2026-10-22 | Скриптуемый дизассемблер с декомпилятором, заменяет radare2 (у нас нет ни r2, ни rizin). |
| `wireshark`/`tshark` 4.6.9 | Homebrew 4.6.9, репо живое | `tshark` для фильтров/экспорта в скриптах. Сейчас нет ни tshark, ни Wireshark. |
| `apk-mitm` (`niklashigi/apk-mitm`) | репо живое, push 2024-07-24 (переехало с `shroudedcode`) | Патч `network_security_config` + пересборка под MITM. Быстрее, чем ручной патч пиннинга. |
| `unluac` (`viruscamp/unluac`) | живой форк, push 2025-12-19 | Декомпиляция `luac` 5.3 (xLua). Основной репозиторий `mjspi/unluac` отдаёт 404 — брать форк. |
| `LSPosed` + `Shamiko` | `LSPosed/LSPosed` живой, push 2026-10-14 | Скрыть root/хуки от ACE. `JingMatrix/PlayIntegrityFix` — для аттестации (оригинал `chiteroman` отдаёт 404, живёт форк, push 2026-09-30). |

**Уже стоит (проверено на месте, дублировать не надо):** `frida` 17.15.3 (py) / CLI 17.24.3, `frida-tools` 14.10.4, `frida-apk`, `jadx` 1.5.6, `apktool` 3.0.3, `mitmproxy`/`mitmdump`, `adb`, `java`, `dotnet`.
Дефект: версии Frida CLI и python-модуля расходятся (17.24.3 против 17.15.3) — при апгрейде сервера до CLI-версии модуль отвалится, выравнивать `pip install -U frida==<версия CLI>`.

**Не ставить:** `frida-dexdump` (archived 2023, на ART 14 всё равно даёт мусор), `Manticore` (нет поддержки ARM64 в нужном виде), `Triton` (Pin x86-only), `IdaCpp`-плагины под VMP.

---

## Итог (5 строк)

1. Методология клона сильна там, где нам её не хватало: JNI-границы, анти-анализ, патч проверок, сетевой триаж — 14 техник берутся сегодня без переделки.
2. Ключевая находка — `tools-mcp.md:626-762`: харнесс `dlopen` + `JNIEnv` и таблица «симптом → фолбэк» дают вызов native-функции за минуты, но только если в APK есть x86_64 ABI.
3. Античит-раздел бесполезен (EAC/BattlEye/VAC — PC), анти-VM написан под Windows и требует ручного перевода на Android-свойства, `linux-privesc.md` не нужен совсем.
4. Дыры закрываются инструментами, а не текстом: `objection`, `jnitrace`, `unidbg`, `Il2CppDumper`, `APKiD`, `ghidra`+MCP, `rizin`, `tshark` — все проверены как живые; Frida, jadx, apktool, mitmproxy уже стоят.
5. Расхождение версий Frida (17.24.3 CLI / 17.15.3 py) — единственный технический дефект окружения, выровнять до начала работы.
