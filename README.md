# BewareMyPower.github.io

绿灯军团誓词：

> In brightest day, In blackest night.
> No evil shall escape my sight.
> Let those who worship evil's might.
> Beware my power,
> Green Lantern's light!

## 使用说明

首先下载 hexo 分支及第三方库（`next` 主题）

```bash
git clone https://github.com/BewareMyPower/BewareMyPower.github.io.git -b hexo --recursive
```

或者

```bash
git clone https://github.com/BewareMyPower/BewareMyPower.github.io.git -b hexo
git submodule update --init --recursive
```

然后用本地配置覆盖 next 主题的默认配置

```bash
cp themes/next_patch/_config.yml themes/next
```

此外，不知为何，在Windows上部署时，`_config.yml`配置`language`会失效，一直显示德文，即使手动改成`en`或者`zh-CN`也是。

暴力解决方法：把 `themes/next/languages/` 下的除了 `zh-CN.yml` 外的文件全部删了：

```bash
cd themes/next
mv languages languages.bak
mkdir languages
cp languages.bak/zh-CN.yml languages
```
