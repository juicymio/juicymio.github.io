# WSL2中C/C++开发环境配置

## 基础
- 编辑器: LunarVim
- 基于clangd的代码提示, compile_commands.json由bear生成, 命令是`bear -- make`
- tmux管理窗口(目前又改成了zellij, 感觉快捷键更好用一点)
## 美化
- WSL2: 运行在Windows Terminal中, 安装oh my posh, One Half Dark主题
- 字体: Fira Code Nerd Font (为了显示LunarVim以及终端中各种小图标)
- Powerlevel10k

效果:  
![img][https://upload.cc/i1/2023/07/08/J54ryo.png]
![img][https://upload.cc/i1/2023/07/08/v7qjNg.png] 

比之前那个运行在CMD里调个窗口大小就闪退, 配色还极其阴间, 显示不出来图标的状态强了不知道多少.  

## 效率提升
- lazygit
- LunarVim中打包的一些插件差不多够用, 懒得折腾新的了
## 问题
- 挂代理也连不上各种东西, 经一通检查之后发现
![088274827cd90a0270e69debfb4bfd3a.jpg](https://s2.loli.net/2023/08/28/st6uFvakz3OY8RL.png)
感觉是wsl和steam++一起干的好事...

