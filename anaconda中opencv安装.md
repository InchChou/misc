这里我使用的 Python 发行版是 Anaconda，在windows中使用，通过 `conda` 命令安装所需的依赖包。使用 vscode 作为编辑器。

## 虚拟环境

Anaconda 中包含有虚拟环境管理，默认虚拟环境为 `base`，可以通过 `conda activate <env-name>` 来激活对应的虚拟环境。

OpenCV 最好不要装在 base 环境中，如果装在了 base 环境中的话，在 vscode 中运行使用了 OpenCV 的 Python 脚本的话，会找不到 cv2 包。所以最好新建一个环境，将 OpenCV 装在新环境中，在 VSCode 中调试脚本时，使用此新环境作为 Python 解释器。

```powershell
conda create -n opencv
conda info --envs
conda activate opencv
```

## 安装 OpenCV

```powershell
conda install -c conda-forge opencv
```

## 在 VSCode 中设置 Python 代码补全

在默认设置下， VSCode 能补全 Python 的部分代码，但是无法对 OpenCV 进行自动补全，此时需要改变 VSCode 的 Python 扩展的设置，将 `languageServer` 设置为 Jedi。

```
"python.languageServer": "Jedi"
```

