# Wine 版网易 UU 远程控制端安装与迁移指南

> 本文记录当前 Linux Mint 22.3 / x86_64 / Cinnamon X11 主机上已经验证的
> **UU Remote controller-only** 安装方式，供其他机器复现。
>
> 目标是：Linux 主机只运行 Windows 版 UU Remote 作为控制端，远程控制
> Mac 或 Windows。本文**不是** Ubuntu 被控端 Bridge 的安装说明。

## 1. 已验证的基线

| 项目 | 本次基线 |
| --- | --- |
| Linux | Linux Mint 22.3，x86_64，Cinnamon，X11 |
| Wine | Wine 11.0，`/opt/wine-stable/bin/wine` |
| UU Remote | `4.33.0.8907`，x86_64 |
| 安装包 SHA-256 | `5e3cfe8cfdc6552c1fc26f1ad2c94df133ca20dc3c45c23155358c32ac9bf53e` |
| WebView2 | `151.0.4129.93` |
| Wine prefix | `$HOME/application/uuyc-controller/wineprefix` |
| 控制器根目录 | `$HOME/application/uuyc-controller` |
| X display | `:0` |
| 运行方式 | `GameViewerService` + `GameViewerHealthd.exe` + `GameViewerServer.exe` + `GameViewer.exe` |
| 远端类型 | Mac 或 Windows |

已验证的能力：

- UU GUI 可以正常打开和登录；
- 远程画面正常；
- 页面点击、键盘操作、鼠标点击等输入正常；
- 远程鼠标最初不可见，后来通过 process-local cursor guard 修复；
- 没有使用 Xvfb、FreeRDP、GNOME Remote Desktop 或本地 Linux Bridge；
- 没有修改通用 prefix `$HOME/.wine`；
- Wine 版本保持 11.0，没有升级到其他版本。

## 2. Controller-only 架构

```text
Linux Mint X11 :0
    |
    +-- 独立 Wine prefix
          |
          +-- GameViewerService.exe   Wine service
          +-- GameViewerHealthd.exe   后台健康监控
          +-- GameViewerServer.exe    UU 本地后台组件
          +-- GameViewer.exe          GUI / WebView2
                  |
                  +-- UU signaling / media
                          |
                          +-- 远程 Mac 或 Windows
```

Controller-only 模式不需要把 Linux 本地桌面提供给远端，因此不要执行当前
仓库的完整 `install.sh` 或 `scripts/uu-remote-bridge`。那些脚本用于另外的
场景：让 Windows UU 控制 Ubuntu 桌面，并包含 RDP、输入 Broker、Xvfb 等组件。

## 3. 来源、脚本和文件清单

### 3.1 参考来源

| 文件/目录 | 用途 | 是否直接运行 |
| --- | --- | --- |
| `source/uuyc-wine/PKGBUILD` | AUR 包的版本、固定下载地址、WebView2 和 `wevtapi.dll` 构建方式 | 否，作为参考 |
| `source/uuyc-wine/uuyc-wine` | AUR 包的 prefix 生命周期、服务启动顺序和校验逻辑 | 否，Arch/pacman 专用 |
| `source/uuyc-wine/uuyc-wevtapi.S` | Wine Event Log shim 的汇编源码 | 构建时使用 |
| `source/uuyc-wine/uuyc-wevtapi.def` | `wevtapi.dll` 导出定义 | 构建时使用 |
| `patches/uu-remote-4.33.0.8907.json` | 已审核的 UU Server 二进制补丁清单 | 可选，不能盲目套用 |
| `src/uu_cursor_guard.c` | 远程鼠标光标兼容 DLL 源码 | 构建时使用 |
| `src/uu_injector.c` | 将兼容 DLL 注入运行中 Server 的工具源码 | 构建时使用 |

`ParticleG/uuyc-wine` 当前固定提交为：

```text
1d67ffa9e8df05006ac8272a4e35c23875dfb450
```

它是 Arch/AUR 包，当前 Linux Mint 没有 `pacman` 或 `makepkg`，所以本次只
移植了其关键思路，没有直接安装 AUR 包。

### 3.2 本次创建或编辑的文件

```text
$HOME/application/uuyc-controller/
├── env.sh                         Wine 环境变量
├── launch.sh                      controller-only 启动器
├── compat/
│   ├── uu-cursor-guard.dll        远程鼠标光标兼容层
│   └── uu-injector.exe             DLL 注入工具
├── downloads/
│   └── uuyc_4.33.0.8907.exe       已校验的官方安装包
├── source/
│   ├── uuyc-wine/                  固定提交的 AUR 参考源码
│   └── wevtapi-build/              Event Log shim 构建产物
├── state/                          安装和运行状态；不要公开日志
└── wineprefix/                     独立 Wine prefix，约 2.6 GB
```

Desktop 文件：

```text
$HOME/.local/share/applications/网易UU远程（Wine）.desktop
```

### 3.3 本次没有使用的文件

以下文件属于 Ubuntu 被控端 Bridge，不应复制到 controller-only 机器作为
必需依赖：

```text
install.sh
scripts/uu-remote-bridge
src/uu_input_bridge.c
src/uu_input_broker.c
src/uu_x11_input.c
FreeRDP / GNOME Remote Desktop / Xvfb 相关脚本
```

`scripts/build-compat.sh` 会构建完整 Bridge 的多个 DLL 和 helper，本次没有
整体执行，只单独编译了光标 guard 和 injector。

## 4. 安装前检查

### 4.1 Wine 版本和路径

```bash
/opt/wine-stable/bin/wine --version
/opt/wine-stable/bin/wineboot --version 2>/dev/null || true
```

预期保持：

```text
wine-11.0
```

确认不要让 `/usr/bin/wine` 指向另一个版本：

```bash
command -v wine
readlink -f "$(command -v wine)"
```

### 4.2 需要的工具

Controller-only 不需要安装 Xvfb、FreeRDP 或 GNOME Remote Desktop，但需要：

- Wine 11.0 及 `wineboot`、`winecfg`、`wineserver`、`winetricks`；
- `curl`、`sha256sum`；
- `7z`、`cabextract`；
- X11 会话、`XAUTHORITY`、`xdotool`、`xinput`、`xrdb`；
- 如果构建兼容层：`gcc`、`binutils`、`x86_64-w64-mingw32-gcc`、
  `x86_64-w64-mingw32-strip`、`x86_64-w64-mingw32-objdump`；
- `update-desktop-database` 和可选的 `desktop-file-validate`。

可以先检查：

```bash
for command in wine wineboot winecfg wineserver winetricks \
    curl 7z cabextract xdotool xinput xrdb \
    x86_64-w64-mingw32-gcc x86_64-w64-mingw32-strip; do
    command -v "$command" || printf 'missing: %s\n' "$command"
done
```

本次 `sudo apt-get install` 因为非交互终端无法读取密码没有执行；检查结果
显示主要依赖已经存在。需要补包时，在真实交互终端中执行，不要为了这个
controller-only 场景安装完整 Bridge 依赖。

## 5. 安装步骤

下面命令中的 `$CTRL` 可以换成其他机器上的目录，但建议继续放在
`$HOME/application` 下。

### 5.1 创建目录

```bash
export CTRL="$HOME/application/uuyc-controller"
mkdir -p "$CTRL"/{downloads,state,source,compat}
chmod 700 "$CTRL" "$CTRL"/{downloads,state,compat}
```

### 5.2 创建环境文件

建议使用动态 `$HOME`，不要把旧机器的 `/home/yujk01` 写死：

```bash
cat >"$CTRL/env.sh" <<'EOF'
#!/usr/bin/env bash

export UURB_CONTROLLER_ROOT="$HOME/application/uuyc-controller"
export WINEPREFIX="$UURB_CONTROLLER_ROOT/wineprefix"
export WINEARCH=win64
export WINEDEBUG=-all
export UURB_CURSOR_SIZE="${UURB_CURSOR_SIZE:-48}"
export DISPLAY="${DISPLAY:-:0}"
export XAUTHORITY="${XAUTHORITY:-$HOME/.Xauthority}"
export WINEDLLOVERRIDES='mscoree,mshtml=;winemenubuilder.exe=d'
export WINE_BIN=/opt/wine-stable/bin/wine
export WINEBOOT_BIN=/opt/wine-stable/bin/wineboot
export WINECFG_BIN=/opt/wine-stable/bin/winecfg
export WINESERVER_BIN=/opt/wine-stable/bin/wineserver
EOF
chmod 700 "$CTRL/env.sh"
source "$CTRL/env.sh"
```

如果希望显式强制使用本地 `wevtapi.dll`，可以把
`WINEDLLOVERRIDES` 改为：

```bash
export WINEDLLOVERRIDES='wevtapi=n,b;mscoree,mshtml=;winemenubuilder.exe=d'
```

### 5.3 初始化独立 prefix

```bash
source "$CTRL/env.sh"

WINEARCH=win64 WINEPREFIX="$WINEPREFIX" \
  WINEDEBUG=-all "$WINEBOOT_BIN" --init

WINEPREFIX="$WINEPREFIX" "$WINECFG_BIN" -v win10
```

确认 prefix 是 64 位并配置为 Windows 10：

```bash
WINEPREFIX="$WINEPREFIX" "$WINE_BIN" cmd /c ver
```

不要将这个 prefix 与 `$HOME/.wine` 混用。

### 5.4 下载并校验固定版本安装包

本次不能直接信任动态下载接口，因为它曾返回了不在审核清单中的
`4.37.0.9232`。本次使用的固定官方地址是：

```text
https://a56.gdl.netease.com/UURemote_Setup_4.33.0.8907_0715193023_gwqd.exe
```

下载并校验：

```bash
installer="$CTRL/downloads/uuyc_4.33.0.8907.exe"
url='https://a56.gdl.netease.com/UURemote_Setup_4.33.0.8907_0715193023_gwqd.exe'
sha256='5e3cfe8cfdc6552c1fc26f1ad2c94df133ca20dc3c45c23155358c32ac9bf53e'

curl -fL --retry 3 "$url" -o "$installer"
printf '%s  %s\n' "$sha256" "$installer" | sha256sum -c -
chmod 700 "$installer"
```

如果下载接口返回其他版本或 SHA-256 不匹配，停止，不要安装。

### 5.5 安装 UU Remote

安装程序是 NSIS 安装包。本次使用静默参数：

```bash
source "$CTRL/env.sh"
installer="$CTRL/downloads/uuyc_4.33.0.8907.exe"

"$WINE_BIN" "$installer" /S /launchapp=0 /autorun=0
```

验证文件和注册表版本：

```bash
app="$WINEPREFIX/drive_c/Program Files/Netease/GameViewer"

test -f "$app/GameViewer.exe"
test -f "$app/bin/GameViewerServer.exe"
test -f "$app/bin/GameViewerHealthd.exe"

"$WINE_BIN" reg query 'HKCU\Software\Netease\GameViewer' /v Version
```

预期版本：

```text
4.33.0.8907
```

### 5.6 安装 UU 自带的 WebView2

不要先随意用 winetricks 安装另一个 WebView2 版本。优先使用 UU 安装目录
中随附的安装程序：

```bash
webview="$WINEPREFIX/drive_c/Program Files/Netease/GameViewer/bin/MicrosoftEdgeWebview2Setup.exe"
"$WINE_BIN" "$webview" /silent /install
```

验证：

```bash
"$WINE_BIN" reg query \
  'HKLM\Software\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}' \
  /v pv
```

本次版本：

```text
151.0.4129.93
```

### 5.7 写入 Wine 兼容配置

```bash
source "$CTRL/env.sh"

"$WINE_BIN" reg add \
  'HKCU\Software\Wine\AppDefaults\msedgewebview2.exe' \
  /v Version /t REG_SZ /d win8 /f

"$WINE_BIN" reg add \
  'HKCU\Software\Wine\X11 Driver' \
  /v UseTakeFocus /t REG_SZ /d N /f
```

这两个设置的用途：

- WebView2 使用 Wine `win8` application override，避免 GUI 启动异常；
- `UseTakeFocus=N` 减少 Wine X11 窗口焦点切换问题。

### 5.8 构建并安装 `wevtapi.dll` shim

UU 会调用 `EvtOpenPublisherMetadata`。Wine 11.0 对该 API 的实现不完整，
会导致 Server 进程退出。参考 AUR 源码构建一个只返回正常 Windows 错误形态
的 shim。

从当前仓库源码构建：

```bash
source "$CTRL/env.sh"
repo="$HOME/workSpace/code/github.com/uu-remote-ubuntu-bridge"
src="$CTRL/source/uuyc-wine"
build="$CTRL/source/wevtapi-build"
mkdir -p "$build"

as --64 "$src/uuyc-wevtapi.S" \
  -o "$build/uuyc-wevtapi-elf.o"

objcopy --remove-section=.note.gnu.property -O pe-x86-64 \
  "$build/uuyc-wevtapi-elf.o" \
  "$build/uuyc-wevtapi-coff.o"

wine_lib_dir=/opt/wine-stable/lib/wine/x86_64-windows
ld -mi386pep --dll --no-insert-timestamp --entry=DllMain \
  --subsystem windows -o "$build/wevtapi.dll" \
  "$build/uuyc-wevtapi-coff.o" "$src/uuyc-wevtapi.def" \
  -L"$wine_lib_dir" -lkernel32

install -m 644 "$build/wevtapi.dll" \
  "$WINEPREFIX/drive_c/Program Files/Netease/GameViewer/bin/wevtapi.dll"
```

本次安装后的 shim SHA-256：

```text
af08d41c6ba97377530d419a7d53a098b6e499727ce70a3145a6b4a22ac8f949
```

验证是否加载：

```bash
server_pid=$(pgrep -o -f 'GameViewerServer\.exe')
grep -i 'wevtapi.dll' "/proc/$server_pid/maps"
```

### 5.9 可选：应用已审核的 Server 输入补丁

清单：

```text
patches/uu-remote-4.33.0.8907.json
```

用途是将 UU 的 `gvinput.sys` 默认路径切换为用户态 `SendInput`。本次
controller-only 安装**没有应用该二进制补丁**，当前 Server 保持原版：

```text
原版 SHA-256:
be1c6c108e6e4d0d5cc15dcd22650dc5fde34c7e7b9f19eee72aba0160ea3494

清单中预期的补丁版 SHA-256:
30cad61560213c7a66244c6f79c9017cc9dfa81996d7faa15a0e8bf330aa0948
```

如果另一台机器确实遇到 `gvinput`、HID 或输入路径问题，先备份并使用清单
工具，不要手工改偏移：

```bash
source "$CTRL/env.sh"
repo="$HOME/workSpace/code/github.com/uu-remote-ubuntu-bridge"
server="$WINEPREFIX/drive_c/Program Files/Netease/GameViewer/bin/GameViewerServer.exe"
manifest="$repo/patches/uu-remote-4.33.0.8907.json"
backup="$server.uu-original"

python3 "$repo/scripts/patch-gameviewer.py" status \
  "$server" --manifest "$manifest"

python3 "$repo/scripts/patch-gameviewer.py" patch \
  "$server" --manifest "$manifest" --backup "$backup"

python3 "$repo/scripts/patch-gameviewer.py" verify \
  "$server" --manifest "$manifest" --expect patched
```

应用或恢复补丁前必须停止当前 prefix 中的 UU 进程。该补丁与 cursor guard
是两个独立问题；不要为了修复光标就盲目修改 Server 二进制。

### 5.10 构建并启用鼠标光标 guard

本次远程鼠标不可见的直接证据是客户端日志中的：

```text
cursor type:-1, hasData=0
isEnterGameModeForNullCursor=1
```

这是 Wine 下 UU Server 获取 cursor shape 失败，不是鼠标点击输入失败。

构建两个 64 位 Windows 兼容文件：

```bash
source "$CTRL/env.sh"
repo="$HOME/workSpace/code/github.com/uu-remote-ubuntu-bridge"
out="$CTRL/compat"
mkdir -p "$out"
chmod 700 "$out"

x86_64-w64-mingw32-gcc -std=c11 -O2 -Wall -Wextra -Werror \
  -Wl,--no-insert-timestamp -shared \
  -o "$out/uu-cursor-guard.dll" \
  "$repo/src/uu_cursor_guard.c" -luser32 -lgdi32

x86_64-w64-mingw32-gcc -std=c11 -O2 -Wall -Wextra -Werror \
  -Wl,--no-insert-timestamp -municode \
  -o "$out/uu-injector.exe" \
  "$repo/src/uu_injector.c"

x86_64-w64-mingw32-strip \
  "$out/uu-cursor-guard.dll" "$out/uu-injector.exe"
chmod 600 "$out/uu-cursor-guard.dll" "$out/uu-injector.exe"
```

启动 `GameViewerServer.exe` 后注入：

```bash
source "$CTRL/env.sh"
export UURB_CURSOR_SIZE=48

dll_windows_path=$("$WINE_BIN" winepath -w \
  "$CTRL/compat/uu-cursor-guard.dll")

"$WINE_BIN" "$CTRL/compat/uu-injector.exe" \
  "$dll_windows_path" GameViewerServer.exe
```

验证：

```bash
cat "$WINEPREFIX/drive_c/users/$USER/AppData/Local/Temp/uu-cursor-guard.log"
```

预期：

```text
UU cursor reader guard active (cursor 48x48)
```

本次 `launch.sh` 已将上述注入做成自动步骤，并且会检查 DLL 是否已经映射，
避免重复注入。若直接关闭并重新连接远程会话，UU 会重新请求 cursor shape。

## 6. 启动顺序和桌面入口

### 6.1 推荐启动顺序

必须按以下顺序：

```text
GameViewerService
    -> GameViewerHealthd.exe
    -> GameViewerServer.exe
    -> cursor guard injection
    -> GameViewer.exe
```

`GameViewerService` 的 `sc start` 在 Wine 下可能返回 `32`，但服务随后实际
进入运行状态。本次以查询结果为准：

```text
STATE : 4  RUNNING
```

不要只根据 `sc start` 的返回码判断失败。

### 6.2 当前 launcher

```text
$HOME/application/uuyc-controller/launch.sh
```

它负责：

- 加载独立 `env.sh`；
- 查询并启动 `GameViewerService`；
- 启动 Healthd 和 Server；
- 自动注入 cursor guard；
- 避免重复启动已有 GUI；
- 最后启动 `GameViewer.exe`。

停止这个 prefix 的 Wine 进程：

```bash
source "$HOME/application/uuyc-controller/env.sh"
timeout 15s "$WINESERVER_BIN" -k || true
timeout 5s "$WINESERVER_BIN" -w || true
```

### 6.3 Desktop 文件

```text
$HOME/.local/share/applications/网易UU远程（Wine）.desktop
```

核心内容：

```ini
[Desktop Entry]
Type=Application
Name=网易UU远程（Wine）
Exec=/绝对路径/application/uuyc-controller/launch.sh
Icon=application-x-executable
Terminal=false
StartupNotify=true
StartupWMClass=gameviewer.exe
Categories=Wine;Network;RemoteAccess;
```

`Exec` 必须是目标机器上的绝对路径；Desktop 文件不会可靠展开 `$HOME`。
安装完成后刷新菜单：

```bash
chmod 644 "$HOME/.local/share/applications/网易UU远程（Wine）.desktop"
update-desktop-database "$HOME/.local/share/applications"
```

Cinnamon 的 `Wine` 是自定义菜单分类，因此 `desktop-file-validate` 可能提示
它不是标准 freedesktop 分类；Cinnamon 的应用菜单仍会把它放到 Wine 分类。

## 7. 验证清单

### 7.1 安装文件

```bash
app="$WINEPREFIX/drive_c/Program Files/Netease/GameViewer"

test -f "$app/GameViewer.exe"
test -f "$app/bin/GameViewerServer.exe"
test -f "$app/bin/GameViewerHealthd.exe"
test -f "$app/bin/wevtapi.dll"
```

### 7.2 进程

```bash
ps -eo pid,ppid,stat,comm,args | grep -E \
  '[G]ameViewer(Service|Healthd|Server|\.exe)|[m]sedgewebview2'
```

### 7.3 服务

```bash
source "$CTRL/env.sh"
"$WINE_BIN" sc query GameViewerService
```

必须看到：

```text
STATE : 4  RUNNING
```

### 7.4 WebView2 和 Wine 配置

```bash
"$WINE_BIN" reg query 'HKCU\Software\Netease\GameViewer' /v Version
"$WINE_BIN" reg query \
  'HKLM\Software\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}' \
  /v pv
"$WINE_BIN" reg query \
  'HKCU\Software\Wine\AppDefaults\msedgewebview2.exe' /v Version
"$WINE_BIN" reg query \
  'HKCU\Software\Wine\X11 Driver' /v UseTakeFocus
```

### 7.5 功能验收

每台新机器至少测试：

1. UU 账号登录；
2. 远端设备列表出现；
3. Mac 连接；
4. Windows 连接；
5. 画面和缩放；
6. 页面点击；
7. 键盘按键和快捷键；
8. 鼠标移动、左键、右键、拖拽、滚轮；
9. 光标可见；
10. 断开后重新连接；
11. 关闭并重新启动 launcher 后仍能连接。

## 8. 日志和故障排查

### 8.1 日志位置

Wrapper 日志：

```text
$CTRL/state/
$CTRL/state/runtime-logs/
```

UU 原生日志：

```text
$WINEPREFIX/drive_c/Program Files/Netease/GameViewer/log/
```

光标 guard 日志：

```text
$WINEPREFIX/drive_c/users/$USER/AppData/Local/Temp/uu-cursor-guard.log
```

### 8.2 GUI 能打开但远程鼠标不可见

查看客户端日志：

```bash
log_dir="$WINEPREFIX/drive_c/Program Files/Netease/GameViewer/log/client/Log"
latest=$(find "$log_dir" -type f -name 'log_*.txt' \
  -printf '%T@ %p\n' | sort -nr | head -1 | cut -d' ' -f2-)
rg 'setCursorShape|cursor type|hasData|NullCursor' "$latest"
```

如果看到：

```text
cursor type:-1, hasData=0
```

重新注入 cursor guard，并断开/重连远程会话。

### 8.3 Server 四分钟左右退出

优先检查：

```bash
rg -i 'wevtapi|EvtOpenPublisherMetadata|unimplemented' \
  "$WINEPREFIX/drive_c/Program Files/Netease/GameViewer/log/server"
```

确认应用目录存在本地 `wevtapi.dll`，并在必要时设置：

```bash
export WINEDLLOVERRIDES='wevtapi=n,b;mscoree,mshtml=;winemenubuilder.exe=d'
```

### 8.4 输入事件不工作但 GUI 正常

查看 Server 日志中的：

```text
virtual switch state:
gvinput
SearchMatchingHwID
SendInput
```

本次机器的 Server 仍是原版，日志中可以看到 Wine 下找不到真实的 `gvinput`
HID 设备，但实际键盘和鼠标事件已经可用。若另一台机器输入完全失败，再
考虑应用第 5.9 节的 approved Server patch；先保留原版备份。

### 8.5 `sc start` 返回 32

重新查询：

```bash
"$WINE_BIN" sc query GameViewerService
```

如果是 `STATE : 4 RUNNING`，则服务已经正常运行，返回码 32 是 Wine 服务
启动过程中的非权威返回值。

## 9. 为其他机器准备哪些文件

### 9.1 建议复制/版本控制的内容

```text
docs/controller-only-wine-installation.md
patches/uu-remote-4.33.0.8907.json
src/uu_cursor_guard.c
src/uu_injector.c
source/uuyc-wine/uuyc-wevtapi.S
source/uuyc-wine/uuyc-wevtapi.def
source/uuyc-wine/PKGBUILD

当前机器的 controller-only 配置：
$HOME/application/uuyc-controller/env.sh
$HOME/application/uuyc-controller/launch.sh
$HOME/.local/share/applications/网易UU远程（Wine）.desktop

版本库中的可复用文档和源码：
docs/controller-only-wine-installation.md
src/uu_cursor_guard.c
src/uu_injector.c
```

另外保留已审核的官方安装包，或在新机器上重新从固定地址下载并校验：

```text
uuyc_4.33.0.8907.exe
SHA-256 = 5e3cfe8cfdc6552c1fc26f1ad2c94df133ca20dc3c45c23155358c32ac9bf53e
```

### 9.2 不建议直接复制的内容

不要把以下内容作为通用安装包直接分发：

```text
整个 wineprefix/
state/ 下的 UU 日志
WebView2 cache
GameViewer 登录数据和 cookies
包含账号、设备 ID、房间 ID 的日志
$HOME/.wine/
```

prefix 中含有用户目录、登录状态、设备信息和 Wine 的绝对路径链接。对另一
台机器，建议使用相同版本安装包重新创建 prefix，而不是直接复制整个 prefix。

如果确实要迁移 prefix，必须：

1. 在原机停止全部 UU/Wine 进程；
2. 只迁移到相同架构和相同用户语义的环境；
3. 检查 `dosdevices/z:` 等绝对路径链接；
4. 更新 `env.sh` 和 Desktop 文件；
5. 将所有日志和登录状态视为敏感数据。

## 10. 升级策略

当前已验证版本是 `4.33.0.8907`，不要因为官方动态接口返回新版本就自动升级。
升级必须同时完成：

1. 获取新安装包；
2. 校验安装包 SHA-256；
3. 记录新版本的 `GameViewerServer.exe` 原始 hash；
4. 重新审核是否需要 Server patch；
5. 重新确认 WebView2 兼容性；
6. 重新验证光标、鼠标、键盘和断线重连；
7. 在 disposable prefix 中测试后再替换正式 prefix。

当前仓库的 `scripts/patch-gameviewer.py` 是 fail-closed 工具：如果版本或
hash 不在 approved manifest 中，应停止，不要手工添加一个新 hash 继续运行。

## 11. 当前机器最终状态摘要

```text
UU Remote:              4.33.0.8907
Wine:                   11.0
WebView2:               151.0.4129.93
Prefix:                 ~/application/uuyc-controller/wineprefix
Service:                GameViewerService = RUNNING
Healthd:                running
Server:                 running
Server binary:          original, not patched
wevtapi shim:           installed and loaded
cursor guard:           installed, injected, 48x48
Desktop entry:          installed under Cinnamon Wine category
```

本次安装的核心原则是：固定 UU 版本、固定 Wine 版本、独立 prefix、只运行
controller-only 组件、先验证原版行为、按日志逐项加入最小兼容层，而不是把
Ubuntu 被控端 Bridge 的全部依赖搬过来。
