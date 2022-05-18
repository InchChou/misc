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

例如要编译运行实例程序 `BouncingBall.java`，首先进入 java 文件所在的目录，然后编译运行。在 windows 平台下执行以下操作：

```powershell
cd xxxxx

# 将BouncingBall.java 编译为 BouncingBall.class
javac -cp ..\algs4.jar .\BouncingBall.java

# 运行 BouncingBall.java
java -cp ..\algs4.jar .\BouncingBall.java # 不标准，见后续讨论
```

其中有三个需要注意的地方：

1. 这里需要先编译再运行，因为运行时 java 虚拟机需要读取与 java 文件同名的 class 文件（编译得到的字节码文件），否则的话直接运行会报以下错误：

```powershell
错误: 找不到或无法加载主类 BouncingBall
原因: java.lang.ClassNotFoundException: BouncingBall
```



2. 在编译和运行的时候都带上了 `-cp ..\algs4.jar` 参数，因为在 `BouncingBall.java` 中引入了 jar 包 `algs4.jar`，所以要将它的路径指定，这样的话编译器和虚拟机才能到指定位置寻找 jar 包并正常编译运行，否则的话会出现找不到类的错误。

> java 中有一个比较重要的概念：**类加载路径**，即 CLASSPATH。虚拟机类加载器加载类的路径只能在classpath类加载路径指明的位置中查找。



3. 这里的运行形式与书中的 `java BouncingBall` 不同，带上了 java 后缀，不带上的话会报错：

```powershell
错误: 找不到或无法加载主类 BouncingBall
原因: java.lang.ClassNotFoundException: BouncingBall
```

这是因为 java 虚拟机在运行时也会在 `CLASSPATH` 中寻找需要执行的类。上面指定了 `-cp ..\algs4.jar` 参数，当使用了这个参数（或者 `-classpath`）后，环境变量 `CLASSPATH` 就不起作用，转而使用 `-cp` 所指定的路径了。这里的 `-cp` 只指定了 `..\algs4.jar`，没有指定 `BouncingBall` 所在的路径（即为当前路径），所以在运行时自然找不到 `主类 BouncingBall` 了。

解决办法就是将 `-cp ..\algs4.jar` 参数修改为： `-cp ..\algs4.jar;.`，后面的 `.` 指的即为当前路径。

> 在执行运行命令时，需要在 `-cp` 参数所指定的路径中添加上目的 java 文件的目录。



**如果 java 文件不在当前路径怎么办？**

例如 java 文件在 `src` 目录中，jar 文件在当前目录，则编译和运行命令如下：

```powershell
javac -cp ".\algs4.jar" src\BouncingBall.java
java -cp ".\algs4.jar;.\src" BouncingBall
```



> 引用：
>
> [Java命令行运行错误之找不到或无法加载主类问题的解决方法_java_脚本之家 (jb51.net)](https://www.jb51.net/article/235808.htm)