1. 下载hugo,下载extend,不然模版的scss可能用不了
```
https://github.com/gohugoio/hugo/releases
```
2. 新建一个文件夹后进入后命令
```
hugo new site 我的博客名
```
3. 下载主题,主题列表
https://themes.gohugo.io/
    - 使用git下载主题或者直接wget什么的都行,具体看模版维护者的使用说明
    ```
     git submodule add https://github.com/hugo-next/hugo-theme-next.git themes/hugo-theme-next
    ```
4. 创建第一个md
```
hugo new tests/ttt.md
```
![](/static/images/kube/event1.png)