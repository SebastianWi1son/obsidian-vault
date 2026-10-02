![](asset/merge.canvas)## 1. Git 解决什么问题
### 1.1 单线迭代
如何保证我加新功能不会把整个项目搞炸--commit
### 1.2 并行开发
如果开发过程中旧版本反馈了bug怎么办--branches
### 1.3 协作开发
多人开发怎么办--分布式repo

## 2. git配置文件层级
| 层级     | 作用范围      | 优先级    |
| ------ | --------- | ------ |
| system | all user  | low    |
| global | curr user | medium |
| local  | curr repo | high   |

## 3. 什么是git repo
> 存储项目中所有commit记录， branch结构的repository
- github每个项目都有一个或者多个git repo
- 现有repo才能让git管理你的仓库

## 4. commit & branches
对commit和branch的基本认识
- 箭头应该新的指向旧的
- branch在指向commit而不是在箭头上
![](asset/branch.canvas)
## 5. merge
- merge先回main
- 从最近的公共commit开始计算
- HEAD指向谁，谁就移动
- main来"拉取"develop
- 拒绝fast-forward merge
![](asset/merge.canvas)

## 6. merge conflict
