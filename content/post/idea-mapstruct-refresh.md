---
title: "IDEA手动触发MapStruct生成Mapper实现" 

date: 2021-02-05T17:30:00+08:00 

categories: ["Java"]

tags: ["Java", "工作效率"] 

author: "Me" 

showToc: false

TocOpen: false 

draft: false 

hidemeta: false 

comments: false 

description: "" 

disableHLJS: true  

disableShare: false 

cover:
  image: "https://raw.githubusercontent.com/jiayanju/imgrepo/main/idea.png"
  alt: ""
  caption: ""
  relative: false
  hidden: false



---



项目中使用了MapStruct来生成前端的request json对象到后端DTO或者domain的映射。

但是有的时候在web项目的类里面添加了字段，相关的MapperImpl实现类并没有更新，还是旧的设值逻辑。但是大部分时候会自动更新。

如果使用maven的complile任务去触发也可以，但是时间有的长。

出现这个问题的时候很是烦人，影响工作效率。

试了一下，可以使用IDEA的compile对Mapper进行编译，就会再次生成Mapper实现类

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/idea-mapstruct.png)

希望对有这个问题的同学有点帮助。