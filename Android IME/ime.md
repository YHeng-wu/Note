# Android输入法架构

>IMMS：输入法管理服务 InputMethodManagerService

>IMM：输入法管理 InputMethodManager

>IMS：输入法服务 InputMethodService


# 包的管理
android.view.inputmethod ：提供给IME应用

# 类的涉及
 >@deprecated KeyboradView: UI widget for keyboard,


## InputMethodManager


在 Android 输入法框架（IMF，Input Method Framework）中，`InputMethodManager` 是 **核心系统服务**，负责协调应用与输入法（IME）之间的交互。它充当了应用与当前活动的 IME 之间的桥梁，用于触发键盘的显示/隐藏、切换输入法、管理输入焦点等操作。

---

### **InputMethodManager 的核心作用**
1. **显示/隐藏软键盘**  
   - 当用户点击 `EditText` 等可输入控件时，通过 `InputMethodManager` 请求显示键盘。
   - 当用户按下返回键或应用主动隐藏键盘时，通过它关闭键盘。

2. **切换输入法**  
   - 允许用户或应用切换不同的 IME（如从系统键盘切换到第三方输入法）。

3. **管理输入焦点**  
   - 跟踪当前获得焦点的视图（View），确保输入事件正确路由到目标控件。

4. **与 IME 通信**  
   - 提供方法让应用向 IME 发送额外数据（如 `sendAppPrivateCommand`）。

---

### **代码中的典型使用场景**

#### **1. 显示软键盘**
当用户点击 `EditText` 时，通常会自动弹出键盘。但某些场景需要手动触发（例如跳转到新页面后自动聚焦并弹出键盘）：

```java
public static void showSoftKeyboard(Context context, View view) {
    InputMethodManager imm = (InputMethodManager) context.getSystemService(Context.INPUT_METHOD_SERVICE);
    if (imm != null) {
        imm.showSoftInput(view, InputMethodManager.SHOW_IMPLICIT);
    }
}

// 在 Activity 中使用：
EditText editText = findViewById(R.id.editText);
editText.requestFocus(); // 确保视图获得焦点
showSoftKeyboard(this, editText);
```

#### **2. 隐藏软键盘**
当用户完成输入或需要主动隐藏键盘时：

```java
public static void hideSoftKeyboard(Context context, View view) {
    InputMethodManager imm = (InputMethodManager) context.getSystemService(Context.INPUT_METHOD_SERVICE);
    if (imm != null) {
        imm.hideSoftInputFromWindow(view.getWindowToken(), 0);
    }
}

// 在 Activity 中调用：
View currentFocus = getCurrentFocus();
if (currentFocus != null) {
    hideSoftKeyboard(this, currentFocus);
}
```

#### **3. 切换输入法**
允许用户选择已安装的输入法：

```java
// 跳转到系统输入法选择界面
public static void showInputMethodPicker(Context context) {
    InputMethodManager imm = (InputMethodManager) context.getSystemService(Context.INPUT_METHOD_SERVICE);
    if (imm != null) {
        imm.showInputMethodPicker();
    }
}
```

#### **4. 检查键盘是否已显示**
判断当前键盘的可见状态（仅限 API 30+）：

```java
public static boolean isKeyboardVisible(Context context) {
    InputMethodManager imm = (InputMethodManager) context.getSystemService(Context.INPUT_METHOD_SERVICE);
    if (imm != null && Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        return imm.isActive() && imm.isAcceptingText();
    }
    // 低版本需通过其他方式判断（如监听布局变化）
    return false;
}
```

---

### **关键方法与参数说明**
| 方法 | 说明 |
|------|------|
| `showSoftInput(View view, int flags)` | 显示键盘。`flags` 可选 `SHOW_IMPLICIT`（隐式请求）或 `SHOW_FORCED`（强制显示）。 |
| `hideSoftInputFromWindow(IBinder token, int flags)` | 隐藏键盘。`token` 通常来自 `View.getWindowToken()`。 |
| `restartInput(View view)` | 强制重新初始化输入连接（如输入类型变化时）。 |
| `switchToNextInputMethod(IBinder token, boolean onlyCurrentIme)` | 切换到下一个输入法。 |

---

### **注意事项**
1. **生命周期与焦点管理**  
   - 键盘操作需在视图获得焦点后进行（例如在 `onResume` 或 `View.post()` 中延迟调用 `showSoftInput`）。
   - 避免在 `onCreate` 中直接调用 `showSoftInput`，此时视图可能尚未完成布局。

2. **输入法切换的权限**  
   - 切换输入法需要用户授权，直接调用 `switchToNextInputMethod` 可能需要 `BIND_INPUT_METHOD` 权限（系统级应用才可获取）。

3. **兼容性问题**  
   - Android 11+ 对 `InputMethodManager` 的部分方法进行了限制（如后台应用无法直接控制键盘）。

---

### **与 IME 生命周期的关联**
- 当 `InputMethodManager` 触发键盘显示时，IME 的 `onStartInput()` 和 `onStartInputView()` 会被调用。
- 当键盘隐藏时，IME 的 `onFinishInputView()` 和 `onFinishInput()` 会被触发。

---

### **总结**
`InputMethodManager` 是 Android 输入法框架的“指挥官”，负责调度键盘的显示、隐藏、切换，并协调应用与 IME 的交互。在代码中需注意：  
- 确保视图焦点正确后再操作键盘。
- 处理不同 Android 版本的兼容性问题。
- 遵循系统权限和安全策略。

在 Android 输入法框架（IMF）中，`InputMethodManagerService`（IMMS）是 **系统级服务**，负责全局管理输入法的生命周期、绑定输入法服务、处理跨进程通信，并协调应用与输入法之间的交互。它是 `InputMethodManager` 的后端实现，运行在 `system_server` 进程中，是 IMF 的核心中枢。

---

### **InputMethodManagerService 的核心作用**
1. **管理输入法服务绑定**  
   - 负责启动/绑定用户选择的输入法服务（如 Gboard、系统默认键盘等）。
   - 处理输入法服务的连接（Binder）和断开。

2. **输入法切换逻辑**  
   - 处理用户或系统触发的输入法切换请求（如切换语言或第三方输入法）。

3. **会话管理**  
   - 维护每个客户端（应用窗口）的输入法会话（`InputMethodSession`）。
   - 协调多窗口场景下的输入焦点。

4. **权限与安全性**  
   - 验证调用者权限（如 `BIND_INPUT_METHOD`），防止未授权的输入法操作。

5. **多用户支持**  
   - 隔离不同用户的输入法配置（如多用户设备中的独立设置）。

---

### **源码解析（基于 Android 13）**
IMMS 的源码位于：  
`frameworks/base/services/core/java/com/android/server/inputmethod/InputMethodManagerService.java`

#### **1. 输入法绑定流程**
当用户选择输入法时，IMMS 通过 `bindCurrentInputMethodService()` 绑定目标 IME 的 `InputMethodService`：

```java
// 绑定输入法服务的核心方法
private boolean bindCurrentInputMethodService(
        Intent service, ServiceConnection conn, int flags) {
    // 通过 Context.bindServiceAsUser() 绑定服务
    return mContext.bindServiceAsUser(service, conn, flags, UserHandle.of(mSettings.getCurrentUserId()));
}
```

#### **2. 输入法切换逻辑**
切换输入法的入口方法 `setInputMethodLocked()`，处理输入法 ID 的更新和重启会话：

```java
// 切换输入法的核心逻辑
void setInputMethodLocked(String id, int subtypeId) {
    // 验证输入法是否存在
    final InputMethodInfo imi = mMethodMap.get(id);
    if (imi == null) {
        Slog.w(TAG, "setInputMethod: unknown id: " + id);
        return;
    }
    // 更新系统设置中的输入法 ID
    mSettings.setSelectedInputMethod(id);
    // 重启当前输入法会话
    startInputUncheckedLocked(/* ... */);
}
```

#### **3. 处理输入焦点变化**
当应用窗口焦点变化时，通过 `startInputOrWindowGainedFocus()` 更新输入法会话：

```java
// 窗口获得焦点时的处理
void startInputOrWindowGainedFocus(/* ... */) {
    // 检查当前焦点窗口
    final WindowState window = mWindowManagerInternal.getFocusedWindow();
    // 启动或更新输入会话
    startInputLocked(/* ... */);
}
```

#### **4. 会话管理**
每个客户端（如 Activity）的输入会话由 `ClientState` 表示，IMMS 通过 `mClients` 维护所有客户端状态：

```java
// 客户端状态管理
final ArrayMap<IBinder, ClientState> mClients = new ArrayMap<>();

// 客户端会话的创建
void onClientCreated(IBinder client) {
    ClientState state = new ClientState(client);
    mClients.put(client, state);
}
```

#### **5. 输入法列表管理**
加载所有已安装输入法（`InputMethodInfo`）并缓存到 `mMethodList`：

```java
// 更新输入法列表
void updateInputMethodsFromSettingsLocked(boolean enabledMayChange) {
    // 从 PackageManager 查询所有输入法
    List<InputMethodInfo> imis = mSettings.getEnabledInputMethodListLocked();
    mMethodList.clear();
    mMethodList.addAll(imis);
    // 更新到内存缓存
    buildInputMethodListLocked(/* ... */);
}
```

---

### **关键交互流程**
IMMS 与其他系统服务的协作：

1. **与 `WindowManagerService`**  
   - 通过 `WindowManagerInternal` 获取当前焦点窗口信息。
   - 监听窗口焦点变化事件，触发输入法会话的更新。

2. **与 `ActivityManagerService`**  
   - 处理跨进程 Binder 调用，验证客户端进程的权限和身份。

3. **与 `PackageManagerService`**  
   - 监控输入法 APK 的安装/卸载，动态更新输入法列表。

---

### **权限验证示例**
在关键操作前检查调用者权限（如切换输入法）：

```java
// 检查 BIND_INPUT_METHOD 权限
final int uid = Binder.getCallingUid();
if (mContext.checkCallingPermission(Manifest.permission.BIND_INPUT_METHOD)
        != PackageManager.PERMISSION_GRANTED) {
    throw new SecurityException("UID " + uid + " lacks BIND_INPUT_METHOD");
}
```

---

### **调试与日志**
IMMS 内部通过 `dumpsys input_method` 输出状态信息：

```java
// 在 dump 方法中输出调试信息
protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
    pw.println("Current Input Method Service: " + mCurMethodId);
    pw.println("Enabled Input Methods:");
    for (InputMethodInfo imi : mMethodList) {
        pw.println("  " + imi.getId());
    }
}
```

通过 ADB 查看 IMMS 状态：
```bash
adb shell dumpsys input_method
```

---

### **总结**
`InputMethodManagerService` 是 Android 输入法框架的底层核心，负责：
- 输入法服务的绑定与生命周期管理。
- 处理输入法切换、焦点变化和多窗口场景。
- 维护系统级输入法配置和权限控制。

其源码实现集中在 `InputMethodManagerService.java`，通过 Binder 与客户端（如 `InputMethodManager`）通信，并与 `WindowManagerService` 等协作完成复杂交互。


在 Android 输入法框架（IMF）中，`InputMethodService` 是 **输入法（IME）的核心实现基类**。开发者通过继承 `InputMethodService` 来创建自定义输入法（如键盘、手写输入等）。它直接负责与用户交互（如显示输入界面）、处理输入事件（如按键、手势）以及与客户端应用（如 `EditText`）的通信。

---

### **InputMethodService 的核心作用**
1. **管理输入法界面（UI）**  
   - 创建并控制输入视图（如软键盘、候选词栏）。
   - 处理输入视图的显示与隐藏逻辑。

2. **处理输入事件**  
   - 将用户操作（如点击键盘按键）转换为文本输入事件，传递给目标应用。

3. **与客户端应用通信**  
   - 通过 `InputConnection` 接口与当前焦点的输入控件（如 `EditText`）交互。

4. **生命周期管理**  
   - 响应系统对输入法的启动、停止、销毁等操作。

---

### **源码与关键方法解析**
`InputMethodService` 的源码位于：  
`frameworks/base/core/java/android/inputmethodservice/InputMethodService.java`

#### **1. 输入视图的创建与生命周期**
- **`onCreateInputView()`**  
  创建输入法的主界面（如键盘布局）：

  ```java
  @Override
  public View onCreateInputView() {
      // 返回自定义的键盘视图（如从 XML 布局加载）
      return getLayoutInflater().inflate(R.layout.keyboard_view, null);
  }
  ```

- **`onStartInputView()`**  
  当输入视图需要显示时调用（如用户点击输入框）：

  ```java
  @Override
  public void onStartInputView(EditorInfo info, boolean restarting) {
      super.onStartInputView(info, restarting);
      // 根据输入类型（如数字、文本）调整键盘布局
      updateKeyboardType(info.inputType);
  }
  ```

- **`onFinishInputView()`**  
  当输入视图需要隐藏时调用（如用户离开输入框）：

  ```java
  @Override
  public void onFinishInputView(boolean finishingInput) {
      super.onFinishInputView(finishingInput);
      // 清理临时状态
  }
  ```

---

#### **2. 处理输入事件**
通过 `InputConnection` 向应用发送文本或命令：

```java
// 示例：发送字符到 EditText
private void handleKeyPress(char c) {
    InputConnection ic = getCurrentInputConnection();
    if (ic != null) {
        ic.commitText(String.valueOf(c), 1); // 提交文本
    }
}

// 示例：删除前一个字符
private void handleDelete() {
    InputConnection ic = getCurrentInputConnection();
    if (ic != null) {
        ic.deleteSurroundingText(1, 0); // 删除光标前的一个字符
    }
}
```

---

#### **3. 与客户端应用交互**
通过 `InputConnection` 获取当前输入上下文信息：

```java
// 获取光标前的文本（用于预测输入）
public void getTextBeforeCursor() {
    InputConnection ic = getCurrentInputConnection();
    if (ic != null) {
        CharSequence text = ic.getTextBeforeCursor(10, 0);
        // 处理文本（如显示候选词）
    }
}
```

---

#### **4. 生命周期回调**
- **`onCreate()`**  
  输入法服务首次创建时调用：

  ```java
  @Override
  public void onCreate() {
      super.onCreate();
      // 初始化资源（如主题、数据库连接）
  }
  ```

- **`onDestroy()`**  
  输入法服务销毁时调用：

  ```java
  @Override
  public void onDestroy() {
      super.onDestroy();
      // 释放资源（如关闭数据库、断开网络）
  }
  ```

---

### **关键组件与交互流程**
#### **1. `InputMethodService` 与 `InputMethodManagerService` 的关系**
- **`InputMethodManagerService`（IMMS）** 负责绑定 `InputMethodService` 并管理其生命周期。
- **`InputMethodService`** 通过 Binder 接口（如 `IInputMethodWrapper`）与 IMMS 通信。

#### **2. 输入事件传递流程**
1. 用户点击键盘按键 → `InputMethodService` 捕获事件。
2. 通过 `InputConnection` 调用目标应用的方法（如 `commitText`）。
3. 目标应用（如 `EditText`）更新显示内容。

---

### **代码示例：简单键盘实现**
```java
public class SimpleKeyboard extends InputMethodService 
        implements View.OnClickListener {

    private KeyboardView mKeyboardView;

    @Override
    public View onCreateInputView() {
        mKeyboardView = (KeyboardView) getLayoutInflater().inflate(R.layout.keyboard, null);
        Keyboard keyboard = new Keyboard(this, R.xml.qwerty);
        mKeyboardView.setKeyboard(keyboard);
        mKeyboardView.setOnKeyboardActionListener(new KeyboardActionListener());
        return mKeyboardView;
    }

    private class KeyboardActionListener implements KeyboardView.OnKeyboardActionListener {
        @Override
        public void onKey(int primaryCode, int[] keyCodes) {
            InputConnection ic = getCurrentInputConnection();
            if (ic != null) {
                switch (primaryCode) {
                    case Keyboard.KEYCODE_DELETE:
                        ic.deleteSurroundingText(1, 0);
                        break;
                    default:
                        char code = (char) primaryCode;
                        ic.commitText(String.valueOf(code), 1);
                }
            }
        }
        // 其他方法省略...
    }
}
```

---

### **Android 版本适配注意事项**
1. **暗黑主题支持（Android 10+）**  
   在 `onCreate()` 中动态加载主题：

   ```java
   @Override
   public void onCreate() {
       super.onCreate();
       // 根据系统主题设置输入法主题
       if ((getResources().getConfiguration().uiMode & UI_MODE_NIGHT_MASK) 
               == UI_MODE_NIGHT_YES) {
           setTheme(R.style.DarkTheme);
       } else {
           setTheme(R.style.LightTheme);
       }
   }
   ```

2. **多窗口模式**  
   在 `onStartInputView()` 中检查窗口尺寸，调整键盘布局：

   ```java
   @Override
   public void onStartInputView(EditorInfo info, boolean restarting) {
       super.onStartInputView(info, restarting);
       // 根据窗口高度调整键盘大小
       View root = getWindow().getWindow().getDecorView();
       int height = root.getHeight();
       mKeyboardView.setKeyboardHeight(height / 2);
   }
   ```

---

### **总结**
`InputMethodService` 是 Android 输入法框架中 **实现输入法的核心类**，主要职责包括：
- 管理输入法界面（如键盘、候选栏）。
- 处理用户输入事件并通过 `InputConnection` 与客户端应用通信。
- 响应系统生命周期事件（创建、销毁、显示/隐藏）。

开发者通过扩展此类，可以创建符合 Android 标准的输入法，实现文本输入、手势识别、预测候选词等功能。其源码逻辑集中在与输入事件处理和 UI 生命周期管理上，是 IMF 中直接面向用户交互的关键组件。


以下是 Android 输入法框架（IMF）的核心类及其关系的 **文字化类图描述**，基于 UML 类图简化表示。类图展示了 IMF 中关键组件之间的交互和依赖关系，适用于理解框架设计。

---

### **1. 核心类与接口**
#### **`InputMethodManager`**
- **作用**：客户端（如 Activity）与 IMF 交互的入口。
- **依赖关系**：
  - 通过 Binder 调用 `IInputMethodManager` 接口（代理 `InputMethodManagerService`）。
  - 持有 `Context` 和 `View`（用于焦点管理）。

#### **`InputMethodManagerService` (IMMS)**
- **作用**：系统服务，全局管理输入法生命周期和会话。
- **继承关系**：`IInputMethodManager.Stub`（Binder 服务端）。
- **关联关系**：
  - 维护 `InputMethodInfo` 列表（已安装输入法信息）。
  - 管理 `ClientState`（每个客户端窗口的会话状态）。
  - 绑定 `InputMethodService` 实例。

#### **`InputMethodService`**
- **作用**：输入法实现的基类（如键盘、手写输入）。
- **继承关系**：`Service` → `AbstractInputMethodService`。
- **关键组件**：
  - `InputMethodImpl`：内部类，实现 `IInputMethod` 接口，与 IMMS 通信。
  - `InputConnection`：与应用输入控件（如 `EditText`）通信的接口。
- **依赖关系**：
  - 通过 `InputConnection` 与客户端应用交互。
  - 使用 `KeyboardView` 或自定义视图显示输入界面。

#### **`InputConnection`**
- **作用**：输入法与客户端应用（如 `EditText`）的通信接口。
- **实现类**：`BaseInputConnection`（默认实现）。
- **关键方法**：
  - `commitText()`：提交文本到应用。
  - `deleteSurroundingText()`：删除文本。

#### **`EditorInfo`**
- **作用**：描述输入字段的属性（如 `inputType`、包名）。
- **关联关系**：
  - 由客户端应用通过 `onStartInput()` 传递给输入法。

#### **`InputMethodInfo`**
- **作用**：描述已安装输入法的元数据（如包名、服务类名）。
- **关联关系**：
  - 由 `InputMethodManagerService` 维护列表。

---

### **2. 类图关系总结**
```plaintext
+------------------------+       +---------------------------+
|   InputMethodManager   |<>-----|       Context             |
+------------------------+       +---------------------------+
| + showSoftInput()      |       | (提供系统服务访问)         |
| + hideSoftInput()      |       +---------------------------+
| + switchToNextInputMethod() |
+------------------------+
          |  Binder 调用
          v
+---------------------------+       +-----------------------+
| IInputMethodManager.Stub  |<|-----| InputMethodManagerService |
+---------------------------+       +-----------------------+
| (Binder接口定义)           |       | - mMethodList: List<InputMethodInfo> |
|                           |       | - mClients: ArrayMap<IBinder, ClientState> |
+---------------------------+       +-----------------------+
                                          | 管理
                                          v
+---------------------------+       +-----------------------+
|   InputMethodService      |<>-----| InputMethodImpl       |
+---------------------------+       +-----------------------+
| - mInputConnection        |       | (实现IInputMethod接口) |
| - onCreateInputView()     |       +-----------------------+
| - onStartInputView()      |                   |
| - handleKeyPress()        |                   | 绑定
+---------------------------+                   v
          |                              +-----------------------+
          | 使用                          | IInputMethodSession   |
          v                              +-----------------------+
+---------------------------+           | (会话操作接口)         |
|   InputConnection         |           +-----------------------+
+---------------------------+                   |
| + commitText()            |                   | 实现
| + deleteSurroundingText() |                   v
+---------------------------+       +-----------------------+
          ʌ                          | ClientState          |
          |                          +-----------------------+
          | 实现                      | (客户端会话状态管理)   |
+---------------------------+       +-----------------------+
|   BaseInputConnection     |
+---------------------------+
```

---

### **3. 关键交互流程**
1. **用户点击输入框**：
   - `InputMethodManager` 调用 `IMMS` 的 `startInput()`。
   - `IMMS` 绑定当前输入法的 `InputMethodService`。
   - `InputMethodService` 的 `onStartInputView()` 被调用，显示键盘。

2. **用户输入字符**：
   - `InputMethodService` 通过 `InputConnection.commitText()` 将文本发送到客户端应用（如 `EditText`）。

3. **切换输入法**：
   - `IMMS` 调用 `setInputMethodLocked()`，更新当前输入法并重启会话。

---

### **4. 补充说明**
- **多窗口支持**：`IMMS` 通过 `WindowManagerService` 跟踪焦点窗口，每个窗口对应独立的 `ClientState`。
- **权限控制**：`IMMS` 在关键操作前检查 `BIND_INPUT_METHOD` 等权限。
- **输入法生命周期**：`InputMethodService` 的生命周期由 `IMMS` 通过 Binder 绑定/解绑控制。

---

### **5. 实际代码中的体现**
- **`InputMethodManager` 获取 IMMS 代理**：
  ```java
  // 在 InputMethodManager 中：
  public static InputMethodManager getInstance() {
      return IInputMethodManagerSingleton.get();
  }
  private static final Singleton<InputMethodManager> IInputMethodManagerSingleton =
      new Singleton<InputMethodManager>() {
          protected InputMethodManager create() {
              final IBinder b = ServiceManager.getService(Context.INPUT_METHOD_SERVICE);
              final IInputMethodManager service = IInputMethodManager.Stub.asInterface(b);
              return new InputMethodManager(service, Looper.getMainLooper());
          }
      };
  ```

- **`IMMS` 管理输入法列表**：
  ```java
  // 在 InputMethodManagerService 中：
  void buildInputMethodListLocked() {
      mMethodList.clear();
      List<InputMethodInfo> imis = mSettings.getEnabledInputMethodListLocked();
      mMethodList.addAll(imis);
  }
  ```

---

### **总结**
Android IMF 框架的类图展示了以下核心关系：
1. **客户端入口**：`InputMethodManager` 是应用与 IMF 交互的入口。
2. **系统中枢**：`InputMethodManagerService` 管理输入法生命周期、会话和权限。
3. **输入法实现**：`InputMethodService` 是开发者扩展的基类，依赖 `InputConnection` 与客户端通信。
4. **数据传递**：`EditorInfo` 和 `InputConnection` 是输入法与应用之间的数据载体。

通过理解这些类的关系，可以更清晰地设计自定义输入法或调试输入法相关问题。


# 1.IMF 框架


上述三个模块分别对应不同的进程，客户端 IMM 属于 App 应用进程，输入法 IMS 属于输入法自己的进程，输入法管理服务 IMMS 属于系统进程，三者之间需要通过 IPC 机制来进行通信， Androd中最主要的 IPC 通信方式都就是 Binder 机制。


其中客户端 IMM 与 输入法 IMS 之间的交互已经通过 EditText 及其父类 TextView 实现，如果是自定义的编辑框，需要自行实现与输入法的交互。

# 2.输入法拉起流程
当点击一个输入框 EditText 将获得焦点，系统默认的输入法将会被拉起，EditText 继承自 TextView，关键实现在 TextView 中，拉起输入法的关键流程如下：


可知，输入法 IMS 的拉起可由按键和触摸事件触发，经由 IMM、IMMS 通过一系列 IPC 调用拉起系统默认输入法 IMS，上述流程中省略了一些操作，实际流程要更复杂，重点关注如下返回值：

```java
// InputBindResult.java
// 请求输入法成功
ResultCode.SUCCESS_WITH_IME_SESSION,
// 输入法已经启动，等待IME创建session
ResultCode.SUCCESS_WAITING_IME_SESSION,
// 输入法还未启动，等待输入法绑定
ResultCode.SUCCESS_WAITING_IME_BINDING,
```
当没有绑定输入法和 session 没创建的时候都在等待输入法绑定和 session 创建，当完成这两步之后客户端会再次请求输入法，为便于理解上图中没有体现这个过程，拉起 IMS 的关键代码如下：
```java
// InputMethodManagerService.java
InputBindResult startInputInnerLocked() {
    // ...
    // 绑定输入法
    // InputMethod.SERVICE_INTERFACE(android.view.InputMethod)
    mCurIntent = new Intent(InputMethod.SERVICE_INTERFACE);
    mCurIntent.setComponent(info.getComponent());
    mCurIntent.putExtra(Intent.EXTRA_CLIENT_LABEL,
            com.android.internal.R.string.input_method_binding_label);
    mCurIntent.putExtra(Intent.EXTRA_CLIENT_INTENT, PendingIntent.getActivity(
            mContext, 0, new Intent(Settings.ACTION_INPUT_METHOD_SETTINGS), 0));
    // 绑定输入法
    if (bindCurrentInputMethodServiceLocked(mCurIntent, this, IME_CONNECTION_BIND_FLAGS)) {
        // ...
        // 绑定成功
        return new InputBindResult(
                InputBindResult.ResultCode.SUCCESS_WAITING_IME_BINDING,
                null, null, mCurId, mCurSeq,
                mCurUserActionNotificationSequenceNumber);
    }
    // 绑定失败
    return InputBindResult.IME_NOT_CONNECTED;
}

private boolean bindCurrentInputMethodServiceLocked(
        Intent service, ServiceConnection conn, int flags) {
    // ...
    // 绑定输入法
    return mContext.bindServiceAsUser(service, conn, flags,
            new UserHandle(mSettings.getCurrentUserId()));
}
```

当绑定输入法 IMS 成功后将获取到 IInputMethod 的远程服务对象 mCurMethod，通过 mCurMethod 就可以操作 IInputMethod.aidl 中定义的相关接口了，如下：
```java
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    synchronized (mMethodMap) {
        if (mCurIntent != null && name.equals(mCurIntent.getComponent())) {
            // 绑定输入法成功后获得操作输入法的接口IInputMethod
            mCurMethod = IInputMethod.Stub.asInterface(service);
            if (mCurToken == null) {
                Slog.w(TAG, "Service connected without a token!");
                unbindCurrentMethodLocked(false);
                return;
            }
            if (DEBUG) Slog.v(TAG, "Initiating attach with token: " + mCurToken);
            // 发送MSG_ATTACH_TOKEN生成与系统服务IMMS会话的token
            executeOrSendMessage(mCurMethod, mCaller.obtainMessageOO(
                    MSG_ATTACH_TOKEN, mCurMethod, mCurToken));
            // 如果当前绑定到给输入法的客户端存在则重新创建session
            if (mCurClient != null) {
                clearClientSessionLocked(mCurClient);
                requestClientSessionLocked(mCurClient);
            }
        }
    }
}
```

到此，系统输入法从编辑器 EditText 获得焦点开始到输入法 IMS 服务绑定成功的流程结束，这里注意下 startInputUncheckedLocked 方法，后续 IMM、IMMS 和 IMS 三者之间的交互关系都在其中，最终调用 showSoftInput 显示输入法。

# 输入法管理服务(IMMS)



# 6.IMS、IMMS和IMM之间的交互
输入法(IMS)、输入法管理服务(IMMS)和客户端(IMM)之间的交互主要是通过一系列 aidl 进行交互的，关键 aidl 如下图所示：

IMS、IMMS和IMM之间的交互如下：
IMM 使用 IInputMethodManager 请求 IMMS。
IMMS 绑定 IMS 获得操作输入法的相关接口 IInputMethod。
IMMS 请求 IMS 创建 IInputMethodSession。
IMMS 通过 IInputMethodClient 告知 IMM 当前 IInputMethodSession。
IMM 和 IMS 通过 IInputMethodSession 和 IInputContext 交互。
IMM、IMMS 和 IMS 三者之间的交互都是 IPC 调用，对于使用者来说无需关注三者之间的复杂调用，如上就是 Android 机制下的输入法框架。