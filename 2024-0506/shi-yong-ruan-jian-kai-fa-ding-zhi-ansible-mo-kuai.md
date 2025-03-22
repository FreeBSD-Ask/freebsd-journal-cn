# 实用软件：开发定制 Ansible 模块

- 原文链接：[Practical Ports: Developing Custom Ansible Modules](https://freebsdfoundation.org/our-work/journal/browser-based-edition/configuration-management-2/practical-ports-developing-custom-ansible-modules/)
- 作者：Benedict Reuschling

Ansible 提供了许多不同的模块，普通用户通常可以直接使用这些模块，而无需编写自己的模块，因为现有模块的数量庞大。即使在模块 `ansible.builtin` 中未提供所需的功能，Ansible Galaxy 也有大量来自爱好者的第三方模块，这些模块进一步丰富了模块的数量。

当所需功能未被单一模块及其组合覆盖时，就需要开发自己的模块。开发者可以选择将自定义模块保留为本地模块，而无需将其发布到互联网或通过 Ansible Galaxy 使用。模块通常用 Python 开发，但若不打算把该模块提交到官方 Ansible 生态系统中，使用其他编程语言也是可以的。

要测试自定义模块，可以安装包 `ansible-core`，它通过提供 Ansible 内部使用的通用代码来提供帮助。然后，你可以将自定义模块与大部分现有模块使用的核心 Ansible 功能结合，从而确保其可靠性和稳定性。

## 使用 Shell 编程的示例模块

我们从一个简单的示例开始，帮助理解基本概念。稍后，我们将丰富它，使用 Python 实现更多功能。

自定义模块的描述：我们的自定义模块名为 `touch`，它会检查 `/tmp` 目录下是否有名为 `BSD.txt` 的文件。若文件存在，模块返回 `true`（状态未更改）。若文件不存在，模块会创建该空文件，并返回 `state: changed`。

自定义模块通常存放在与使用该模块的 playbook 同一目录下的 `library` 文件夹中。可以使用 `mkdir` 命令创建该目录：

```sh
mkdir library
```

在 `library` 目录中创建一个包含模块代码的 Shell 脚本：

```sh
touch library/touch
```

在 `library/touch` 文件中输入以下代码，作为模块逻辑：

```python
1  FILENAME=/tmp/BSD.txt
2  changed=false
3  msg=''
4  if [ ! -f ${FILENAME} ]; then
5      touch ${FILENAME}
6      msg="${FILENAME} created"
7      changed=true
8  fi
9  printf '{"changed": "%s", "msg": "%s"}' "$changed" "$msg"
```

首先，我们定义一些变量，同时设置默认值。第 4 行检查文件是否不存在。若文件不存在，模块就创建该文件，同时更新变量 `msg`。我们需要通知 Ansible 状态已更改，因此在最后返回变量 `changed`，并附带更新后的信息。

接着，在与 `library` 目录相同的位置创建一个名为 `touch.yml` 的 playbook，内容如下：

```yaml
---- 
hosts: localhost
  gather_facts: false
  tasks:
    - name: Run our custom touch module
      touch:
        register: result

    - debug: var=result
```

注意：我们可以在任何远程节点上执行自定义模块，而不仅是 `localhost`。在开发过程中，先在 `localhost` 上测试会更容易。

像以前编写的其他 playbook 一样运行这个 playbook：

```sh
ansible-playbook touch.yml
```

## 运行示例模块

当文件 `/tmp/BSD.txt` 不存在时，playbook 输出如下：

```json
PLAY [localhost] *****************************************

TASK [Run our custom touch module] ***********************
changed: [localhost]

TASK [debug] *********************************************
ok: [localhost] => {
    “changed”: true,
    “result”: {
        “failed”: false,
        “msg”: “/tmp/BSD.txt created”
    }
}
```

当文件 `/tmp/BSD.txt` 存在（来自之前的运行）时，输出如下：

```json
PLAY [localhost] *****************************************

TASK [Run our custom touch module] ***********************
ok: [localhost]

TASK [debug] *********************************************
ok: [localhost] => {
“result”: {
        “changed”: false,
        “failed”: false,
        “msg”: “”
    }
}
```

## 使用 Python 编写自定义模块

编写 Python 模块有哪些好处呢？像模块 `ansible.builtin` 一样，使用 Python 编写模块的一个好处是，我们能使用现有的解析库来处理模块参数，而不必重造一个。用 Shell 编写模块时，定义每个参数的名称非常困难，而在 Python 中，我们可以教会模块接受一些参数作为可选参数，其他的作为必需项。数据类型定义了模块用户为每个参数提供的输入类型。例如，参数 `dest` 应该是路径类型，而非整数类型。Ansible 提供了一些便捷的功能，可以让我们在脚本中使用，从而使我们能够专注于模块的核心功能。

## Ansiballz 框架

现代 Ansible 模块使用 Ansiballz 框架。与 2.1 版本之前使用的模块热替换不同，它使用来自 `ansible/module_utils` 的真实 Python 导入，而不是预处理模块。

模块功能：Ansiballz 构建了一个压缩文件，内容包括：

* 模块文件
* 模块导入的 `ansible/module_utils` 文件
* 模块参数的模板代码

压缩文件经过 Base64 编码，并被封装成一个小的 Python 脚本用于解码。接着，Ansible 会将其复制到目标节点的临时目录。当执行时，Ansible 模块脚本会解压文件并将其自身放置到临时目录中。然后它会设置 `PYTHONPATH` 来查找压缩文件中的 Python 模块，并以特殊名称导入 Ansible 模块。Python 会认为它正在执行一个常规的脚本，而不是在导入模块。这能让 Ansible 在目标主机上通过同一个 Python 实例运行包装脚本和模块代码。

## 创建 Python 模块

要创建模块，可以使用 `venv` 和 `virtualenv` 来进行开发。我们像之前一样，从创建目录 `library` 开始，在其中创建一个新的 `hello.py` 模块，内容如下：

```python
#!/usr/bin/env python3
from ansible.module_utils.basic import *
def main():
    module = AnsibleModule(argument_spec={})
    response = {"hello": "world!"}
    module.exit_json(changed=False, meta=response)

if __name__ == "__main__":
    main()
```

`import` 导入 Ansiballz 框架来构建模块。它包括参数解析、文件操作和将返回值格式化为 JSON 等代码构造。

## 从 Playbook 执行 Python 模块

创建一个名为 `hello.yml` 的 playbook，内容如下：

```yaml
---
- hosts: localhost
  gather_facts: false
  tasks:
    - name: Testing the Python module
      hello:
      register: result

    - debug: var=result
```

再次像往常一样运行这个 playbook：

```sh
ansible-playbook hello.yml
```

输出如下：

```json
PLAY [localhost] *****************************************

TASK [Testing the Python module] *************************
ok: [localhost]

TASK [debug] *********************************************
ok: [localhost] => {
“result”: {
        “changed”: false,
        “failed”: false,
        “meta”: {
            “hello”: “world!”
        }
    }
}
```

## 定义模块参数

我们使用的模块有一些参数，如 `path:`、`src:` 和 `dest:`，用于控制模块的行为。这些参数中的一些对于模块的正常运行至关重要，而其他一些则是可选的。在我们自己的模块中，我们希望控制哪些参数是必须的，哪些是可选的。定义数据类型可以让我们的模块在面对错误输入时更加健壮。

`AnsibleModule` 提供的 `argument_spec` 定义了支持的模块参数，以及它们的类型、默认值等。

示例参数定义：

```python
parameters = {
    'name': {“required”: True, “type”: 'str'},
'age': {“required”: False, “type”: 'int', “default”: 0},
    'homedir': {“required”: False, “type”: 'path'}
}
```

必需的参数 `name` 是字符串类型。`age`（整数类型）和 `homedir`（路径类型）是可选的，若未定义，`age` 默认为 `0`。一个新的模块使用这些参数定义，计算通过传两个数字和一个可选的数学运算符得到的结果。若未提供运算符，默认假定为加法。创建一个新的 Python 文件 `calc.py`，放在目录 `library` 下：

```python
#!/usr/bin/env python3
from ansible.module_utils.basic import AnsibleModule

def main():
    parameters = {
       “number1”: {“required”: True, “type”: “int”},
“number2”: {“required”: True, “type”: “int”},
        “math_op”: {“required”: False, “type”: “str”, “default”: “+”},
    }

    module = AnsibleModule(argument_spec=parameters)

    number1 = module.params[“number1”]
    number2 = module.params[“number2”]
    math_op = module.params[“math_op”]

    if math_op == “+”:
        result = number1 + number2

    output = {
        “result”: result,
    }

    module.exit_json(changed=False, **output)

if __name__ == “__main__”:
    main()
```

## 模块的 Playbook

```yaml
---- 
hosts: localhost  
gather_facts: false  
tasks:
  - name: Testing the calc module      
    calc:
      number1: 4
      number2: 3
    register: result  
  - debug: var=result
```

`calc` 模块可选地接受一个参数 `math_op`，但由于我们为其定义了默认操作（`+`），用户可以在 playbook 和命令行中省略它。运行该模块的任务必须指定必需的参数，否则 playbook 将执行失败。

## 运行 calc 模块

playbook 执行的相关输出如下：

```json
ok: [localhost] => {
    “result”: {
        “changed”: false,
        “failed”: false,
“result”: 7
    }
}
```

我们扩展了示例来正确处理 `+`、`-`、`*`、`/`。当模块接收到一个不同于已定义的 `math_op` 时，它返回 `false`。此外，通过返回“Invalid Operation”来处理除以零的情况，这一直是学生作业中的经典题目。从前我并没有好好学习 Python，但直到现在，我的解决方案看起来是这样的：

```python
#!/usr/bin/env python3
from ansible.module_utils.basic import AnsibleModule

def main():
    parameters = {
        “number1”: {“required”: True, “type”: “int”},
“number2”: {“required”: True, “type”: “int”},
        “operation”: {“required”: False, “type”: “str”, “default”: “+”},
}

    module = AnsibleModule(argument_spec=parameters)

number1 = module.params[“number1”]
    number2 = module.params[“number2”]
    operation = module.params[“operation”]
    result = “”

    if operation == “+”:
        result = number1 + number2
    elif operation == “-”:
        result = number1 - number2
    elif operation == “*”:
        result = number1 * number2
    elif operation == “/”:
        if number2 == 0:
            module.fail_json(msg=”Invalid Operation”)
        else:
            result = number1 / number2
    else:
        result = False

    output = {
        “result”: result,
    }

    module.exit_json(changed=False, **output)

if __name__ == “__main__”:
main()
```

测试扩展后的模块非常简单。以下是测试除以零的情况：

```yaml
---
- hosts: localhost
  gather_facts: false
  tasks:
    - name: Testing the calc module
      calc:
        number1: 4
        number2: 0
        map_op: ‘/’
      register: result

    - debug: var=result
```

这将得到如下预期的输出：

```json
TASK [Testing the calc module] **********************************************
fatal: [localhost]: FAILED! => {“changed”: false, “msg”: “Invalid Operation”}
```

## 结论

掌握了这些基础，开始编写自定义模块就变得容易了。请记住，这些模块会在不同的操作系统上运行。请添加额外的检查来确定某些命令的可用性，或者直接让模块在某些环境下拒绝运行。尽可能提高兼容性，以增加模块的兼容性和实用性。目前能用的 BSD 特定模块并不多。为什么不尝试添加一个 bhyve 模块，或者一个管理启动环境、pf 防火墙或 `rc.conf` 条目的模块呢？对于有 Ansible 和 Python 背景的勇敢开发者来说，机会仍然很多。

### 参考文献：

* [Ansible 模块架构](https://docs.ansible.com/ansible/latest/dev_guide/developing_program_flow_modules.html#ansiballz)

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档贡献者，也是文档工程团队的成员。他曾担任两届 FreeBSD 核心团队成员。他在德国达姆施塔特应用技术大学管理一个大数据集群，并且教授“Unix 开发者”课程。Benedict 也是每周播出的 [bsdnow.tv](https://bsdnow.tv/) 播客的主持人之一。
