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

