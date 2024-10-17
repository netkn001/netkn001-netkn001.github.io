---
title: rust 操作 hwnd
tags: ["rust", "windows"]
date: 2024-10-16
---

最近使用 `rust` 做跨平台的客户端开发，在 `windows` 上碰到了一个窗口句柄 `HWND` 相关操作的问题，由于对 `windows` 开发不是太了解，前期比较折腾，也学习到了些许知识，这里小记录下。

在 `windows` 系统，`句柄(handle)` 是非常重要的概念，是一个标识符，简单理解就是万事万物都有一个唯一标识，在 `windows` 是用 4 字节无符号整形来表示。

`窗口句柄(hwnd - handle to window)` 重点是窗口，简单理解就是各种可见的窗口。

启动进程会返回唯一进程 `id`，同时这个进程系统也会分配一个句柄，线程也一样，所以想操作进程相关行为，可以基于进程 `id`，也可以基于句柄(`handle`)。

很简单的一个场景，我通过命令行启动了一个进程，然后关掉这个进程

```rust
extern crate windows_sys;

use std::time::Duration;

use tokio::process::Command;
use windows_sys::Win32::Foundation::HWND;
use windows_sys::Win32::UI::WindowsAndMessaging::{
    EnumWindows, GetWindowThreadProcessId, SendMessageW, WM_CLOSE,
};

fn close_window(hwnd: HWND) {
    unsafe {
        SendMessageW(hwnd, WM_CLOSE, 0, 0);
    }
}

// 这得加 unsafe extern "system"
unsafe extern "system" fn enum_windows_callback(hwnd: HWND, dest_pid: isize) -> i32 {
    let mut pid = 0;
    GetWindowThreadProcessId(hwnd, &mut pid);
    if pid == dest_pid as u32 {
	    // 找到匹配的 id，拿到这个窗口句柄，然后发关窗口操作
        close_window(hwnd);
        return 0;
    }
    1
}

#[tokio::main]
async fn main() {
    let mut child = Command::new("C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe")
        .spawn()
        .unwrap();
    // 这个是进程的 handle
    // let handle = child.raw_handle().unwrap();
    let id = child.id().unwrap();
    tokio::time::sleep(Duration::from_secs(5)).await;
    // 为什么不使用 kill
    // child.kill().await.unwrap();
    unsafe {
		// 遍历所有 windows 窗口对象
        EnumWindows(Some(enum_windows_callback), id as isize);
    }
}
```

重点看注释的部分：
- `unsafe extern "system"`
- 不使用 `child.kill()`

`extern "system"` 是指定使用**系统调用**约定，通常在 `Windows` 上使用, 普通的动态库是使用 `extern "C"`，以 `C` 的 `ABI` 方式。

不使用 `child.kill()` 是因为在这个方法是强杀进程，不能让被杀的程序优雅退出），会造成些数据状态丢失等问题。 `windows` 系统不像 `unix` 系有各种信号机制，所以这个退出只能另用其他方式，`windows` 也提供了对应的关闭方式就是 `WM_CLOSE` 消息。

在 `rust` 调用系统库的一些方法，也是碰到语言特性的问题，`rust` 安全性保障太高了。
看如上的 `EnumWindows` 方法，其源码是
```rust
windows_targets::link!("user32.dll" "system" fn EnumWindows(lpenumfunc : WNDENUMPROC, lparam : super::super::Foundation:: LPARAM) -> super::super::Foundation:: BOOL);

pub type WNDENUMPROC = Option<unsafe extern "system" fn(param0: super::super::Foundation::HWND, param1: super::super::Foundation::LPARAM) -> super::super::Foundation::BOOL>;
```

第一个参数的返回值是 `bool` 型，其作为是否要继续遍历，这是 `windows` 官方库提供的。 所以想通过 `pid` 来获取 `hwnd` 是没办法的，只能使用全局变量，但是全局变量，并且加锁。

```rust
static HWNDS: LazyLock<Arc<Mutex<u32>>> = LazyLock::new(|| Arc::new(Mutex::new(9)));
unsafe extern "system" fn enum_windows_callback(hwnd: HWND, dest_pid: isize) -> i32 {
    let mut pid = 0;
    GetWindowThreadProcessId(hwnd, &mut pid);
    if pid == dest_pid as u32 {
        *HWNDS.lock() = hwnd as u32;
        return 0;
    }
    1
}
```

这在其他语言就可以很容易就获取了，根本不需要这么麻烦。但是全局变量是有害的，而且又不好排查问题，这也是 `rust` 强制限制的原因。

最后：
`system-sys` 是微软的官方库，有太多的 `feature` 了， 对 `windows` 不了解，加什么 `feature` 都很麻烦。

就上面的小功能都折腾很久，终究是对 `windows` 了解太少了。
