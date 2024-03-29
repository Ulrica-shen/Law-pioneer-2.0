# 介绍

Swift是一个提供LLM模型轻量级训练和推理的开源框架。Swift提供的主要能力是`efficient tuners`，tuners是运行时动态加载到模型上的额外结构，在训练时将原模型的参数冻结，只训练tuner部分，这样可以达到快速训练、降低显存使用的目的。比如，最常用的tuner是LoRA。

总之，在这个框架中提供了以下特性：

- **具备SOTA特性的Efficient Tuners**：用于结合大模型实现轻量级（在商业级显卡上）训练和推理，并取得较好效果
- **使用ModelScope Hub的Trainer**：基于`transformers trainer`提供，支持LLM模型的训练，并支持将训练后的模型上传到[ModelScope Hub](https://www.modelscope.cn/models)中
- **可运行的模型Examples**：针对热门大模型提供的训练脚本和推理脚本，并针对热门开源数据集提供了预处理逻辑，可直接运行使用

# 快速开始

在本章节会介绍如何快速安装swift并设定好运行环境，并跑通一个用例。

安装swift的方式非常简单，用户只需要在python>=3.8环境中运行：

```shell
pip install ms-swift
```

下面的代码使用LoRA在分类任务上训练了`bert-base-uncased`模型：

**运行下面的代码前请额外安装modelscope: **

```shell
pip install modelscope>=1.9.0
```

```python
import os
os.environ['CUDA_VISIBLE_DEVICES'] = '0'

from modelscope import AutoModelForSequenceClassification, AutoTokenizer, MsDataset
from transformers import default_data_collator

from swift import Trainer, LoRAConfig, Swift, TrainingArguments


model = AutoModelForSequenceClassification.from_pretrained(
            'AI-ModelScope/bert-base-uncased', revision='v1.0.0')
tokenizer = AutoTokenizer.from_pretrained(
    'AI-ModelScope/bert-base-uncased', revision='v1.0.0')
lora_config = LoRAConfig(target_modules=['query', 'key', 'value'])
model = Swift.prepare_model(model, config=lora_config)

train_dataset = MsDataset.load('clue', subset_name='afqmc', split='train').to_hf_dataset().select(range(100))
val_dataset = MsDataset.load('clue', subset_name='afqmc', split='validation').to_hf_dataset().select(range(100))


def tokenize_function(examples):
    return tokenizer(examples["sentence1"], examples["sentence2"],
    padding="max_length", truncation=True, max_length=128)


train_dataset = train_dataset.map(tokenize_function)
val_dataset = val_dataset.map(tokenize_function)

arguments = TrainingArguments(
    output_dir='./outputs',
    per_device_train_batch_size=16,
)

trainer = Trainer(model, arguments, train_dataset=train_dataset,
                    eval_dataset=val_dataset,
                    data_collator=default_data_collator,)

trainer.train()
```

在上面的例子中，我们使用了`bert-base-uncased`作为基模型，将LoRA模块patch到了['query', 'key', 'value']三个Linear上，进行了一次训练。

训练结束后可以看到outputs文件夹，它的文件结构如下：

> outputs
>
> ​    |-- checkpoint-xx
>
> ​                    |-- configuration.json
>
> ​                    |-- default
>
> ​                              |-- adapter_config.json
>
> ​                              |-- adapter_model.bin
>
> ​                    |-- ...

可以使用该文件夹执行推理：

```python
from modelscope import AutoModelForSequenceClassification, AutoTokenizer
from swift import Trainer, LoRAConfig, Swift


model = AutoModelForSequenceClassification.from_pretrained(
            'AI-ModelScope/bert-base-uncased', revision='v1.0.0')
tokenizer = AutoTokenizer.from_pretrained(
    'AI-ModelScope/bert-base-uncased', revision='v1.0.0')
lora_config = LoRAConfig(target_modules=['query', 'key', 'value'])
model = Swift.from_pretrained(model, model_id='./outputs/checkpoint-21')

print(model(**tokenizer('this is a test', return_tensors='pt')))
```
