# RASA

Rasa-1.4.3

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

改名了会寻找data/目录下的数据文件来训练模型，并且训练好的模型会保存在models/目录下，模型以nlu为命名前缀。

### 启动服务

```shell
$ rasa shell nlu
```

该命令可以让用户通过命令行的方式去验证模型的效果，默认启用最新的训练模型

也可以指定模型

```shell
$ rasa shell -m models/nlu-20191031-120620.tar.gz
```

甚至可以启动一个NLU的服务

```shell
$ rasa run --enable-api -m models/nlu-20191031-120620.tar.gz
$ curl localhost:5005/model/parse -d '{"text":"hello"}'
```

## NLU

NLU是一个开源的用于意图分类，响应检索和实体抽取的自然语言处理工具，是rasa框架的一个模块，也可以单独使用。

比如由`"I am looking for a Mexican restaurant in the center of town"`可以获得结构化数据如下：

```json
{
  "intent": "search_restaurant",
  "entities": {
    "cuisine" : "Mexican",
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

## CORE



