# WeexSaoLei

纯用Weex内置模块做的一个简单的扫雷游戏

+ `/Minesweeper.vue`是主代码文件，连同图片资源复制到weex项目中使用。
+ `/image`里面的是代码中要用到的图片资源，当然，图片资源的引用方式视个人项目配置不同而不同，请把代码中`require(...)`这部分代码改成适合自己项目的，否则代码可能报错
+ 游戏规则:单击格子可以清除旗子或打开格子，长按或者双击可以插旗，找出所有雷为赢，点开雷为输。
+ 已实现功能：单击、双击、长按识别；首次点击不会触雷；自动展开连续空白的格子
+ 可拓展功能：难度升级（修改雷数和格子数）
