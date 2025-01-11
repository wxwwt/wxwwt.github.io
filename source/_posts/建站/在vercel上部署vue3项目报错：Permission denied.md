---

title: 在vercel上部署vue3项目报错：Permission denied
date: 2025-01-11 20:00:00
updated: 2025-01-11 20:00:00
tags: 建站

---



今天部署的是一个vue3的项目，结果一直报错，

![image-20250111193616782](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20250111193616782.png)

根据文字我以为是什么权限不够，结果就找了一些资料看看，尝试好几个方案之后发现，竟然是要在

要在部署的命令里面增加一个 rm -rf node_modules && npm install

原本默认的是npm run build

修改完之后变成了：rm -rf node_modules && npm install && npm run build

就感觉挺离谱的，难道vercel能识别到这个项目是vue3，但是不会自动去安装依赖吗？后来也没研究具体原因了，反正改完就能用了。

![image-20250111172258083](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20250111172258083.png)