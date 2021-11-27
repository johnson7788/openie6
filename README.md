# OpenIE6 System 

This repository contains the code for the paper:\
OpenIE6: Iterative Grid Labeling and Coordination Analysis for Open Information Extraction\
Keshav Kolluru*, Vaibhav Adlakha*, Samarth Aggarwal, Mausam and Soumen Chakrabarti\
EMNLP 2020

\* denotes equal contribution

## Installation
```
conda create -n openie6 python=3.6
conda activate openie6
pip install -r requirements.txt
python -m nltk.downloader stopwords
python -m nltk.downloader punkt 
```
所有结果都是在使用CUDA 10.0的V100 GPU上获得的。
注意：HuggingFacetransformer==2.6.0是必要的。最新版本在代码中使用tokenizer的方式有一个突破性的变化。它不会引发错误，但会给出错误的结果

## 下载资源
Download Data (50 MB)
```
zenodo_get 4054476
tar -xvf openie6_data.tar.gz
```

Download Models (6.6 GB)
```
zenodo_get 4055395
tar -xvf openie6_models.tar.gz
```
<!-- wget www.cse.iitd.ac.in/~kskeshav/oie6_models.tar.gz
tar -xvf oie6_models.tar.gz
wget www.cse.iitd.ac.in/~kskeshav/oie6_data.tar.gz
tar -xvf oie6_data.tar.gz
wget www.cse.iitd.ac.in/~kskeshav/rescore_model.tar.gz
tar -xvf rescore_model.tar.gz
mv rescore_model models/ -->

## 运行模型

New command:
```
python run.py --mode splitpredict --inp sentences.txt --out predictions.txt --rescoring --task oie --gpus 1 --oie_model models/oie_model/epoch=14_eval_acc=0.551_v0.ckpt --conj_model models/conj_model/epoch=28_eval_acc=0.854.ckpt --rescore_model models/rescore_model --num_extractions 5 
```

Expected models: \
models/conj_model: Performs coordination analysis \
models/oie_model: Performs OpenIE extraction \
models/rescore_model: Performs the final rescoring 

--inp sentences.txt - File with one sentence in each line 
--out predictions.txt - File containing the generated extractions

gpus - 0 for no GPU, 1 for single GPU

Additional flags -
```
--type labels // outputs word-level aligned labels to the file path `out`+'.labels'
--type sentences // outputs decomposed sentences to the file path `out`+'.sentences'
```

补充说明。

1. 该模型是用tokenized的句子训练的，因此在预测过程中也需要tokenized的句子。该代码目前使用nltk tokenization来实现这一目的。这将导致最终的句子与输入的句子不同，因为它们将是tokenization的版本。如果这不可取，你可以在data.py中注释nltk tokenization，并确保你的句子事先被tokenize。
2. 由于连接模型的训练数据的缺陷，它要求句子以句号结尾才能正常运行。


## 训练模型

### Warmup Model
训练:
```
python run.py --save models/warmup_oie_model --mode train_test --model_str bert-base-cased --task oie --epochs 30 --gpus 1 --batch_size 24 --optimizer adamW --lr 2e-05 --iterative_layers 2
```

测试:
```
python run.py --save models/warmup_oie_model --mode test --batch_size 24 --model_str bert-base-cased --task oie --gpus 1
```
Carb F1: 52.4, Carb AUC: 33.8


预测
```
python run.py --save models/warmup_oie_model --mode predict --model_str bert-base-cased --task oie --gpus 1 --inp sentences.txt --out predictions.txt
```

Time (Approx): 142 extractions/second

### 限制模型
Training
```
python run.py --save models/oie_model --mode resume --model_str bert-base-cased --task oie --epochs 16 --gpus 1 --batch_size 16 --optimizer adam --lr 5e-06 --iterative_layers 2 --checkpoint models/warmup_oie_model/epoch=13_eval_acc=0.544.ckpt --constraints posm_hvc_hvr_hve --save_k 3 --accumulate_grad_batches 2 --gradient_clip_val 1 --multi_opt --lr 2e-5 --wreg 1 --cweights 3_3_3_3 --val_check_interval 0.1
```

Testing
```
python run.py --save models/oie_model --mode test --batch_size 16 --model_str bert-base-cased --task oie --gpus 1 
```
Carb F1: 54.0, Carb AUC: 35.7

Predicting
```
python run.py --save models/oie_model --mode predict --model_str bert-base-cased --task oie --gpus 1 --inp sentences.txt --out predictions.txt
```

Time (Approx): 142 extractions/second

NOTE: Due to a bug in the code, [link](https://github.com/dair-iitd/openie6/issues/10), we end up using a loss function based only on the constrained loss term and not the original Cross Entropy (CE) loss. It still seems to work well as the warmup model is already trained with the CE loss and the constrained training is initialized from the warmup model.

### 运行协调分析 Coordination Analysis
```
python run.py --save models/conj_model --mode train_test --model_str bert-large-cased --task conj --epochs 40 --gpus 1 --batch_size 32 --optimizer adamW --lr 2e-05 --iterative_layers 2
```

F1: 87.8

### 最终模型

Running
```
python run.py --mode splitpredict --inp carb/data/carb_sentences.txt --out models/results/final --rescoring --task oie --gpus 1 --oie_model models/oie_model/epoch=14_eval_acc=0.551_v0.ckpt --conj_model models/conj_model/epoch=28_eval_acc=0.854.ckpt --rescore_model models/rescore_model --num_extractions 5 
python utils/oie_to_allennlp.py --inp models/results/final --out models/results/final.carb
python carb/carb.py --allennlp models/results/final.carb --gold carb/data/gold/test.tsv --out /dev/null
```
Carb F1: 52.7, Carb AUC: 33.7
Time (Approx): 31 extractions/second

Evaluate using other metrics (Carb(s,s), Wire57 and OIE-16)
```
bash carb/evaluate_all.sh models/results/final.carb carb/data/gold/test.tsv
```

Carb(s,s): F1: 46.4, AUC: 26.8
Carb(s,m) ==> Carb: F1: 52.7, AUC: 33.7
OIE16: F1: 65.6, AUC: 48.4
Wire57: F1: 40.0

## CITE
If you use this code in your research, please cite:

```
@inproceedings{kolluru&al20,
    title = "{O}pen{IE}6: {I}terative {G}rid {L}abeling and {C}oordination {A}nalysis for {O}pen {I}nformation {E}xtraction",\
    author = "Kolluru, Keshav  and
      Adlakha, Vaibhav and
      Aggarwal, Samarth and
      Mausam, and
      Chakrabarti, Soumen",
    booktitle = "The 58th Annual Meeting of the Association for Computational Linguistics (ACL)",
    month = July,
    year = "2020",
    address = {Seattle, U.S.A}
}
```


## LICENSE

Note that the license is the full GPL, which allows many free uses, but not its use in proprietary software which is distributed to others. For distributors of proprietary software, you can contact us for commercial licensing.

## CONTACT

In case of any issues, please send a mail to ```keshav.kolluru (at) gmail (dot) com```

