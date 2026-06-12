
|**维度**|**含义**|**你要学会什么**|
|---|---|---|
|K|Knowledge 知识|软件工程概念、需求、设计、构造、测试、过程|
|A|AI 协作与判断|写 Prompt、审查 AI 输出、发现冲突、做取舍|
|S|Skill 工程技能|写文档、建模、编码、调试、Git、测试|
|D|Discipline 过程与态度|时间管理、团队协作、自主学习、工程纪律|

AI  编程常见能力包括：
代码生成、代码解释、Bug修复、代码注释、测试生成、代码重构

角色：让 AI 以什么身份工作，例如“你是熟悉 Spring Boot 的后端工程师”。  
任务：让 AI 做什么，例如“定位 NullPointerException 的根因”。  
约束：不允许做什么，例如“只修改出错方法，不要重构其他文件”。  
格式：输出什么，例如“给出完整方法代码，并解释修改原因”。

### 开发工具链与Git 工作流

Git 部分讲了四个区域：
  
工作区：你正在编辑的文件。  
暂存区：准备提交的快照。  
本地仓库：保存提交历史。  
远程仓库：团队共享的版本库。


基本命令包括：

`git clone` 克隆项目。
`git add` 添加到暂存区。
`git commit -m` 提交到本地仓库。
`git push` 推送到远程。
`git pull` 拉取并合并远程更新。
`git checkout -b feature/...` 创建功能分支。
`git status` 查看状态。`
git log --oneline` 查看历史。`
git diff` 查看修改。`
git reset HEAD` 取消暂存。`
git reset --soft HEAD~1` 撤销最近提交但保留修改。`
git reset --hard` 会永久丢弃修改，所以要特别小心。

团队协作采用功能分支和 Pull Request。每个功能在自己的 `feature` 分支开发，完成后发 PR，由队友 Review，再合并到主分支。这不是形式主义，而是为了防止未经审查的代码进入主线。

### AI编程进阶与Agent时代：从“AI辅助人”到“人辅助AI”

AI 从局部补全走向项目级 Agent，但仍然需要人提供目标、约束、审查和验收

项目级AI的关键时上下文，让AI在Spring Boot项目中新增“按角色查询用户列表”街头，如果AI只看到Controller，它可能会重复创建Service、Mapper、XML，甚至破坏已有架构。
正确做法是先让AI扫描项目结构，了解已有接口、数据表、Mapper、Service规范，再生成修改方案
安全问题、性能问题、边界条件、可维护性、命名规范、异常处理、业务一致性等。AI 比较擅长发现语法、重复代码、常见漏洞和局部坏味道，但对业务隐含规则、架构取舍、需求合理性仍然不可靠。

Spec 和 skill 
Spec :Specification 规格说明/任务规格/需求说明
Spec 是给 Agent 的精确任务说明，例如接口路径、请求体、返回格式、权限、密码加密、修改文件范围。

Skill：技能/可复用工作流/固定操作套路
Skill 是可复用流程，比如“新增 API 总是创建 Controller、Service、Mapper、XML、Entity 并写基础测试”。
> Agent 越强，Spec 越重要。AI 时代的软件工程不是需求工程变得不重要，而是需求工程变得更重要。

### 前端开发基础：现代前端从静态页面变成独立工程体系
DOM树：Document Object Model Tree 文档对象模型树
浏览器读取HTML后，会把HTML标签解析成一棵”树状结构“，这棵树就是DOM树
```html
<!DOCTYPE html>
<html>
  <head>
    <title>我的页面</title>
  </head>
  <body>
    <h1>欢迎</h1>
    <p id="intro">这是第一段文字</p>
    <button>点击我</button>
  </body>
</html>*
```

浏览器会大致把它理解为：
```
document
└── html
    ├── head
    │   └── title
    │       └── "我的页面"
    └── body
        ├── h1
        │   └── "欢迎"
        ├── p#intro
        │   └── "这是第一段文字"
        └── button
            └── "点击我"
```
DOM树不是写出来的HTML文本本身，而是浏览器把HTML解析后生成的对象结构

Web 的前端三层：
HTML：内容和结构层，它描述标题、段落、连接、图片、表单、语义结构。流拉起把HTML解析为DOM树
CSS：样式和表现层。它控制颜色、字体、布局、动画。浏览器根据CSS选择器计算样式，再和DOM合成渲染树
JavaScript ： 行为层。它处理事件、操作DOM、发送异步请求、更新页面状态

**Vue** ： **数据驱动视图**
Vue.js 是一个用于构建用户界面的 JavaScript 前端框架
Vue让你不用频繁手动操作DOM，而是通过“数据驱动视图”的方式开发界面
原生JavaScript需要手动查找DOM、更新文本、同步状态，复杂界面中会导致数据和视图耦合、状态分散、维护成本上升；
Vue的核心思想是数据驱动视图，开发者修改响应式数据，框架自动更新DOM

**Node.js V8引擎 事件循环 非阻塞I/O**
Node.js 让 JavaScript 不只能运行中浏览器里，也能运行在电脑、服务器、命令行和后端环境中

Node.js 适合API 服务和实时场景，同时让前后端都使用JavaScript，降低全栈切换成本

> 浏览器 JS 主要控制网页；Node.js 主要让 JS 具备服务器、文件系统、命令行、网络编程能力。

V8引擎：
V8 上 Google Chrome 使用的 JavaScript 引擎
它负责把 JavaScript 代码解析、编译、执行

Node API：
浏览器提供DOM API，而Node.js 提供的是服务器开发相关API，例如
```JavaScript
fs      // 文件系统
http    // HTTP 服务器
path    // 路径处理
os      // 操作系统信息
crypto  // 加密
events  // 事件机制
stream  // 流处理
```

事件循环和非阻塞I/O
事件循环 Event Loop 
非阻塞I/O Non-blocking I/O

遇到耗时操作时，Node.js不会傻等，而是先去处理其他任务，等耗时操作完成后再执行回溯

Node.js 和 npm的关系
npm = Node Package Manager Node包管理器

用于安装、管理第三方库

Express
Express 是 Node.js 中最常见的 Web 后端框架之一

### Java Web 后端开发：接口、数据、安全三层防线
Java Web 后端开发就是用 Java 写服务器端程序，让前端、浏览器、移动App、Postman 等客户端可以通过网络访问后端提供的功能

> **接受HTTP请求，执行业务逻辑，安全地访问数据库，然后返回JSON数据**

**HTTP协议**
HyperText Transfer Protocol 超文本传输协议 是应用层协议
浏览器访问服务器时，本质上就是发送 HTTP Request ，服务器处理后返回 HTTP Response

HTTP有一个重要特点 **无状态**
服务器默认不会记住“上一次请求是谁发来的”
你第一次请求`/login` 第二次请求 `/api/sites` HTTP协议本身不会自动把这两个请求关联起来

因此需要
```
Session / Cookie
或者
JWT Token
```
来解决“用户身份保持”的问题

**URL**
URL 是网络资源地址。比如：

```
http://blog.example.com:8080/users?gender=male&page=1
```

可以拆成：

```
http://              协议
blog.example.com     主机名
8080                 端口
/users               路径?
gender=male&page=1  查询参数
```

在 Java Web 后端中，你经常会设计这样的 URL：

```
GET    /api/sites
GET    /api/sites/1
POST   /api/sites
PUT    /api/sites/1
DELETE /api/sites/1
```

这些 URL 就是后端 API 的入口。

**HTTP Request请求格式**
一个 HTTP 请求通常包括三部分：

```
请求行
请求头 Headers
请求体 Body
```

例如：

```
POST /api/sites HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "name": "南京大学",
    "url": "https://www.nju.edu.cn"
}
```

请求行里有三个东西：

```
POST            HTTP 方法
/api/sites      请求路径
HTTP/1.1        协议版本
```
常见HTTP方法：

| 方法     | 作用   |
| ------ | ---- |
| GET    | 获取资源 |
| POST   | 新增资源 |
| PUT    | 更新资源 |
| DELETE | 删除资源 |
这几个方法后面会直接映射到 Spring Boot 的注解：

```java
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
```

**Spring Boot： 快速构建 RESTful API**
Spring Boot 的目标是解决传统 Spring 项目配置繁琐的问题
传统 Spring 需要大量XML配置，而 Spring Boot 通过自动配置、Starter 依赖、内嵌服务器让项目可以快速启动
