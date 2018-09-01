---
published: true
layout: post
catagories: cheatsheet
publish: true
author: Moevis
---
做毕设的时候用 matplotlib 画图表，当时有一个想法是给坐标轴加上中文，但是发现要设置一些东西，当时时间比较赶，就算了。今天正好想写一点 matplotlib 教程，所以就重新研究一下。

实际上你需要做的是告诉 matplotlib 去用指定字体，因为 matplotlib 默认用的是 DejaVu Sans，这个字库包含了大多数西文字体，但是不包含东亚字体，所以如果遇上中文，会变成黑框框。

在 mbp 上有很多预装的中文字体比如苹方啊，汉仪之类的，但是用它们的难点是在于要知道它们的字体英文名字。

实际上你可以用下面的命令获取系统中的字体：

```python
import matplotlib.font_manager as fm

fonts = fm.findSystemFonts()

lst = []
for f in fonts:
    font = fm.FontProperties(fname=f)
    try:
        lst.append((f,font.get_name(), font.get_family()))
    except Exception as e:
        print(f)
df = pd.DataFrame(lst, columns=['path', 'name', 'family'])

df.head()
```

|path                                                       | name                 | family         |
|-----------------------------------------------------------|:---------------------|:---------------|
| /Library/Fonts/STIXSizTwoSymBol.otf                        | STIXSizeTwoSym       | ['sans-serif'] |
| /Library/Fonts/PlantagenetCherokee.ttf                     | Plantagenet Cherokee | ['sans-serif'] |
| /Library/Fonts/Wingdings 3.ttf                             | Wingdings 3          | ['sans-serif'] |
| /Library/Fonts/Zapfino.ttf                                 | Zapfino              | ['sans-serif'] |
| /Users/moevis/Library/Fonts/SourceCodePro-ExtraLightIt.otf | Source Code Pro      | ['sans-serif'] |
| ... | ... | ... |

你可以在里面找到你想要的字体，当然还有一个更方便的方法是去 mac 自带的 Font Book 应用中找。

如果你是新安装了字体，那么可以运行这句话更新 matplotlib 字体缓存：

```python
from matplotlib.font_manager import _rebuild; _rebuild()
```

使用字体也很简单：

```python
matplotlib.rcParams['font.sans-serif'] = ['LiSong Pro']
plt.title('正弦')
plt.show()
```

我筛了几个可用的字体如下：
`BiauKai`, `Hei`, `Kai`, `LingWai SC`, `STFangsong`, `STHeiti`。

我还在 [https://www.google.com/get/noto/](Google Noto Fonts) 上下载了 noto sans cjk sc，安装了 `Noto Sans CJK SC`，和 `Noto Sans Mono CJK` 也都可以用。

最后在强调一下，安装了新字体一定要记得更新字体缓存。
