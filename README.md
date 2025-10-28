# 博客自定义修改记录

## 需要从旧主题(bak)复制到新主题(papermod)的文件：

### 1. TOC相关文件
- [`themes/bak/layouts/partials/toc.html`](themes/bak/layouts/partials/toc.html:1)
- [`themes/bak/assets/css/extended/toc.css`](themes/bak/assets/css/extended/toc.css:1)

### 2. 扩展样式文件
- [`themes/bak/assets/css/extended/code.css`](themes/bak/assets/css/extended/code.css:1)
- [`themes/bak/assets/css/extended/friend.css`](themes/bak/assets/css/extended/friend.css:1)
- [`themes/bak/assets/css/extended/tagscloud.css`](themes/bak/assets/css/extended/tagscloud.css:1)

### 3. 滚动条样式
- [`themes/bak/assets/css/includes/scroll-bar.css`](themes/bak/assets/css/includes/scroll-bar.css:1)


## 需要检查的其他文件
- 标签云样式
- 友链页面样式
- 代码块样式优化
- 滚动条统一样式

## 使用命令
```bash
hugo server -D --bind 0.0.0.0 --port 1313
```
