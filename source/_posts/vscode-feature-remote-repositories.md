---
title: "VS Code 新功能: Remote Repositories"
date: 2021-06-12 16:35:00
categories: [news]
tags: [ide, vscode, github, development]
---

VS Code 在 [1.57](https://code.visualstudio.com/updates/v1_57) 版中，
Remote Development 系列 extension 加入了新成員:
[Remote Repositories](https://marketplace.visualstudio.com/items?itemName=GitHub.remotehub)。

有了這個 extension 之後，如果遇上臨時想看的 project，就可以直接在 VS Code
中叫出來看，不需要事先 clone 至某個 local 資料夾。

不過.. 因為這個 extension 實際上是建一個
[Virtual Workspaces](https://github.com/microsoft/vscode/wiki/Virtual-Workspaces)
並把 code 放在裡面閱覽，
所以用 Remote Repositories 開出來的 workspace 功能非常受限。
諸如 Debug, Terminal 及大部分的 extension 基本上都不能用。
但話雖如此，當看 code 看一看想要開始進行比較深入的修改及除錯時，
其實也是有提供轉換成一般 workspace 的功能。 使用上非常方便！

可惜的是，目前此 extension 支援的 remote repository 種類只有 GitHub。
且如同其他 Remote Development Series，這個 extension 並非 open source project：

- [Visual Studio Code Remote Development Frequently Asked Questions](https://code.visualstudio.com/docs/remote/faq#_why-arent-the-remote-development-extensions-or-their-components-open-source)
- [Cannot use Remote Development extension pack · Issue #196 · VSCodium/vscodium](https://github.com/VSCodium/vscodium/issues/196)

未來會不會支援 GitHub 以外的 Git repositories，甚至其他種類的 VCS，
只能看微軟爸爸的眼色了。
