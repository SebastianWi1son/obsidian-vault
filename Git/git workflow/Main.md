## 1. Semantic Versioning
> SemVer 管的是“发布版本”，不是开发过程中每一次代码变化。

```
MAJOR.MINOR.PATCH
  2  .  3  .  1
```
- **MAJOR:** 有不兼容的API更新
- **MINOR:** 有向后兼容的功能更新
- **PATCH:** 无新功能,修旧版本bug

## 2. 跟进主分支
> 避免在master上merge造成bug

- **跟进:** 有新的branch在旧分支进行了修改，先将该分支的修改pull进本分支

## 3. `rebase`
- 嫁接变基，拉直commit chain
- 过程中可能会有多次merge conflict
![](asset/rebase.canvas)
## 4. 
### 4.1 amend
### 4.2 stash
### 4.3 checkout detached