+++
title = 'NestJS 学习笔记'
date = 2024-04-10T17:51:55+08:00
draft = true
tags = [ "nestjs" ]

+++



基础命令

```bash
# 创建项目
nest new project-name

# 创建控制器
nest g controller cats

# 创建模块
nest g module cats

```



文件结构

```
src
 ├── app.controller.spec.ts
 ├── app.controller.ts
 ├── app.module.ts
 ├── app.service.ts
 └── main.ts
```

|                        |                                                              |
| :--------------------- | ------------------------------------------------------------ |
| app.controller.ts      | 带有单个路由的基本控制器示例。                               |
| app.controller.spec.ts | 对于基本控制器的单元测试样例                                 |
| app.module.ts          | 应用程序的根模块。                                           |
| app.service.ts         | 带有单个方法的基本服务                                       |
| main.ts                | 应用程序入口文件。它使用 `NestFactory` 用来创建 Nest 应用实例。 |

