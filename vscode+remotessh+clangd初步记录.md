最近使用云 IDE 进行开发，它实际上是在一个容器内运行了 vscode 的 server，然后预装了 clangd、gitlens 等插件。使用下来的感受只能说能用，其搜索和跳转功能基本上是废的。看到其有 IP 和端口，于是想到了使用 Remote-SSH 连接到云 IDE 进行使用。

### Remote-SSH 设置

如果知道云 IDE 用户名和密码，可以直接使用 Remote-SSH 连接对应的 IP，输入用户名和密码使用；也可以使用 rsa 公私钥来进行 ssh 连接使用。下面简要说明使用秘钥如何连接：

1. 使用 ssh-gen 生成公私钥，一般生成在 `C:\Users\<用户名>\.ssh` （windows）或 `~/.ssh` （Linux）下，如果是 rsa 加密的，会生成两个文件 `id_rsa`（私钥）和 `id_rsa.pub`（公钥）
2. 在云 IDE 的 `~/.ssh`  下新建一个 `authorized_keys` 文件，将刚刚生成的 `id_rsa.pub` 中的内容拷贝进去
3. 修改 ssh 的配置文件 `C:\Users\<用户名>\.ssh\config`， vscode 的 Remote-SSH 插件会读取它，来连接服务器：

```yaml
Host <想给云IDE取的名字>
  HostName <云IDE的IP>
  User <用户名，一般为honor>
  Port <端口号>
  IdentityFile "~/.ssh/id_rsa"

Host *
  HostkeyAlgorithms +ssh-rsa
  PubkeyAcceptedKeyTypes +ssh-rsa
```

然后在 vscode 中选择 CloudIDE 项，连接即可，它会自动在云 IDE 中下载 `.vscode-server` 并安装运行相关服务。

### Clangd 设置

在云 IDE 中下载代码，并通过 Remote-SSH 连接云 IDE 后，就可以打开项目目录进行工作了，但是此时还无法进行代码补全和跳转，这里推荐用 clangd 帮助完成。

> clangd 文档见 [What is clangd? (llvm.org)](https://clangd.llvm.org/)

推荐使用 `compile_commands.json` 文件来进行 clangd 的索引。

在云 IDE 中下载好代码后，推荐先将它编译一遍，看是否能够编译通过。然后再次编译，在编译前加上 COMPDB 相关的环境变量，用以生成 `compile_commands.json`。`SOONG_LINK_COMPDB_TO` 是可选项：

```bash
export SOONG_GEN_COMPDB=1
export SOONG_GEN_COMPDB_DEBUG=1
export SOONG_LINK_COMPDB_TO=<你要创建compile_commands.json软链接的目录>
```

调用 `./build_system.sh` 及相关参数编译，此时会生成 `out/soong/development/ide/compdb/compile_commands.json`。如果设置了 `SOONG_LINK_COMPDB_TO`，会在设置的目录下生成软链接。

这里生成的 `compile_commands.json` 包含了此次编译涉及到的所有模块的编译命令，理论上可以直接使用，但不推荐。这个文件过于庞大，在我的云 IDE 中有 700+MB，并且因为我们只关注项目对应的模块，所以最好进行精简。这里使用了 python 脚本进行精简。

```python
import json
import sys
from typing import *

def simplify_compile_commands_json(completed_json_file: str, simplified_json_file: str, repositories: List[str]):
    with open(completed_json_file) as input_file:
        command_content = input_file.read()
        command_json = json.loads(command_content)
        target_command_list = []
        for command in command_json :
            file: str = command['file']
            if any((file.startswith(repository) for repository in repositories)):
                target_command_list.append(command)

        with open(simplified_json_file, 'w') as output_file:
            output_file.write(json.dumps(target_command_list, indent=4))


if  __name__ == '__main__':
    if len(sys.argv) != 4:
        print('Usage: python3 {} <complete.json> <simply.json> <repo[,repo[,repo]...]>'.format(sys.argv[0]))
        print('Eg: python3 {} ./compile_commands.json.big ./compile_commands.json system,hardware,frameworks'.format(sys.argv[0]))
        exit (1)
    else:
        input_compile_commands = sys.argv [1]
        output_compile_commands =  sys.argv[2]
        repositories = sys.argv[3].split(',')
        simplify_compile_commands_json(input_compile_commands, output_compile_commands, repositories)
```

将全量 `compile_commands.json` 改名为 `compile_commands.full.json` 并放到脚本目录下，运行脚本：

```bash
python3 simplify_compile_commands_json.py compile_commands.full.json compile_commands.json 'vendor/honor/system/base/frameworks/XXX'
```

得到精简后的 `compile_commands.json`，理论上这个文件可以放在任何地方，但是推荐将其放在项目目录的 `.vscode` 目录下。

再在 `.vscode` 目录下创建 `settings.json`，添加 `clangd.arguments` 的 `--compile-commands-dir` 项：

```json
{
    // ... 其他参数

    "clangd.arguments": [
        "--compile-commands-dir=${workspaceFolder}/.vscode",
    ],

    // ... 其他参数
}
```

重新加载 vscode 窗口，等待 clangd 索引完毕，即可进行跳转和补全。

#### 无法得到 `compile_commands.json` 文件时

在某些情况下实在无法得到 `compile_commands.json` 文件时，可以设置 `clangd.fallbackFlags` 项中的 `-I` 参数，添加项目的包含路径，也可以进行索引。

```json
{
    //...其他参数
    "clangd.arguments": [
    ],
    "clangd.fallbackFlags": [
        "--target=aarch64-linux-android", // 指定平台，可以设置 aarch64 宏
        "-std=c++20",
        "-I${workspaceFolder}/include",
        "-I<其他包含目录>"
    ]
}
```

这种方式不如使用 `compile_commands.json` 文件精确。