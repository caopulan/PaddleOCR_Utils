# 简介
本仓库是基于PaddleOCR写的一些小工具、修改。在使用的时候需要先正确配置好PaddleOCR相关环境及代码。

首先，很感谢PaddleOCR项目组，提供一个非常好的OCR开源工具，能够非常方便的进行训练、推理等。

在使用的过程中，会遇到一些小问题及需求，所以本仓库整理发布了我在使用过程中用到的一些自己编写的代码，希望能帮助到其他在使用PaddleOCR的朋友。代码和模块设计会尽量负责PaddleOCR规范。因为目前是在训练识别模型，所以更改基本也是基于识别模块，同时我会给出对于其他模块修改的建议，但是由于没有实践过所以不直接提供代码。

如果有一些别的工具性的需要，尤其是rec模块，可以提交issue。有一些写好了的可以直接提pr。

本仓库中有部分修改/工具会接下来对PaddleOCR提PR，如果被merge了我会在这个Readme中说明。

## 工具箱概览
计划中/待整理有部分是我已经对自己的代码进行了修改，所以会较快发布。

已发布
- Eval结果保存
- 数据集乱序修正（计划提交pr）

计划中/待整理
- infer封装成函数接口
- 基于inference模型端到端评估
- 训练过程保存global_step来方便断点训练（计划提交pr）
- 对DB相关参数搜索（训练出的识别模型对DB相关参数非常敏感，通过自动对inference模型推理得到端到端指标来搜索合适的DB参数）
## 使用方法
按照步骤将我的文件或代码替换PaddleOCR原本的代码即可。但是需要注意，在替换文件前一定要先对两个文件进行compare，你的PaddleOCR和我的版本不一定一样，所以先比较避免修改到其他代码。

也可以不直接修改PaddleOCR代码。对于大部分代码，在需要修改的目录下新建一个文件，把我的文件复制过去，再修改同目录__init__.py和support_dict来确保能正确导入。

有部分操作比较麻烦的过程不提供增添新代码的使用方法（即仅提供修改PaddleOCR文件的方法），如果有需要可以自己查看代码逻辑来调整使用方式。

以数据集乱序为例：
- 使用方法①：直接替换文件
  - 用```ppocr/data/simple_dataset_keepseq.py```文件替换PaddleOCR```ppocr/data/simple_dataset.py```文件
  - 用```ppocr/data/__init__.py```文件替换PaddleOCR```ppocr/data/__init__.py.py```文件
- 使用方法②：增添新文件
    - 将```ppocr/data/simple_dataset_keepseq.py```文件复制到PaddleOCR```ppocr/data/```目录下
    - 用```ppocr/data/__init__.py```替换PaddleOCR```ppocr/data/__init__.py```，或手动修改（①增加import；②support_dict加入'SimpleDataSet_Seq'）
    - 修改config文件```Train```，```Eval```中```dataset/name: SimpleDataSet_Seq```（通常```Train```需要乱序，所以对eval结果有要求的话仅需设置```Eval```下```dataset```。

显而易见，直接替换文件更加方便快捷，增添新文件相对麻烦，但是更加稳定，能够避免因本仓库的PaddleOCR版本和你的不一致而导致文件在其他地方出现的问题。


# 工具箱
## 保存Eval结果
每次Eval的时候保存所有的预测结果，方便对模型进行分析。

### 效果
模型会在Eval的时候，存储Eval的结果到```eval_result/{logger.title}```文件中。每次Eval的时候会生成一个txt文本，文件名包含```epoch```,```step```,```time```,```accurate```，方便查询分析。

### 使用方法
- 用```ppocr/metrics/rec_metric.py```替换PaddleOCR中```ppocr/metrics/rec_metric.py```
- ```tools/program.py```添加 ```import datetime```
- 修改```tools/program.py```中```train```函数:
```python
cur_metric = eval(
                  model,
                  valid_dataloader,
                  post_process_class,
                  eval_class,
                  use_srn=use_srn)
```
后添加代码：
```python
try:
    output = cur_metric.pop('output')
    acc = cur_metric['acc']
    title = config['Eval']['logger']['title']
    if not os.path.exists(f'./eval_result/{title}/'):
        os.makedirs(f'./eval_result/{title}/')
    output_dir = f"./eval_result/{title}/epoch_{epoch}_step_{global_step}_{datetime.datetime.now().strftime('%Y%m%d%H%M%S')}_acc_{acc}.txt"
    with open(output_dir, "w", encoding="utf-8") as fp:
        fp.write(output)
except:
    pass
```
- 修改config：```Eval```添加字段```logger/title```，添加后如下：
```angular2html
loader:
    shuffle: False
    drop_last: False
    batch_size_per_card: 256
    num_workers: 2
logger:
    title: rec_model

```

### 其他
- 如果需要保存检测/方向分类模型Eval结果，可以修改相应的metric文件，返回的字典增加```output```来保存Eval过程
- 可以修改```ppocr/metrics/rec_metric.py```中```self.output```相关代码来返回不同格式。
## 数据集乱序
Dataset模块在对数据取样的时候用的 
``` python
lines = random.sample(lines, round(len(lines) * ratio_list[idx]))
```
命令，会导致即使ratio设置为1也会发生数据乱序。在保存Eval结果的时候，会出现每次Eval得到的结果乱序。

### 代码变更
```python
if int(ratio_list[idx]) != 1:
    lines = random.sample(lines, round(len(lines) * ratio_list[idx]))
```
仅当```ratio```设置不为1的时候进行采样。

### 使用方法
使用方法1：
- 将```ppocr/data/simple_dataset_keepseq.py```文件复制到PaddleOCR```ppocr/data/```目录下
- 用```ppocr/data/__init__.py```替换PaddleOCR```ppocr/data/__init__.py```，或手动修改（①增加import；②support_dict加入'SimpleDataSet_Seq'）
- 修改config文件```Train```，```Eval```中```dataset/name: SimpleDataSet_Seq```（通常```train```设置的乱序，所以对eval结果有要求的话仅需设置Eval下dataset。

使用方法2：
- 用```ppocr/data/simple_dataset_keepseq.py```文件替换PaddleOCR```ppocr/data/simple_dataset.py```文件（仅修改sample处）
