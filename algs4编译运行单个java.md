在学习 algs4 的过程中有时候会遇到需要运行单个 java 文件的情况。我在学习过程中主要用的编辑器是 idea，其次是 vscode，如果需要运行单个 java 文件，我目前已使用的方法有两种：

1. 使用 idea 编译运行单个java；
2. 使用命令行编译并运行单个 java。

## idea编译运行单个java

使用 idea 编译运行单个 java 比较麻烦，因为 idea 主要是为整个工程设计的。如果在 `source` 目录下有许多 java 文件，在构建的时候会将这些文件都编译一遍。这时其他的 java 文件如果有错误的话，就无法编译完成并调试。所以需要将不进行调试的 Java 文件排除。按以下步骤操作：

1. 点击<kbd>File</kbd> -> <kbd>Settings</kbd> -> <kbd>Build,Execution,Deployment</kbd>，进入构建页面；
2. 点击 `Compiler` 条目，选择 `Exludes`；
3. 点击 <kbd>+</kbd> 添加需要排除的文件（除了需要调试的文件，其他的都需要加进去），如 `xxx.java`；
4. 最后点击 <kbd>OK</kbd> 来应用修改
5. 观察主界面左边的 `Project` 视图，此时其他的 java 文件的图标的右上角有一个 `x` 号，表示已被排除，找到自己需要调试的文件，在文件上右击，选择 `Run xxx` 或者 `Debug xxx` 即可。

## 使用命令行编译并运行单个 java

