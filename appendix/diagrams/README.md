# 附录：配图/示意图

本目录用于存放教程中的配图和示意图。

## 建议使用的图片格式

- PNG/SVG：适合架构图、流程图
- GIF：适合动态演示（如 B+Tree 查找过程）

## 图片命名规范

```
ch{章节号}-{描述}.{格式}

例如：
ch03-page-structure.png
ch04-bplus-tree-index.png
ch05-redo-undo-relation.png
```

## 图片存放建议

也可以考虑将图片放在各章节目录下的 `images/` 文件夹中，这样在 Markdown 中引用路径更短：

```markdown
![Page 结构](images/page-structure.png)
```
