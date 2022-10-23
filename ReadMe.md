# 升级
首先安装`npm-check`和`npm-upgrade`:
```shell
npm install -g npm-check npm-upgrade
```
安装完后，执行 `npm-check` 即可检查本地各插件版本情况。

执行 `npm-upgrade` 可根据当前版本和最新版本比较，让用户确认和选择是否升级。  
若用户确认升级，则会自动把 `package-lock.json` 和 `package.json` 文件内容进行更新后保存，然后执行：
```shell
npm update -g --save   
```

上述命令执行完毕，则所有通过 `npm-upgrade` 确认的插件全部都升级到最新（包括 Hexo）。
升级完后通过 `hexo version` 验证 Hexo 版本。

# 命令
- `hexo g`：生成网站配置
- `hexo s`：本地服务器，预览服务
- `hexo d`：发布到远程服务
- `hexo c`：清除缓存

# 配置
- `Butterfly-theme`：https://github.com/jerryc127/hexo-theme-butterfly