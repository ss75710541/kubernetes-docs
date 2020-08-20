# macOS Chrome访问https://registry-console-default.appsxxx.xxx.xxx/页面显示ERR_CERT_INVALID，且不能点继续


有些网站可以选择跳过，继续访问，但是偶尔有些网站不允许继续，且提示：


```
您的连接不是私密连接
攻击者可能会试图从 XX.XX.XX.XX 窃取您的信息（例如：密码、通讯内容或信用卡信息）。了解详情

NET::ERR_CERT_INVALID

将您访问的部分网页的网址、有限的系统信息以及部分网页内容发送给 Google，以帮助我们提升 Chrome 的安全性。隐私权政策

XX.XX.XX.XX 通常会使用加密技术来保护您的信息。Google Chrome 此次尝试连接到 XX.XX.XX.XX 时，此网站发回了异常的错误凭据。这可能是因为有攻击者在试图冒充 XX.XX.XX.XX，或 Wi-Fi 登录屏幕中断了此次连接。请放心，您的信息仍然是安全的，因为 Google Chrome 尚未进行任何数据交换便停止了连接。

您目前无法访问 XX.XX.XX.XX，因为此网站发送了 Google Chrome 无法处理的杂乱凭据。网络错误和攻击通常是暂时的，因此，此网页稍后可能会恢复正常。
```

经过很多尝试，发现只有一种有效方法可以跳过：

在chrome该页面上，直接键盘敲入这11个字符：`thisisunsafe`

（鼠标点击当前页面任意位置，让页面处于最上层即可输入）

参考 ：http://www.hackdig.com/03/hack-68770.htm