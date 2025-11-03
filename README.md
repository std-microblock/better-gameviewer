## Better GameViewer

GameViewer / UU 远程补丁，修复了 UU 远程在 Windows 下键盘/鼠标输入的数个问题：
- 鼠标无法双击拖拽
- 键盘输入时漏事件导致某个键一直被按下/按不下去
- 有时其他地方的输入被输入到远程机器内去

### 使用
<img width="607" height="239" alt="image" src="https://github.com/user-attachments/assets/358d3522-0628-48b6-b3ad-57cb90e39050" />

将 `HID.dll` 放置于 GameViewer/bin 文件夹下即可

### 技术细节

评论区有群友提到一个 Bug ，在双击的时候对鼠标行为的处理是不正确的；我稍微逆向研究了一下，把这个 Bug 修掉了

<img width="1079" height="1266" alt="image" src="https://github.com/user-attachments/assets/e7562953-3873-4a31-a628-3465b59eaa6c" />

分析图中关键代码可知（函数名都是我乱标的，不要在意），网易 UU 接收了 WM_LBUTTONDBLCLK 事件（0x140318171），然后将这个事件用一个单独的枚举值，也就是 4，表示 “收到了一个双击”，传递给按键处理函数；然后将全局变量 data_14214acdc 赋值为 1，这个全局变量其实是一个 flag，表示“跳过下一个 WM_LBUTTONUP”（0x140318080）

这样操作以后，如果输入序列为
按下 弹起 按下 弹起
则消息处理函数会按顺序收到
 2 3 4
分别表示 单击按下，单击弹起，双击事件

这其实是很奇怪的一种行为，因为双击事件是在第二次鼠标按下的时候触发的，此时我们的鼠标依然为按下状态；而网易 UU 远程桌面将其擅自转化为一次双击事件，导致鼠标提前抬起按键，以及后续的一次抬起鼠标事件被吃掉，会导致我们鼠标的手感非常不一样。

修复方法其实也很简单，从 Microsoft 文档可见：
双击鼠标左键实际上会生成四条消息的序列：WM_LBUTTONDOWN、WM_LBUTTONUP、WM_LBUTTONDBLCLK和 WM_LBUTTONUP

所以，我们完全可以将这个 WM_LBUTTONDBLCLK 直接当成 WM_LBUTTONDOWN 处理，依然发送为单击事件，问题解决，代码复杂度也降低了许多

不知道这块代码是谁写的...有点怪，明明注册了 Raw Device 但是却不用相关的数据，转而继续使用常规的 WM 方式决定鼠标状态；而且 Windows 默认是不发送双击事件的，双击事件只有在注册特殊 style 后才会发送，说明是哪个大聪明专门注册了这个 style...

如果有认识 UU 那边的人的群友可以帮忙转告一下，让他们顺手把这个很影响体验的 bug 修了~

<img width="1031" height="372" alt="image" src="https://github.com/user-attachments/assets/301e8253-b29c-4431-8393-4175c953c852" />

<img width="268" height="152" alt="image" src="https://github.com/user-attachments/assets/56bc7659-da73-498c-94f1-a0fea2e9cc08" />


然后发现 uu 的键盘输入也有问题...太离谱了

他们居然是用低级键盘钩子做的输入...在拿到焦点的时候安装钩子，失去的时候取消钩子

低级键盘钩子是没什么问题的，毕竟要拦截 Windows 徽标键之类的特殊按键...但是这玩意会漏输入的啊（如图三）...

解决方法也很简单，直接把他这个 hook 关掉，然后把键盘事件从正常的 WndProc 里面接出来给他就可以了
