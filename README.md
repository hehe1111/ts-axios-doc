# ts-axios-doc

TypeScript 从零实现 axios 文档教材

## 启动电子书

首先 clone 本项目：

```bash
git clone https://github.com/hehe1111/ts-axios-doc.git
```

【node 的版本必须小于等于 16，否则 npm run dev 会报错并直接退出】

进入 `ts-axios-doc` 目录后安装项目依赖：

```bash
npm install
```

安装依赖后运行电子书：

```bash
npm run dev
```

浏览器打开 `http://localhost:8080/ts-axios/` 即可。

## 发布到 gh-pages

请先确保远程仓库已有 gh-pages 分支及在 settings 里开启了 gh-pages 预览

```bash
npm run deploy
```
