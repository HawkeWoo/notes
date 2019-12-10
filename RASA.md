# RASA

`requirements.txt`

```reStructuredText
--extra-index-url https://pypi.rasa.com/simple
rasa-x==0.22.2
jieba==0.39
```

## Demo

### 安装rasa

```shell
$ mkdir rasa-demo
$ cd rasa-demo
$ virtualenv venv			# requires Python 3.6.0 or higher
$ source venv/bin/activate
$ pip install rasa-x --extra-index-url https://pypi.rasa.com/simple
```

### 初始化项目

```shell
$ rasa init --no-prompt
```

```shell
$ tree rasa-demo -L 2
rasa-demo
├── __init__.py
├── actions.py		# 自定义 actions 的代码文件
├── config.yml		# Rasa NLU 和 Rasa Core 的配置文件
├── credentials.yml	# 定义和其他服务连接的一些细节，例如rasa api接口
├── data
│   ├── nlu.md		# Rasa NLU 的训练数据
│   └── stories.md	# Rasa stories 数据
├── domain.yml		# Rasa domain 文件
├── endpoints.yml	# 和外部消息服务对接的 endpoins 细则，例如 fb messenger
├── models				# 初始训练的模型
│   └── 20191031-115109.tar.gz
│   └── nlu-20191031-120620.tar.gz
└── venv
    ├── LICENSE.md
    ├── bin
    ├── include
    ├── lib
    └── share
```

### 训练模型

```shell
$ rasa train nlu
```

该命令会寻找data/目录下的数据文件来训练模型，并且训练好的模型会保存在models/目录下，模型以nlu为命名前缀。

### 启动NLU服务

```shell
$ rasa shell nlu
```

该命令可以让用户通过命令行的方式去验证模型的效果，默认启用最新的训练模型

也可以指定模型

```shell
$ rasa shell -m models/nlu-20191031-120620.tar.gz
```

甚至可以启动一个rasa的http服务

```shell
$ rasa run --enable-api -m models/nlu-20191031-120620.tar.gz
$ curl localhost:5005/model/parse -d '{"text":"hello"}'
```

---

![](https://aqumon-pic-repo.oss-cn-shenzhen.aliyuncs.com/img20191122103630.png)

## config

```yaml
language: zh
pipeline:
- name: JiebaTokenizer
- name: RegexFeaturizer
- name: CRFEntityExtractor
- name: EntitySynonymMapper
- name: CountVectorsFeaturizer
- name: CountVectorsFeaturizer
  analyzer: char_wb
  min_ngram: 1
  max_ngram: 4
- name: EmbeddingIntentClassifier
policies:
- name: MemoizationPolicy
- name: EmbeddingPolicy
- name: MappingPolicy
- name: FallbackPolicy
  nlu_threshold: 0.3
  core_threshold: 0.3
  ambiguity_threshold: 0.15
  fallback_action_name: action_default_fallback
similar_nlu_threshold: 0.1
similar_intents: 3

```

---

## NLU

NLU是一个开源的用于意图分类，响应检索和实体抽取的自然语言处理工具，是rasa框架的一个模块，也可以单独使用。

比如由`"I am looking for a Chinese restaurant in the center of town"`可以获得结构化数据如下：

```json
{
  "intent": "search_restaurant",
  "entities": {
    "cuisine" : "Chinese",
    "location" : "center"
  }
}
```

### pipeline

#### supervised_embeddings

`supervised_embeddings`的好处在于词向量是通过用户来定制的。同一个单词在不同场景下的意思可能是不一样的，比如说在英文中balance和symmetry意思相似，但是和cash意思就大不相同，但是在金融领域，balance和cash就意思就比较相似，因此你的模型需要捕捉到这种特性。supervised_embeddings不使用任何特定语言的模型，只要语言可以被分词它就可以使用。

```yaml
language: "en"

pipeline: "supervised_embeddings"
```

`supervised_embeddings` pipeline支持任何一个可以被分词的语言，在以前的版本中它是`tensorflow_embedding`，其默认使用空格来分词。用户可以自定义其组件，下面的配置是`supervised_embeddings`的默认组件，上下两个配置效果是等同的。

```yaml
language: "en"

pipeline:
- name: "WhitespaceTokenizer"
- name: "RegexFeaturizer"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
- name: "CountVectorsFeaturizer"
- name: "CountVectorsFeaturizer"
  analyzer: "char_wb"
  min_ngram: 1
  max_ngram: 4
- name: "EmbeddingIntentClassifier"
```

`WhitespaceTokenizer`是根据空格来分词，可以更换成`JiebaTokenizer`或者其它，如果使用`JiebaTokenizer`，需要安装`jieba`：`pip install jieba`。

`JiebaTokenizer`可以添加自定义词典补充：

```yaml
pipeline:
- name: "JiebaTokenizer"
  dictionary_path: "path/to/custom/dictionary/dir"
```

上述配置文件使用了两个`CountVectorsFeaturizer`实例。第一个是基于单词特征化文本，第二个是基于N-Gram模型（N-Gram是基于一个假设：第n个词出现与前n-1个词相关，而与其他任何词不相关）来特征化文本。根据经验第二个`CountVectorsFeaturizer`会更有效，但是为了保持特征化的鲁棒性保留了第一个featurizer。

当一句话有[多个意图](https://blog.rasa.com/how-to-handle-multiple-intents-per-input-using-rasa-nlu-tensorflow-pipeline/?_ga=2.215280596.218036060.1572323030-1217531547.1571801337)的时候，用户可以选择使用组合意图

```yaml
pipeline:
- name: "CountVectorsFeaturizer"
- name: "EmbeddingIntentClassifier"
  intent_tokenization_flag: true
  intent_split_symbol: "+"
```

data/nlu.md

```markdown
## intent: meetup
- I am new to the area. What meetups I could join in Berlin? 
- I have just moved to Berlin. Can you suggest any cool meetups for me?

## intent: affirm+ask_transport
- Yes. How do I get there?
- Sounds good. Do you know how I could get there from home?
```

#### pretrained_embeddings_spacy

`pretrained_embeddings_spacy`适用的场景是较少的样本，比如少于1000个的时候。`pretrained_embeddings_spacy`使用了通过`GloVe`或者`fastText`预先训练好的词向量。

例如有一个句子：`I want to buy apples`，这个句子被要求判断其意图为`get pears`，rasa模型就需要判断`apples`和`pears`很相似。这个在训练样本很少的时候非常有用。

`pretrained_embeddings_spacy`使用了spacy，目前似乎还不支持中文。

```yaml
language: "en"

pipeline: "pretrained_embeddings_spacy"
```

其具体组件如下：

```yaml
language: "en"

pipeline:
- name: "SpacyNLP"
- name: "SpacyTokenizer"
- name: "SpacyFeaturizer"
- name: "RegexFeaturizer"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
- name: "SklearnIntentClassifier"
```

#### MITIE

用户还可以选择使用`MITIE` pipeline，但是MITIE只在小的数据集中性能才比较好，如果训练集有几百个以上，训练模型会非常耗时。因此`MITIE`不被推荐，并且很可能在未来会被分离出去。

如果要使用`MITIE`pipeline， 用户必须先从语料库训练词特征向量。

```yaml
language: "en"

pipeline:
- name: "MitieNLP"
  model: "data/total_word_feature_extractor.dat"		
- name: "MitieTokenizer"
- name: "MitieEntityExtractor"
- name: "EntitySynonymMapper"
- name: "RegexFeaturizer"
- name: "MitieFeaturizer"
- name: "SklearnIntentClassifier"
```

另一个版本的`MITIE`pipeline使用了MitieIntentClassifier。`MITIE`模型训练会很慢，不推荐在比较大的训练集场景中使用。

```yaml
language: "en"

pipeline:
- name: "MitieNLP"
  model: "data/total_word_feature_extractor.dat"
- name: "MitieTokenizer"
- name: "MitieEntityExtractor"
- name: "EntitySynonymMapper"
- name: "RegexFeaturizer"
- name: "MitieIntentClassifier"
```

#### Custom pipelines

用户可以使用任何想使用的组件来构建pipelines，详情见[components](https://rasa.com/docs/rasa/nlu/components/)。

### nlu.md配置

Data/nlu.md是作为训练nlu模型的训练集，其格式如下：

```markdown
## intent:意图名1
- 语句1
- 语句2
- 语句3

## intent:意图名2
- 语句4
- 语句5
- 语句6
```

通常我们推荐每个意图至少有三个句子，例子如下：

```markdown
## intent:sys_hi
- 嗨
- 你好
- 您好
- 哈喽
- hi

## intent:什么是T日
- T日是什么
- 什么是T日
- T日
```

---

## CORE

### [Stories](https://rasa.com/docs/rasa/core/stories/)

rasa中stories是用来编辑对话故事的。这里的故事指的是用户和机器人对话的一轮或多轮流程，有点类似于你编写了用户和机器人之间对话的剧本。其中用户输入表示某个意图的语句，机器人则返回相应动作的响应。

stories.md格式

```markdown
## story_name
* intent_name
  - action_name
```

一个简单的例子如下

```markdown
## story_hi
* sys_hi
  - utter_hi
* sys_bye
  - utter_bye

## story_为什么要做风险测评
* 为什么要做风险测评
  - utter_为什么要做风险测评
```

### [Domains](https://rasa.com/docs/rasa/core/domains/)

domain.yml 声明了机器人所需要知道的所有intents, entities, slots, actions,templates。一般来说一个简单的domain.yml只需要包含intents，actions和templates即可。其中templates定义了机器人回复某个意图的模板。

```yaml
actions:
- action_default_fallback
- utter_sys_hi
- utter_什么是T日

intents:
- sys_hi
- 什么是T日

templates:
  utter_sys_hi:
  - text: 你好
  utter_什么是T日:
  - buttons:
    - payload: 已解决
      title: 已解决
    - payload: 未解决
      title: 未解决
    text: T日代表基金的交易日，部分QDII基金交易日可能会受海外节假日的影响，与国内交易日可能不同，具体以基金公司为准。<hr />请问以上是否解决了你的问题？

```

### [Action](https://rasa.com/docs/rasa/core/actions/#)

rasa主要包含四种类型的actions:

1. **Utterance actions**: 以`utter_`开头，发送特定的文本内容给用户。
2. **Retrieval actions**: 以`respond_`开头， 由模型来决定发送何种信息给用户，适用于有大量相似意图的场景。
3. **Custom actions**: 自定义的action，需要在`actions.py`上开发自己的action, 其action必须继承`rasa_sdk.Action`并实现`name`和`run`方法。
4. **Default actions**: 系统默认action，具体有 `action_listen`, `action_restart`, `action_default_fallback`,`action_deactivate_form`, `action_revert_fallback_events`,`action_default_ask_affirmation`, `action_default_ask_rephrase`, `action_back`。这些action都可以被复写，如果自定义了这些默认action则需要在`domain.yml`中注册才能使用

Custom actions Example:

```python
import os
from typing import Any, Text, Dict, List

from rasa_sdk import Action, Tracker
from rasa_sdk.executor import CollectingDispatcher
from rasa.nlu import config


PROJECT_DIR = os.path.dirname(os.path.abspath(__file__))
CFG = config.load(os.path.join(PROJECT_DIR, "config.yml"))
DEFAULT_SYS_INTENT_PREFIX = 'sys_'
MSG_WELCOME = '你好'
MSG_UNHANDLE = '抱歉'


class ActionDefaultFallback(Action):

    def name(self) -> Text:
        return "action_default_fallback"

    def run(self, dispatcher: CollectingDispatcher,
            tracker: Tracker,
            domain: Dict[Text, Any]) -> List[Dict[Text, Any]]:
        similar_nlu_threshold = CFG.get('similar_nlu_threshold', 0.1)
        similar_intents = CFG.get('similar_intents', 3)
        intent_ranking = tracker.latest_message.get("intent_ranking", [])
        if not intent_ranking:
            dispatcher.utter_message(MSG_WELCOME)
        else:
            intents = [i['name'] for i in intent_ranking if i['confidence'] > similar_nlu_threshold
                       and not i['name'].startswith(DEFAULT_SYS_INTENT_PREFIX)][:similar_intents]
            if intents:
                buttons = [{'title': '.'.join([str(index+1), item]), 'payload': f'{item}'}
                           for index, item in enumerate(intents)]
                buttons.append({'title': '以上都不是', 'payload': '以上都不是'})
                dispatcher.utter_button_message(
                    text='你想问的是下面哪个问题喵: \n',
                    buttons=buttons,
                )
            else:
                dispatcher.utter_message(MSG_UNHANDLE)
        return []
```

### [Policies](https://rasa.com/docs/rasa/core/policies/)

`rasa.core.policies.Policy`用来确定对话过程中每个步骤将要采取什么Action。针对每个用户消息，默认可预测的最大action数量为10。你可以通过设置环境变量`MAX_NUMBER_OF_PREDICTIONS`来改变这个值。

每一轮对话中，定义在配置文件中的每个policy在预测下一个action的时候都会有对应的置信度。

如果两个策略得到相同的置信度（比如，memorization和mapping policy的置信度总是1或0），将会考虑到policies的优先级。rasa policies为各个policy设定了默认优先级，更大的数值意味着更高的优先级：

```text
5. FormPolicy
4. FallbackPolicy and TwoStageFallbackPolicy
3. MemoizationPolicy and AugmentedMemoizationPolicy
2. MappingPolicy
1. EmbeddingPolicy, KerasPolicy, and SklearnPolicy
```

优先级设置的情况下，确保了如下行为，如果有个意图预测为mapped action，但是NLU值不高于`nlu_threshold`，bot仍然会fall back。通常情况下，不建议使用相同优先级的多个策略。

如果你创建自己的policy，使用上面的优先级顺序来指导设定你的优先级。比如，你创建了基于深度学习的策略，你的优先级应该是1，和rasa的深度学习策略的优先级保持一致。

**警告**：所有的policy的优先级都可以通过`priority:`参数进行设定，但是我们不建议除了自定义policy以外的策略上进行使用。否则可能会有意料之外的结果。



#### Memoization Policy

MemoizationPolicy仅仅记住了训练数据中的对话。如果确切对话出现在训练数据中，它预测下一个action的置信度为1.0，否则为None，置信度为0.0。

#### Form Policy

FormPolicy是对MemorizationPolicy的扩展，用来处理forms填充的场景。一旦FormAction被调用，FormPolicy会持续预测FormAction，直到所有需要的slots被填满。

#### Mapping Policy

MappingPolicy可以直接将意图映射到actions。这个映射是通过给意图添加一个`triggers`参数实现的，如下`domain.yml`：

```text
intents:
- ask_is_not:
    triggers: action_is_bot
```

一个意图最多被映射成一个action。一旦接收到对应的意图的消息，助手会运行映射的action。然后，它将会监听下一条消息。结合下一个用户消息，将恢复正常预测行为。

#### Fallback Policy

```yaml
policies:
  - name: "FallbackPolicy"
    nlu_threshold: 0.3
    ambiguity_threshold: 0.1
    core_threshold: 0.3
    fallback_action_name: 'action_default_fallback'
```

这个策略的意思是当满足下面三个情况之一则触发`fallback_action_name`对应的action:

1. 意图识别的置信度低于`nlu_threshold`。
2. 排名最高的意图与排名第二的意图之间的置信度差异小于模糊阈值`ambiguity_threshold`。
3. 没有一个对话策略的预测结果的置信度高于`core_threshold`。

#### Keras Policy

`KerasPolicy`使用了基于Keras实现的神经网络选择下一个action。默认的结构是基于[LSTM](https://www.jianshu.com/p/9dc9f41f0b29)（Long short-term memory），这是一种特别的 RNN（Recurrent Neural Networks）。相比于RNN，LSTM解决了长期依赖的问题，通俗来说就是RNN读取的信息，对信息一视同仁：经过处理的信息，RNN认为这些信息的任何一部分都对接下来的信息有影响，全部都抛给接下来处理的程序。对这些信息，RNN进行同样的处理，造成大量无用信息冗余，浪费大量记忆空间，导致关键信息无法突出，更多的信息又无法存储，从而产生较前面的信息RNN记不住的问题。

#### EmbeddingPolicy

[Attention is All You Need](https://arxiv.org/abs/1706.03762)