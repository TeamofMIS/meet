---
layout: post
title: Sacred -- A Python Library for Machine Learning Experiment Management
description: This is post by Yuxuan Yang
author: Yuxuan Yang
date: 2022-06-16 22:15:00
hero_image: https://images.unsplash.com/photo-1474377207190-a7d8b3334068?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1740&q=80
hero_height: is-large
hero_darken: true
image: https://images.unsplash.com/photo-1474377207190-a7d8b3334068?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1740&q=80
tags: resource jekyll docs
---
This is post by Yuxuan Yang

# Sacred：Python机器学习实验管理库

- [Sacred：Python机器学习实验管理库](#sacredpython机器学习实验管理库)
  - [参考](#参考)
  - [例子](#例子)
  - [运行](#运行)
  - [配置](#配置)
  - [命令行接口](#命令行接口)
    - [参数更新](#参数更新)
    - [命令](#命令)
  - [捕获函数](#捕获函数)
  - [观察器](#观察器)
  - [Run对象](#run对象)
    - [Metrics](#metrics)
    - [资源和工件](#资源和工件)
  - [已知问题](#已知问题)

## 参考

- [sacred doc](https://sacred.readthedocs.io/en/stable/index.html)
- [Sacred 教程 (A Tutorial on Sacred)](https://www.jarvis73.com/2020/11/15/Sacred/)

## 例子

```python
"""A standard machine learning task using sacred's magic."""
from sacred import Experiment
from sacred.observers import FileStorageObserver
from sklearn import svm, datasets, model_selection

ex = Experiment("svm")

ex.observers.append(FileStorageObserver("my_runs"))


@ex.config  # Configuration is defined through local variables.
def cfg():
    C = 1.0
    gamma = 0.7
    kernel = "rbf"
    seed = 42


@ex.capture
def get_model(C, gamma, kernel):
    return svm.SVC(C=C, kernel=kernel, gamma=gamma)


@ex.automain  # Using automain to enable command line integration.
def run():
    X, y = datasets.load_breast_cancer(return_X_y=True)
    X_train, X_test, y_train, y_test = model_selection.train_test_split(
        X, y, test_size=0.2
    )
    clf = get_model()  # Parameters are injected automatically.
    clf.fit(X_train, y_train)
    return clf.score(X_test, y_test)
```

常用的：`@ex.automain`、`@ex.config`、`@ex.capture`、观察器、Metrics。

## 运行

1. `@ex.main`搭配`ex.run()`或`ex.run_commandline()`

    ```python
    from sacred import Experiment
    ex = Experiment('run demo')

    @ex.main
    def main():
        print('Main executed.')

    # or use ex.run() when no command is specified
    ex.run_commandline()
    ```

2. `@ex.automain`（**常用**）
	
   ```python
   from sacred import Experiment
   ex = Experiment('run demo')
   
   @ex.automain
   def main():
       print('Main executed.')
   ```

- 使用`@ex.main`注解时需要显式调用`ex.run()`或`ex.run_commandline()`。

- 使用`@ex.automain`注解时，相当于`@ex.main`+`ex.run_commandline()`。但要注意：**带有 `@ex.automain`装饰器的函数必须放到脚本文件的末尾, 否则该函数后面的代码在运行时会是未定义的。**

## 配置

`@ex.config`

```python
from sacred import Experiment
import numpy as np
ex = Experiment('cfg demo')

@ex.config
def cfg():
    a = 1
    b = 2.3
    c = np.array([4,5,6])

@ex.automain
def main(a, b, c):
    result = (c + a) * b
    print(result)
```

若是分配置文件和运行文件：

`cfg_demo_dfg.py`（配置文件）：

```python
from sacred import Experiment
import numpy as np
ex = Experiment('cfg demo')

@ex.config
def cfg():
    a = 1
    b = 2.3
    c = np.array([4,5,6])
```

`cfg_demo_run.py`（运行文件）：

```python
from cfg_demo_cfg import ex

@ex.automain
def main(a, b, c):
    result = (c + a) * b
    print(result)
```

- 主函数命名无所谓，但参数列表要齐全。使用的变量都要列在参数列表中，配置中的变量可以自动注入。
- 配置函数中支持任何Python语法，可以使用分支结构等来动态控制参数。参数类型支持整型, 浮点型, 字符串, 元组, 列表, 字典等可以Json序列化的类型。**但配置函数不能包含任何的return或yield语句。**

- 在代码中更新参数：

  ```python
  ex.add_config({
      'foo': 42,
      'bar': 'baz'
  })
  
  # 或者
  ex.add_config(foo=42, bar='baz')
  
  # 读取配置文件
  ex.add_config('conf.json')
  ex.add_config('conf.pickle')    # 要求配置保存为字典
  ex.add_config('conf.yaml')      # 依赖 PyYAML 库
  ```

- 参数组（`@ex.named_config`）：可以指定多个参数组，选定一个运行。

  代码：

  ```python
  @ex.named_config
  def variant1():
      a = 100
      c = "bar"
  ```

  命令行：

  ```bash
  python named_configs_demo.py with variant1       # 运行
  python named_configs_demo.py with variant1.json  # 保存配置到文件
  ```


## 命令行接口

参阅[命令行接口进阶](https://www.jarvis73.com/2020/11/15/Sacred/#23-%E5%91%BD%E4%BB%A4%E8%A1%8C%E6%8E%A5%E5%8F%A3%E8%BF%9B%E9%98%B6)，经过实际测试，与文章中的结果并不完全相同。

### 参数更新

这是命令行最主要的作用，使用`python file.py with key=value`。

```python
from sacred import Experiment
ex = Experiment('cmd demo')

@ex.config
def cfg():
    a = 1      # int
    b = 'abc'  # str

@ex.automain
def main(a, b, c):
    print(f'a: {type(a)}, {a}')
    print(f'b: {type(b)}, {b}')
    print(f'c: {type(c)}, {c}')
```

对于命令行中的`value`：

- value外层没有引号：整型和浮点型没有问题，但列表类型（无论小括号还是中括号）都报错。
- value外层有引号：**命令行会先褪去这层引号，再解析，以该方式可以传入列表、字典等类型。**
- 解析得到的类型与原配置变量的类型无关。例如上例中`a`为整型，`b`为字符串，使用`with a=value`和`with b=value`，两个同样的value解析出的类型是一样的，而不会随变量`a`和`b`的类型而改变。

使用命令行注入变量`c`，以下是本人尝试所得到的结果，**环境为Ubuntu 20.04系统、zsh shell**。

```bash
python cmd_demo.py with c=1          # int
python cmd_demo.py with c='1'        # int
python cmd_demo.py with c="'1'"      # str
python cmd_demo.py with c=[1,2,3]    # 报错
python cmd_demo.py with c='[1,2,3]'  # sacred内置只读列表类型
python cmd_demo.py with c='["hello", "world"]'  # 只读列表['hello', 'world']
python cmd_demo.py with c='[hello, world]'      # 字符串'[hello, world]'
```

因此为了准确传参，可以遵循：**在最外层加一层引号，引号内按照Python写法来写**，这样可以保证正确地解析为Python内置类型。

### 命令

1. 内置命令：

   ```bash
   # 内置命令
   python demo.py print_config
   python demo.py print_config with a=1
   ```

   | 命令                  | 参数                                                         |
   | --------------------- | ------------------------------------------------------------ |
   | `print_config`        | 仅打印参数. 对于同时更新了的参数, 会使用三种颜色来标记: 更改的(蓝色), 增加的(绿色), 类型改变的(红色) |
   | `print_dependencies`  | 打印程序依赖, 源文件, git 版本控制                           |
   | `save_config`         | 保存当前参数到文件, 默认保存到 `config.json`                 |
   | `print_named_configs` | 打印`@ex.named_config`修饰的参数组                           |

2. 自定义命令：

   使用`@ex.command`：

   ```python
   @ex.command
   def train(_run, _config):
       """
       Training a deep neural network.
       """
       pass
   ```

   运行：

   ```bash
   python demo.py train
   ```

   注意：**在代码中仅定义`@ex.command`函数是无法运行的，依然需要定义主函数（即`@ex.automain`）。**

## 捕获函数

`@ex.capture`

配置域中的参数可以直接注入到捕获函数的参数列表中。使用捕获器，可以使得不必将所有参数都注入到主函数的参数列表中，配置参数可灵活调用。

捕获函数包括：

- `@ex.main`
- `@ex.automain`
- `@ex.capture`
- `@ex.command`

```python
from sacred import Experiment
ex = Experiment('cmd demo')

@ex.config
def cfg():
    a = 1      # int
    b = 'abc'  # str
    c = 1.2    # float

@ex.capture
def print_param(a, b, c=2.3):
    print(f'a={a}, b={b}, c={c}')

@ex.automain
def main():
    print_param()             # a=1, b=abc, c=1.2
    print_param(2, 'def')     # a=2, b=def, c=1.2
    print_param(b=2, c=10.5)  # a=1, b=2, c=10.5
```

注意：`c=2.3`这个默认参数永远不会被使用，参数优先级顺序: **调用时传参 > Sacred 参数 > 默认参数**。

捕获函数可以获取一些 Sacred 内置的变量，要使用时写在捕获函数的参数列表中就可以直接调用:

- `_config` : 所有的参数作为一个字典(只读的)

- `_seed` : 一个随机种子

- `_rnd` : 一个随机状态

- `_log` : 一个 logger，是一个Python标准Logger，可以使用`debug`, `info`, `warning`, `error`, `critical` ，其他可参考[文档](https://docs.python.org/2/library/logging.html#logger-objects)。 

  ```python
  @ex.capture
  def some_function(_log):
      _log.warning('My warning message!')
  ```

  

- `_run` : 当前实验运行时的 run 对象

## 观察器

参考[Observing an Experiment](https://sacred.readthedocs.io/en/stable/observers.html)。

- 文件存储

  通过代码：

  ```python
  from sacred.observers import FileStorageObserver
  
  ex.observers.append(FileStorageObserver('my_runs_demo'))
  ```

  通过命令行：

  ```bash
  >> ./my_experiment.py -F 'my_runs_demo'
  >> ./my_experiment.py --file_storage='my_runs_demo'
  ```

  在`my_runs_demo`文件夹下保存了以下内容：

  - 源代码
  - 配置参数
  - 输出
  - 追踪的metric（如何使用在下文Run对象中会提到）
  - 运行详细配置，包括环境、依赖库等

## Run对象

Run对象可以通过捕获函数（包括`@ex.main`、`@ex.automain`、`@ex.capture`、`@ex.command`）接收的`_run`参数获得。`run = ex.run()`完成之后返回的`run`变量也是Run对象。

### Metrics

使用`_run.log_scalar()`或者`ex.log_scalar()`，记录如损失、精度等信息。

```python
from sacred import Experiment
from sacred.observers import FileStorageObserver

ex = Experiment('metrics demo')
ex.observers.append(FileStorageObserver('my_runs_demo'))

@ex.config
def cfg():
    a = list(range(10))

@ex.capture
def linear(a, _run):
    for i in a:
        _run.log_scalar('data.x', i)     # step从0开始自动递增
        _run.log_scalar('data.y', i*2)
        _run.log_scalar('data,z', i, 2)  # step=2

@ex.automain
def main():
    linear()
```

在`metrics.json`中：

```json
{
  "data,z": {
    "steps": [
      2,
      2,
      2,
      2,
      2,
      2,
      2,
      2,
      2,
      2
    ],
    "timestamps": [
      "2021-05-07T08:51:51.362334",
      "2021-05-07T08:51:51.362346",
      "2021-05-07T08:51:51.362355",
      "2021-05-07T08:51:51.362364",
      "2021-05-07T08:51:51.362373",
      "2021-05-07T08:51:51.362401",
      "2021-05-07T08:51:51.362410",
      "2021-05-07T08:51:51.362419",
      "2021-05-07T08:51:51.362428",
      "2021-05-07T08:51:51.362437"
    ],
    "values": [
      0,
      1,
      2,
      3,
      4,
      5,
      6,
      7,
      8,
      9
    ]
  },
......
```

### 资源和工件

资源（resource）在sacred中指实验中需要用到的文件。使用`_run.open_resource(filename, mode='r')`或者`ex.open_resource(filename, mode='r')`，观察器会记录相关信息，并返回打开的文件对象。也可以简单地`_run.add_resource(filename)`或者`ex.add_resource(filename)`，只让观察器记录下来，但没有返回值。

工件（artifact）在sacred中指实验中产生的文件。使用`_run.add_artifact(filename)`或者`ex.add_artifact(filename)`以提供给观察器记录。

```python
from sacred import Experiment
from sacred.observers import FileStorageObserver
import numpy as np

ex = Experiment('resource demo')
ex.observers.append(FileStorageObserver('my_runs_demo'))

@ex.config
def cfg():
    input_name = 'input.txt'
    output_name = 'output.npz'

@ex.capture
def to_npz(input_name, output_name, _run):
    array = []
    with _run.open_resource(input_name) as f:    # 代替open()函数
        for line in f.readlines():
            array.append(list(map(float, line.strip().split(' '))))
    array = np.array(array)
    np.savez(output_name, array)
    _run.add_artifact(output_name)               # 添加一个输出记录


@ex.automain
def main():
    to_npz()
```

在`run.json`中记录了相应的信息：

```json
{
    "artifacts": [
    "output.npz"
  ],
    ......
    "resources": [
    [
      "/home/yyx/Codebase/python/sacred\u5b66\u4e60/input.txt",
      "my_runs_demo/_resources/input_c73e1b76f6fff783ec65fdf3f5aa8aea.txt"
    ]
  ],
}
```

在`my_runs_demo`文件夹下留存有资源文件和工件文件的副本。在官方文档中介绍了对于Mongo数据库观察器的话，不会保存副本，而是路径、md5等信息。

## 已知问题

1. 在PyCharm使用ssh远程连接服务器进行运行和调试时，[参数更新](#参数更新)的命令行环境取决于本地系统，Windows和Linux下的表现不同，例如在Windows环境下：

    ```cmd
    python main.py with gpus='[1,2,3,4]'  # str: '[1,2,3,4]'
    python main.py with gpus=[1,2,3,4]  # list: [1, 2, 3, 4]
    ```

    建议在运行前使用`print_config`命令查看，或者运行的开始可能有相关warning。

    ```cmd
    WARNING - root - Changed type of config entry "gpus" from list to str
    ```

2. @ex.config函数中的操作并不能像文档中说的那么随意，例如：

   ```python
   video_indices = random.sample(list(range(45)), 40)
   train_index = random.sample(video_indices, 35)
   val_index = [i for i in video_indices if i not in train_index]  # Error: name "train_index" not found
   ```

   上述代码意在取`video_indices`和`train_index`的差集，可以用一行代码解决，但由于sacred的问题会报错。改为下列使用for循环的代码则不会出问题：

   ```python
   video_indices = random.sample(list(range(45)), 40)
       train_index = random.sample(video_indices, 35)
       val_index = []
       for i in video_indices:
           if i not in train_index:
               val_index.append(i)
   ```

   **目前只发现对于列表对象会出问题。**