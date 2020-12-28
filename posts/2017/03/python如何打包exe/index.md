# Python如何打包exe


>项目中有时会写些py脚本文件，为当作项目工具，方便无Python环境下使用，所以需要打包成exe文件。

<!--more-->

## Python打包exe

* Q：py生成exe，总共需几步？
* A：总共分三步！

### 1. 安装PyInstaller


```python
pip install PyInstaller
```
* 注意：安装包名区分大小写

### 2. 打包脚本:TargetPy2exe.py.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


"""
@version: ??
@author: Binge
@file: TargetPy2exe.py.py
@time: 2017-02-07 11:21
@description: convert py to exe by pyinstaller
"""

from PyInstaller.__main__ import run

if __name__ == '__main__':
    # 设置打包exe参数：目标py、打包参数
    # -F 打包成一个exe文件
    # -w 使用窗口，无控制台
    # -c 使用控制台，无窗口
    # --icon = 图标路径
    # --upx-dir 使用upx压缩
    # upx391w ups程序目录文件路径


    # opts = ['tvn_process.py', '-F']
    opts = ['tvn_process.py', '-F', '-w']
    # opts = ['tvn_process.py', '-F', '-c']
    # opts = ['tvn_process.py', '-F', '-w', '--upx-dir', 'upx391w']
    # opts = ['tvn_process.py', '-F', '-w','--icon=tvn_process.ico','--upx-dir','upx391w']

    run(opts)
```


###  3. 运行打包脚本，即可生成exe文件



