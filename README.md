# PBNILib v2.2 — PowerBuilder Native Interface 框架

> PB 12.5 + VS2010 + Win7/10 | C++03 | 四类注册型 + 一类宿主型 | MIT

---

## 一、快速开始

### 1. 环境

| 软件 | 版本要求 |
|------|---------|
| PowerBuilder | 12.5（10.x / 11.x / 12.x 兼容） |
| Visual Studio | 2010 或更高（必须 32 位编译） |
| Windows | Win7 / Win10 |

### 2. 目录结构

```
你的项目/
├── PBNILib/              ← 框架（拷到项目目录下，不动）
│   ├── pbnilib.h         ← 类型 1~4 只需要这个头文件
│   ├── pbnilib.cpp       ← 仅类型 5（PBVM Host）需要，DLL 扩展不用拖
│   ├── README.md
│   └── LICENSE
├── PBNILibDemo/          ← 可编译的 NVO 示例
│   ├── main.cpp
│   ├── MyMath.h
│   ├── MyMath.cpp
│   └── README.md
├── PBNI/                 ← Sybase SDK（PB 安装目录下）
│   ├── include/          ← pbext.h / pbni.h 等头文件
│   └── lib/              ← PBNI.LIB 链接库
├── main.cpp              ← 入口文件（2 行）
├── MyClass.h             ← 你的业务类声明
└── MyClass.cpp           ← 注册宏 + 业务实现
```

### 3. VS2010 项目配置

| 设置项 | 值 |
|--------|-----|
| 项目类型 | Win32 DLL |
| 字符集 | 使用 Unicode 字符集 |
| C/C++ → 附加包含目录 | `..\PBNILib` + `..\PBNI\include` |
| 链接器 → 附加依赖项 | `..\PBNI\lib\PBNI.LIB` |
| 空项目 | ✅ 勾选 |
| 源文件 | 只需拖你自己的 `.cpp` / `.h` + `main.cpp`，**不用拖 `pbnilib.cpp`** |

### 4. main.cpp（永远只有 2 行）

```cpp
#include "pbnilib.h"
PBNI_DLL_MAIN()
```

### 5. 写一个 NVO 类

**MyMath.h：**

```cpp
#include "pbnilib.h"

class MyMath : public PBNIBase
{
public:
    MyMath(IPB_Session *s) : PBNIBase(s) {}
    PBXRESULT Hello(PBCallInfo *ci);
    PBXRESULT Add(PBCallInfo *ci);
    PBNI_NVO_INVOKE_DECL   // MSVC 要求
};
```

**MyMath.cpp：**

```cpp
#include "MyMath.h"

// ① 方法表
PBNI_REGISTER_NVO_BEGIN(MyMath, "n_cpp_math", "nonvisualobject")
    PBNI_FUNC(0, "function string of_hello()")
    PBNI_FUNC(1, "function int of_add(int a, int b)")
PBNI_REGISTER_NVO_END

// ② Invoke 分发
PBNI_REGISTER_NVO_INVOKE(MyMath)
    PBNI_INVOKE_CASE(0, Hello)
    PBNI_INVOKE_CASE(1, Add)
PBNI_INVOKE_END

// ③ 业务实现
PBXRESULT MyMath::Hello(PBCallInfo *ci) {
    ci->returnValue->SetString(_T("Hello from C++!"));
    return PBX_OK;
}

PBXRESULT MyMath::Add(PBCallInfo *ci) {
    pbint a = ci->pArgs->GetAt(0)->GetInt();
    pbint b = ci->pArgs->GetAt(1)->GetInt();
    ci->returnValue->SetInt(a + b);
    return PBX_OK;
}
```

### 6. 编译 + 导入 PB

```
F7 编译 → 生成 DLL
PB 右键 PBL → Import PB Extension → 选 DLL
CREATE n_cpp_math → 开用
```

---

## 二、五类完整模板（四种注册型 + 一种宿主型）

### 类型 1：NVO（非可视对象）

最常见。用于业务逻辑扩展、加密、HTTP、数据库连接池等。

**类必须继承 `PBNIBase`。构造函数必须接受 `(IPB_Session *)` 参数。**

```cpp
// ---- .h ----
#include "pbnilib.h"
class MyNvo : public PBNIBase {
public:
    MyNvo(IPB_Session *s) : PBNIBase(s) {}
    PBXRESULT DoSomething(PBCallInfo *ci);
    PBNI_NVO_INVOKE_DECL
};

// ---- .cpp ----
PBNI_REGISTER_NVO_BEGIN(MyNvo, "n_cpp_xxx", "nonvisualobject")
    PBNI_FUNC(0, "function string of_dosomething(string data)")
PBNI_REGISTER_NVO_END

PBNI_REGISTER_NVO_INVOKE(MyNvo)
    PBNI_INVOKE_CASE(0, DoSomething)
PBNI_INVOKE_END

PBXRESULT MyNvo::DoSomething(PBCallInfo *ci) {
    pbstring in = ci->pArgs->GetAt(0)->GetString();
    LPCTSTR str = d_session->GetString(in);
    // ... 处理 ...
    ci->returnValue->SetString(_T("result"));
    return PBX_OK;
}
```

**PB 端：**

```pb
n_cpp_xxx obj
obj = CREATE n_cpp_xxx
string s = obj.of_dosomething("hello")
DESTROY obj
```

---

### 类型 2：Visual（可视控件）

用于自定义窗口控件——绘图、图表、自定义按钮等。

**推荐继承 `PBNIVisualBase`（框架提供），只需实现 4 个方法。**
**也可直接继承 `IPBX_VisualObject`（需实现 7 个方法）。**

```cpp
// ---- .h ----
#include "pbnilib.h"
class UvoBox : public PBNIVisualBase {
public:
    UvoBox(IPB_Session *s, pbobject obj) : PBNIVisualBase(s, obj) {}
    virtual ~UvoBox();
    PBNI_VISUAL_INVOKE_DECL

    // 必须实现的 4 个（Destroy / GetEventID 由基类提供默认实现）
    LPCTSTR GetWindowClassName();
    HWND CreateControl(DWORD ex, LPCTSTR name, DWORD style,
        int x, int y, int w, int h, HWND parent, HINSTANCE inst);

    // 窗口管理（必须是 static）
    static void RegisterWndClass(HINSTANCE h);
    static void UnregisterWndClass(HINSTANCE h);
    static LRESULT CALLBACK WndProc(HWND, UINT, WPARAM, LPARAM);

    // 业务
    PBXRESULT SetColor(PBCallInfo *ci);
    // Invoke() 由 PBNI_REGISTER_VISUAL_INVOKE 宏自动生成
};

// ---- .cpp ----
// ① 方法表
PBNI_REGISTER_VISUAL_BEGIN(UvoBox, "u_cpp_box")
    PBNI_FUNC(0, "subroutine of_setcolor(int r, int g, int b)")
PBNI_REGISTER_VISUAL_END

// ② Invoke（DllMain 钩子自动生成）
PBNI_REGISTER_VISUAL_INVOKE(UvoBox)
    PBNI_VISUAL_INVOKE_CASE(0, SetColor)
PBNI_VISUAL_INVOKE_END

// ③ + ④ + ⑤ 实现（构造、Visual 接口、业务）
// 详见 PBNILibDemo/UvoBox.cpp
```

**PB 端：** `Insert → Object → User Object → u_cpp_box` 拖到窗口上。

---

### 类型 3：全局函数

PB 脚本里直接调 `f_xxx()`，无需 CREATE。

**注意：必须手动写 `return PBX_OK;`。**

```cpp
#include "pbnilib.h"

PBNI_REGISTER_GLOBAL_BEGIN(f_version, "function string f_version()")
{
    ci->returnValue->SetString(_T("v1.0"));
    return PBX_OK;   // ← 必须写
}
PBNI_REGISTER_GLOBAL_END(f_version)
```

**PB 端：**

```pb
string s = f_version()   // 直接调，不需要 CREATE
```

---

### 类型 4：Marshaler（远程代理）

用于跨进程通信——PB 调代理对象，C++ 把请求转发到远程服务。

**类必须继承 `PBNIMarshalerBase`（基类已存 d_session / d_obj）。PB 端通过 `NewProxyObject + SetMarshaler` 绑定。**

```cpp
// ---- .h ----
#include "pbnilib.h"
class MyProxy : public PBNIMarshalerBase {
public:
    MyProxy(IPB_Session *s, pbproxyObject o) : PBNIMarshalerBase(s, o) {}
    PBXRESULT RemoteEcho(IPB_Session *s, pbproxyObject o, PBCallInfo *ci);
    PBNI_MARSHALER_INVOKE_DECL
    // d_session、d_obj 从基类继承，无需自己声明
};

// ---- .cpp ----
PBNI_REGISTER_MARSHALER_BEGIN(MyProxy, "n_cpp_proxy")
    PBNI_FUNC(0, "function string of_remoteecho()")
PBNI_REGISTER_MARSHALER_END

PBNI_REGISTER_MARSHALER_INVOKE(MyProxy)
    PBNI_MARSHALER_CASE(0, RemoteEcho)
PBNI_MARSHALER_INVOKE_END

PBXRESULT MyProxy::RemoteEcho(IPB_Session *s, pbproxyObject o, PBCallInfo *ci) {
    ci->returnValue->SetString(_T("Remote OK"));
    return PBX_OK;
}
```

---

### 类型 5：PBVM Host（C++ EXE 嵌入 PB）

在自己的 C++ EXE 中加载 PB 虚拟机，运行 PB 应用。

```cpp
#include "pbnilib.h"

int main() {
    PBNIHost host;
    IPB_Session *s = host.Start(_T("C:\\MyApp\\myapp.pbt"));
    if (!s) return 1;

    // 和 PB 交互——调全局函数、创建对象、读写变量
    pbgroup g = s->FindGroup(_T("n_cpp_xxx"), pbgroup_userobject);
    pbclass c = s->FindClass(g, _T("n_cpp_xxx"));
    pbobject obj = s->NewObject(c);

    // ... 使用完毕 ...

    host.Stop();
    return 0;
}
```

---

## 三、方法参数读写速查

### 从 PB 读取参数（ci→pArgs）

| PB 类型 | C++ 代码 |
|---------|---------|
| `integer` / `int` | `ci->pArgs->GetAt(0)->GetInt()` |
| `long` | `ci->pArgs->GetAt(0)->GetLong()` |
| `string` | `ci->pArgs->GetAt(0)->GetString()` → 返回 `pbstring`，需用 `d_session->GetString(pbstr)` 转 `LPCTSTR` |
| `boolean` | `ci->pArgs->GetAt(0)->GetBool()` |
| `double` | `ci->pArgs->GetAt(0)->GetDouble()` |
| `real` | `ci->pArgs->GetAt(0)->GetReal()` |
| `blob` | `ci->pArgs->GetAt(0)->GetBlob()` |
| `date` / `time` / `datetime` | `->GetDate()` / `->GetTime()` / `->GetDateTime()` |

### 返回数据给 PB（ci→returnValue）

| PB 返回类型 | C++ 代码 |
|------------|---------|
| `string` | `ci->returnValue->SetString(_T("结果"))` |
| `int` / `integer` | `ci->returnValue->SetInt(42)` |
| `long` | `ci->returnValue->SetLong(123L)` |
| `boolean` | `ci->returnValue->SetBool(TRUE)` |
| `double` | `ci->returnValue->SetDouble(3.14)` |
| `subroutine`（无返回值） | 不调 Set 方法，只 `return PBX_OK;` |

---

## 四、常用操作

### 在 C++ 里调 PB 的对象方法

```cpp
PBObjectFunctionInvoker invoker(d_session, pbObj, _T("of_someMethod"));
invoker.GetArg(0)->SetInt(42);
invoker.Invoke();
pbint result = invoker.GetReturnValue()->GetInt();
```

### 读写 PB 全局变量

```cpp
pbfieldID fid = d_session->GetGlobalVarID(_T("gs_myvar"));
pbboolean isNull = FALSE;
pbstring val = d_session->GetStringGlobalVar(fid, isNull);
```

### 处理 PB 异常

```cpp
if (d_session->HasExceptionThrown()) {
    pbobject ex = d_session->GetException();
    d_session->ClearException();
    // 处理异常...
}
```

---

## 五、一个 DLL 放多个类

`main.cpp` 不用改。每加一个类，加一对 `.h` / `.cpp`，各自的注册宏独立互不干扰：

```
MyMath.h / MyMath.cpp        → n_cpp_math
MyCrypto.h / MyCrypto.cpp    → n_cpp_crypto
MyHTTP.h / MyHTTP.cpp        → n_cpp_http
UvoBox.h / UvoBox.cpp        → u_cpp_box
MyGlobal.cpp                 → f_version, f_multiply
```

各自注册到不同的 vector 表，编译后自动聚合到 `PBX_GetDescription`。

---

## 六、内存模型

| 资源 | 分配 | 释放 | 时机 |
|------|------|------|------|
| C++ 对象 | `new ClsName(s)` | `delete this` | PB DESTROY 时 |
| 注册表 | 静态 vector | CRT 析构 | 进程退出 |
| 描述缓存 | static tstring | CRT 析构 | 进程退出 |
| `d_session` | PB 创建 | PB 管理 | 无需用户释放 |
| 工厂函数 | 静态函数 | — | — |

---

## 七、编译错误速查

| 错误 | 原因 | 解决 |
|------|------|------|
| `duplicate case value` | 两个方法的 `PBNI_FUNC` 用了相同 id | 改成不同的数字 |
| `methodName is not a member` | `PBNI_INVOKE_CASE` 写的方法名在 `.h` 里没声明 | 检查拼写 |
| `unexpected end of file` | 漏了 `PBNI_INVOKE_END` | 补上 |
| `PB import syntax error` | GetDescription 描述字符串格式不对 | 检查 `PBNI_FUNC` 里的 PB 声明串拼写 |
| 链接错误 `unresolved external _DllMain@12` | 漏了 `PBNI_DLL_MAIN()` 或 `PBNI.LIB` 没配 | 检查 main.cpp 和链接器设置 |

---

## 八、文件夹清单

| 路径 | 说明 |
|------|------|
| `PBNILib/pbnilib.h` | 框架主头文件，全部胶水宏 |
| `PBNILib/pbnilib.cpp` | PBNIHost 实现 |
| `PBNILib/README.md` | 本文件 |
| `PBNILib/LICENSE` | MIT 许可证 |
| `PBNILibDemo/` | 完整可编译的 NVO 示例（含 build 说明） |

