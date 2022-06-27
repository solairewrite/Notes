# 插件
Bracket Pair Colorizer 2  
Markdown All in One  
	Ctrl + Shift + P: Create Table of Contents,在光标位置生成目录  
vscode-icons  
Markdown Preview Enhanced  
Markdown Preview Github Styling  

## Markdown设置代码字体大小
Ctrl + Shift + P 搜索 Customize CSS  
修改style.less文件  
```
.markdown-preview.markdown-preview {
  font-size: 18px;

  code {
    // 自动换行
    white-space: pre-wrap;
    font-size: 16px;
  }
}
```

打开 C:\Users\jizhixin\.vscode\extensions\shd101wyy.markdown-preview-enhanced-0.6.3\node_modules\@shd101wyy\mume\styles\preview_theme\github-light.css  
将里面的!important前面的!删除  
