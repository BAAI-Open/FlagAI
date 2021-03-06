# Data set processing flow
The process of constructing a dataset is the data preprocessing process of NLP. Its main purpose is to rearrange the original scattered file data into data of a unified structure so that the language model can directly use it. The main process of constructing a sample dataset is as follows (take the CommitmentBank dataset as an example):

<div align=center><img src="img/dataset_pipeline.png" width="600px"></div>

At present, there are three kinds of data preprocessing in the project, namely, fine-tuning for classification tasks, pre-training, and fine-tuning for generation tasks. We will expand on them separately in the following.

## Data processing: fine-tuning for classification tasks 

There are two forms of tuning for classification tasks: one is fine-tuning, and the other is the [prompt-tuning](/docs/TUTORIAL_7_PROMPT_LEARNING.md). Prompt-tuning requires an additional cloze template for the task, which is more suitable for limited data. Let's take prompt learning as an example to introduce the data processing method in classification tasks:
### Application code

```python
import torch
from flagai.data.tokenizer import GLMLargeEnWordPieceTokenizer
from flagai.data.dataset import SuperGlueDataset
from flagai.test_utils import CollateArguments
from flagai.data.dataset import ConstructSuperglueStrategy

# get default parameters
cl_args = CollateArguments()

# Create tokenizer
tokenizer = GLMLargeEnWordPieceTokenizer()
            
# Initially read and process the dataset
dataset = SuperGlueDataset(task_name='cb',
                           data_dir='./datasets/',
                           dataset_type='train',
                           tokenizer=tokenizer)

# Construct collate function
collate_fn = ConstructSuperglueStrategy(cl_args, tokenizer, task_name="rte")

# create loader
loader = torch.utils.data.DataLoader(dataset,
                                    batch_size=1,
                                    shuffle=False,
                                    num_workers=1,
                                    drop_last=False,
                                    pin_memory=False,
                                    collate_fn=collate_fn)
```

### Initially read and process the dataset
The corresponding code module is shown below, which consists of two steps: automatically loading the dataset, and unifying the structure of all datasets

```python
dataset = SuperGlueDataset(task_name='cb',
                           data_dir='./datasets/',
                           dataset_type='train',
                           tokenizer=tokenizer)
```
`SuperGlueDataset`is the function in this step???and its major parameters are introduced below???

`task_name`: Identifier of dataset, Supported datasets and their identifiers are given at the table in [1.Load the dataset](#1.Load the dataset).

`data_dir`: Data will be automatically downloaded to `data_dir` directory, which is `./dataset` by default.

`dataset_type`: It can be train/dev/test, which represents train/validation/test set is going to be preprocessed.

`tokenizer`: Constructed tokenizer as introduced in [Tutorial1](/docs/TUTORIAL_1_TOKENIZER.md).




#### 1.Load the dataset

FlagAI currently supports the following classification datasets:

| dataset name                                 | Identifier | Language | Benchmark   | Auto-Download | Fully tested |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------|----------|----------------------------------------------------|---------------|-------------|
| [Broadcoverage Diagnostics](https://github.com/google-research-datasets/boolean-questions)                                               | boolq         | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [CommitmentBank](https://github.com/mcdm/CommitmentBank)                                                                                 | cb            | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [Choice of Plausible Alternatives](https://people.ict.usc.edu/~gordon/copa.html)                                                         | copa          | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [Multi-Sentence Reading Comprehension](https://cogcomp.org/multirc/ )                                                                    | muiltirc      | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [Recognizing Textual Entailment](https://aclweb.org/aclwiki/Recognizing_Textual_Entailment )                                             | rte           | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [Words in Context](https://pilehvar.github.io/wic/ )                                                                                     | wic           | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |                                                   
| [The Winograd Schema Challenge](https://cs.nyu.edu/faculty/davise/papers/WinogradSchemas/WS.html )                                       | wsc           | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [Ant Financial Question Matching Corpus](https://tianchi.aliyun.com/dataset/dataDetail?dataId=106411 )                                   | afqmc         | Chinese  | [CLUE](https://www.clue.ai/index.html)             | ???             | ???           |
| [Short Text Classificaiton for News](https://metatext.io/datasets/toutiao-text-classification-for-news-titles-(tnews)-(clue-benchmark) ) | tnews         | Chinese  | [CLUE](https://www.clue.ai/index.html)             | ???             | ???           |
| [Broadcoverage Diagnostics](https://gluebenchmark.com/diagnostics)                                                                       | ax-b          | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [Winogender Schema Diagnostics](https://github.com/rudinger/winogender-schemas)                                                          | ax-g          | English  | [SuperGLUE](https://super.gluebenchmark.com/tasks) | ???             | ???           |
| [The Corpus of Linguistic Acceptability](https://nyu-mll.github.io/CoLA/)                                                                | cola          | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [The Stanford Sentiment Treebank](https://nlp.stanford.edu/sentiment/index.html)                                                         | sst2          | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [Microsoft Research Paraphrase Corpus](https://microsoft.com/en-us/download/details.aspx?id=52398)                                       | mrpc          | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [Quora Question Pairs](https://data.quora.com/First-Quora-Dataset-Release-Question-Pairs)                                                | qqp           | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [MultiNLI Matched](http://www.nyu.edu/projects/bowman/multinli/)                                                                         | mnli          | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [MultiNLI Mismatched](http://www.nyu.edu/projects/bowman/multinli/)                                                                      | mnli-mm       | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [Question NLI](https://rajpurkar.github.io/SQuAD-explorer/)                                                                              | qnli          | English  | [GLUE](https://gluebenchmark.com/tasks)            | ???             | ???           |
| [X-Stance](https://github.com/ZurichNLP/xstance)                                                                                         | xstance       | English  |                                                    | ???             | ???           |
| [X-Stance (German)](https://github.com/ZurichNLP/xstance)                                                                                | xstance-de    | German   |                                                    | ???             | ???           |
| [X-Stance (French)](https://github.com/ZurichNLP/xstance)                                                                                | xstance-fr    | French   |                                                    | ???             | ???           |
| [RACE](https://www.cs.cmu.edu/~glai1/data/race/)                                                                                         | race          | English  |                                                    | ???             | ???           |
| [AG News](https://www.kaggle.com/datasets/amananandrai/ag-news-classification-dataset)                                                   | agnews        | English  |                                                    | ???             | ???           |
| [Yahoo Answers](https://www.kaggle.com/datasets/jarupula/yahoo-answers-dataset)                                                          | yahoo         | English  |                                                    | ???             | ???           |
| [Yelp Review Polarity](https://www.kaggle.com/datasets/irustandi/yelp-review-polarity)                                                   | yelp-polarity | English  |                                                    | ???             | ???           |
| [Yelp Open Dataset](https://www.yelp.com/dataset)                                                                                        | yelp-full     | English  |                                                    | ???             | ???           |
| [squad](The Stanford Question Answering Dataset)                                                                                         | squad         | English  |                                                    | ???             | ???           |
| [CLUEWSC2020](https://github.com/CLUEbenchmark/CLUEWSC2020)                                                                              | cluewsc       | Chinese  | [CLUE](https://www.clue.ai/index.html)             | ???             | ???           |

The downloaded dataset directory will contain three files, corresponding to the training set data, the validation set data, and the test set data. Take the CommitmentBank dataset as an example, the `train.jsonl` in the directory corresponds to the training set, `val.jsonl `corresponds to the validation set, `test.jsonl` corresponds to the test set. Generally, the training set and test set contain label information, but the test set does not. These data files will be processed separately in the next process.

Different datasets may have different file formats, as well as different structures. Taking the CommitmentBank dataset as an example, the following is an example of it

<div align=center><img src="img/dataset_figure_0.png" width="600px"></div>

It contains four parts, as shown below

| key        | meaning                                                   | value                              |
|-----------|-------------------------------------------------------|--------------------------------|
| premise   | premise text                                                 | Mary is a high school student. |
| hypothesis | hypothetical text                                                 | Mary is a student              |
| label     | The tags representing the relationship between the premise and the hypothesis. Include three kinds of inclusion, neutral and contrast | entailment                     |
| idx       | The sequence number of the sample in the dataset           | 10                             |

The specific structure of all FlagAI supported datasets can be viewed [here](DATASET_EXAMPLE.md).

#### 2. Unified dataset structure
In this step, we will unify the data structures of different datasets to facilitate subsequent processing. The details of this structure are as follows:


| key     | meaning                                                    | format |
|--------|-------------------------------------------------------|------|
| guid   | a unique textual identifier                                                  | str  |
| text_a | the sequence of text                                                  | str  |
| text_b | an optional, second sequence of text | str  |
| label  | an optional label                                            | str  |
| logits | an optional list of per-class logits                                              | list |
| meta   | an optional dictionary to store arbitrary meta information                        | dict |
| ids    | an optional numeric index                                                    | int  |

When the dataset is built, you can view one of the samples directly in the code by indexing:

```python
example = dataset[3]  # The third example in dataset 
```

For instance, the example of CommitBank in the previous step will be processed into the following form

<div align=center><img src="img/dataset_figure_2.png" width="500px"></div>

Noted that if text_a and text_b cannot be filled with background text information because the data structure is too complex, you can put the rest of the information in meta.



### Organize the data into input to the model

The corresponding function is implemented in the following function, which consists of two steps: constructing the template, segmenting the word and constructing the input sample.

```python
collate_fn = ConstructSuperglueStrategy(cl_args,
                                        tokenizer,
                                        task_name=task_name)
```

#### 1. Build the cloze template

A cloze template contains background text, slots, and options provided to the slots. Models need to find the right options and fill in the blanks.

For each different task, we need to construct cloze questions of different structures for the model to answer. Taking the CommitmentBank dataset as an example, it considers whether the hypothesis can be deduced from the premise, and there are only three results: contradiction/neutral/entailment. Then we can construct the following cloze problem, where contrast/neutral/entailment correspond to true/false/neither respectively.

<div align=center><img src="img/dataset_figure_3.png" width="500px"></div>

It can be seen that it can be roughly divided into two steps: the first step is to combine the existing text to make it look like a cloze format; the second step is to convert the original label text into a new label, which may be filled in. Option to enter vacancies.

#### 2. Word segmentation and construct input samples
Next, we need to construct the input of the model. The first step is word segmentation, and then we need to divide it into two cases:
In the first case, the label categories contained in the dataset are limited. For example, in the CommitmentBank dataset, there are only three kinds of label texts???intailment/contradiction/neutral, which are common in classification tasks. In the second case, each cloze will give different options (usually a long text). For example, in some reading comprehension datasets, each option is a different understanding of the text. The two cases are handled as follows:

**a) Cloze for a single token**

| key             | dimension        | meaning         | Construction method                                          |
|-----------------|--------------------------------------|------------|-----------------------------------------------|
| input_ids       | torch.Size([seq_length<sup>1</sup>]) | input matrix   | composed of the cloze text from the previous step, plus some special characters <sup>2</sup>            |
| labels          | labels: torch.Size([1])              | labels         | corresponding numeric labels, such as 0,1,2... |
| position_ids    | torch.Size([2??? seq_length])   | position encoding  | refer to [GLM process] (GLM.md), the first line represents the absolute position of the token, the second line represents the relative position of the occluded part  |
| attention_mask  | torch.Size([1])          | separator position  |                |
| target_ids      | torch.Size([num_labels<sup>3</sup>]) | full label list | All label texts correspond to the labels of a single token, and then put the serial numbers of these labels into target_ids  |
| logit_mask      | torch.Size([seq_length])  | Whether the corresponding text is an answer | For each token, if it is an answer, the corresponding place is 1, otherwise it is 0 |                                                   

<sup>1</sup>: seq_length represents the specified maximum length of each input vector

<sup>2</sup>: As illustrated in the following figure, the process of adding special characters: add the [CLS] symbol at the beginning of the sentence and the [EOS] symbol at the end of the sentence until the length reaches seq_length. If the text output by cloze has two paragraphs, it will be in the middle Add [SEP] symbol
<div align=center><img src="img/dataset_figure_5.png" width="500px"></div>

<sup>3</sup>: num_labels represents the number of options in the cloze problem

**b) Cloze for multiple tokens**



| key             | dimension        | meaning         | Construction method                 |                        
|-----------------|------------------|-----------------|-------------------------------------|
| input_ids       | torch.Size([num_labels, seq_length])    | input matrix | copy the corresponding text num_labels copies                 |
| labels          | labels: torch.Size([1])                 | Label         |                                  |
| position_ids    | torch.Size([num_labels, 2??? seq_length]) | position encoding | Copy the original position code num_labels copies |
| attention_mask  | torch.Size([num_labels])                | separator position      | copy num_labels copies        |
| target_ids      | torch.Size([num_labels, seq_length])    | Information about each option    | Each column of the matrix represents each option, and each row represents the text corresponding to the current option      |
| logit_mask      | torch.Size([num_labels, seq_length])    | Whether the corresponding text is an answer | copy num_labels copies                    |                                                   



### Create loader

Finally, put the data into the [PyTorch Loader](https://pytorch.org/docs/stable/data.html?highlight=dataloader#torch.utils.data.DataLoader).

```python
loader = torch.utils.data.DataLoader(dataset,
                                    batch_size=1,
                                    shuffle=False,
                                    num_workers=1,
                                    drop_last=False,
                                    pin_memory=False,
                                    collate_fn=collate_fn)
```
The data in the Dataloader can be viewed by the following methods

```python
for data_iterator in train_loader:
    for key, value in data_iterator.items():
        print(key, value)
    # break
```
Once the loader is constructed, it can be used for the subsequent training and prediction process. <br /><br /><br />

## Data processing: GLM pre-training task

Sample pre-training task data format:
```text
{
    "RECORDS": [
        {
            "id": 0,
            "uniqueKey": "",
            "titleUkey": "",
            "dataType": "",
            "title": "",
            "content": "?????????????????????????????????????????????????????????????????????
            ?????????????????????????????????????????????????????????????????????????????????????????????
            ?????????????????????????????????????????????????????????????????????????????????????????????
            ????????????????????????????????????????????????????????????????????????????????????????????????
            ????????????????????????????????????????????????????????????????????????????????????????????????
            ???????????????????????????????????????????????????????????????????????????????????????????????????
            ??????????????????????????????????????????????????????????????????????????????????????????????????????
            ?????????????????????????????????????????????????????????????????????????????????????????????????????????
            ????????????????????????????????????????????????????????????????????????????????????????????????????????????
            ????????????????????????????????????????????????????????????????????????????????????????????????
            ??????????????????????????????"
        },
    ]
}
```
Pre-trained task processing example code:

```python
tokenizer = GLMLargeChTokenizer(add_block_symbols=True,
                                add_task_mask=True,
                                add_decoder_mask=False,
                                fix_command_token=True)

ds_args = DatasetArguments()

tokenizer = GLMLargeChTokenizer(fix_command_token=True,
                                add_block_symbols=True,
                                add_task_mask=True,
                                add_decoder_mask=False)

ds_args = add_args(ds_args, tokenizer)

def create_dataset(tokenizer, should_split):
    dataset = get_dataset_lazy("./examples/glm_pretrain/data", # load
                               tokenizer=tokenizer,
                               pre_tokenize=True,
                               num_processes=10,
                               no_lazy_loader=True)
    if should_split:
        datasets = split_ds(dataset, split=[.8, .2, .0], shuffle=True) # Manual segmentation
    else:
        datasets = [dataset]

    datasets = [
        BlockDataset(ds,
                     tokenizer,
                     max_seq_len=512,
                     sample_across_doc=True,
                     non_sentence_start=0.0) if ds is not None else None
        for ds in datasets
    ]
    return datasets

datasets = create_dataset(tokenizer, should_split=True)
```

Pre-training data processing also follows the same process, with the following differences
1. The pre-training data set is not divided into training set, validation set, and test set by default, so it needs to be divided manually
2. Since the pre-training data set is generally relatively large, lazy loading is used. Lazy loading only instantiates the object when it is actually used, which is a relatively resource-saving operation.
3. During pre-training, the collate function will randomly process data according to three different modes: bert mode (occlude random intervals), sentence mode (occlude according to complete sentences) and gpt mode (occlude only a long section). The input of the model will also have one more `mode` key than the general generation task.
4. There is no need to add templates for pre-training, just follow the table below to build the model input


| key             | dimension        | meaning         | Construction method                                            |
|----------------|--------------------------------------|------------|--------------------------------------------------|
| input_ids      | torch.Size([seq_length<sup>1</sup>]) | input matrix       | consists of the template text from the previous step, plus some special characters                          |
| position_ids   | torch.Size([2??? seq_length])          | position encoding | refer to [GLM process](GLM.md), the first line represents the absolute position of the token, the second line represents the relative position of the occluded part |
| attention_mask | torch.Size([1])                      | delimiter position | for the pattern of the generated class, get the position where the source text ends; otherwise get the position where the input text ends |
| target_ids     | torch.Size([num_labels<sup>3</sup>]) | full label list | occluded text |                                         
| logit_mask     | torch.Size([seq_length])             | By occlusion, the model will only process the loss of the target text part | For each token, if it is an answer, the corresponding place is 1, otherwise it is 0 |                       
| mode           | str                                  | Data processing mode            |  

## Data processing: Generating task fine-tuning
The code implementation is as follows:

```python 
import torch
from flagai.data.dataset import Seq2SeqDataset
from flagai.data.tokenizer import GLMLargeEnWordPieceTokenizer
from flagai.test_utils import Seq2SeqCollateArguments
from flagai.data.dataset import ConstructSeq2seqStrategy

# get default parameters
cl_args = Seq2SeqCollateArguments()

# create tokenizer
tokenizer = GLMLargeChTokenizer(add_block_symbols=True,
                       TUTORIAL_4_DATASET.md         add_task_mask=False,
                                add_decoder_mask=False,
                                fix_command_token=False)
            
# Initially read and process the dataset
dataset = Seq2SeqDataset(task_name='cmrc',
                           data_dir='./datasets/',
                           dataset_type='train',
                           tokenizer=tokenizer)

# build collate function
collate_fn = ConstructSeq2seqStrategy(cl_args, tokenizer, task_name="rte")

# Create a loader
loader = torch.utils.data.DataLoader(dataset,
                                    batch_size=1,
                                    shuffle=False,
                                    num_workers=1,
                                    drop_last=False,
                                    pin_memory=False,
                                    collate_fn=collate_fn)
```

### Supported Tasks
| Supported Tasks                                                                                    | Identifier       | Language | Auto-download | Fully-tested |
|----------------------------------------------------------------------------------------------------|------------------|----------|---------------|--------------|
| [Reading Comprehension for Simplified Chinese](https://hfl-rc.github.io/cmrc2018/)                 | cmrc             | Chinese       | ???             | ???            |
| [The Winograd Schema Challenge](https://cs.nyu.edu/faculty/davise/papers/WinogradSchemas/WS.html ) | wsc              | English       | ???             | ???            |
| [English Gigaword](https://catalog.ldc.upenn.edu/LDC2003T05)                                       | gigaword         | English       | ???             | ???            |
| [CNN/Daily Mail](https://paperswithcode.com/dataset/cnn-daily-mail-1)                              | cnn_dm           | English       | ???             | ???            |
| Lang-8 and HSK                                                                                     | lang8_hsk        | Chinese       | ???             | ???            |
| [XSum](https://paperswithcode.com/dataset/xsum)                                                    | xsum             | English       | ???             | ???            |
| [squad](The Stanford Question Answering Dataset)                                                   | squad_generation | English       | ???             | ???            |                                                   




### Initially read and process the dataset

Currently, [CMRC2018](https://www.clue.ai/introduce.html) task is supported. CMRC is a reading comprehension task that needs to answer a series of questions based on the background text. An example of its data structure is as follows:

```text
{'paragraphs': 
    [{'id': 'TRAIN_186', 
    'context': '?????????????????????????????????????????????????????????????????????????????????????????1963?????????
    ????????????1990??????????????????????????????????????????????????????1994?????????????????????????????????????????????
    ???????????????2009???2????????????????????????1919???6???15?????????????????????????????????????????????????????????
    ??????????????????????????????????????????????????????????????????????????????????????????1940???????????????????????????
    ??????????????????????????????1949???6???6??????????????????????????????????????????????????????????????????????????????
    ??????1950?????????????????????????????????????????????????????????????????????????????????????????????1954????????????
    ??????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????
    ??????????????????????????????????????????????????????1960???????????????????????????????????????????????????????????????
    ??????????????????????????????1963???4???5??????????????????????????????????????????????????????????????????8???15??????
    ??????????????????????????????????????????????????????????????????????????????????????????30?????????????????????????????????
    ??????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????
    ??????????????????????????????????????????????????????1990???????????????????????????????????????6???18??????????????????
    ??????????????????????????????????????????????????????????????????????????????1994???3???23??????????????????????????????
    ?????????...', 
    'qas':{'question': '??????????????????????????????', 'id': 'TRAIN_186_QUERY_4', 
    'answers': [{'text': '????????????2009???2???22????????????????????????', 'answer_start': 759}]}]}], 
    'id': 'TRAIN_186', 'title': '?????????'}
```
When using it, we can change the `task_name` parameter to `cmrc`. The implementation process is similar to the fine-tuning of the classification task, and the data set will be initially processed into the same structure in the end. The corresponding code is as follows:

```python 
dataset = Seq2SeqDataset(task_name='cmrc', data_dir='./datasets/', 
                            dataset_type='train', tokenizer=tokenizer) 
```

### Organize the data into input to the model

The code is shown below. Compared with the generation task, it is also the construction template and model input, the difference is that the construction method is different

```python 
collate_fn = ConstructSeq2seqStrategy(cl_args,
                                        tokenizer,
                                        task_name=task_name) 
```

#### 1. Build a fill-in-the-blank template
Since it is a reading comprehension task, it needs to be reflected in the template to answer the specified reading comprehension question, refer to the following construction method
<div align=center><img src="img/dataset_figure_4.png" width="500px"></div>


#### 2. Word segmentation and construct input samples
Similar to pre-training, the difference is that there is no mode key.
