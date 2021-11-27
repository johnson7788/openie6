# IMoJIE

基于迭代记忆的联合OpenIE Iterative Memory based Joint OpenIE
一个BERT-base的OpenIE系统，使用迭代Seq2Seq模型生成提取，如以下出版物所述，在ACL 2020中，[链接](https://arxiv.org/abs/2005.08178)

## Installation Instructions
Use a python-3.6 environment and install the dependencies using,
```
pip install -r requirements.txt
```
这将根据文件夹中的代码安装自定义版本的allennlp和pytorch_transformers。
所有报告的结果都是基于在TeslaV100 GPU（CUDA 10.0）上运行的pytorch-1.2。结果可能随着环境的变化而略有不同。

## 执行说明
### Data Download
bash download_data.sh 

这将下载（训练、开发、测试）数据

### 运行代码

IMoJIE (on OpenIE-4, ClausIE, RnnOIE bootstrapping data with QPBO filtering)
```
python allennlp_script.py --param_path imojie/configs/imojie.json --s models/imojie --mode train_test 
```

Arguments:
- param_path: 包含模型所有参数的文件
- s:  将保存模型的目录的路径
- mode: train, test, train_test

重要基线：

IMoJIE (on OpenIE-4 bootstrapping)
```
python allennlp_script.py --param_path imojie/configs/ba.json --s models/ba --mode train_test 
```

CopyAttn+BERT (on OpenIE-4 bootstrapping)
```
python allennlp_script.py --param_path imojie/configs/be.json --s models/be --mode train_test --type single --beam_size 3
```

### 生成聚合数据

Score using bert_encoder trained on oie4: 
```
python imojie/aggregate/score.py --model_dir models/score/be --inp_fp data/train/4cr_comb_extractions.tsv --out_fp data/train/4cr_comb_extractions.tsv.be 
```

Score using bert_append trained on comb_4cr (random): 
```            
python imojie/aggregate/score.py --model_dir models/score/4cr_rand --inp_fp data/train/4cr_comb_extractions.tsv.be --out_fp data/train/4cr_comb_extractions.tsv.ba
```

Filter using QPBO:
```
python imojie/aggregate/filter.py --inp_fp data/train/4cr_comb_extractions.tsv.ba --out_fp data/train/4cr_qpbo_extractions.tsv
```

### Note on the notation
我们一直在内部称我们的模型为“bert append”（ba），直到论文提交之日，CopyAttention+bert称为“bert编码器”（be）。因此，您将在整个代码库中找到类似的引用。在这种情况下，IMoJIE是基于qpbo过滤数据进行训练的。

### Expected Results
Format: (Prec/Rec/F1-Optimal, AUC, Prec/Rec/F1-Last)

models/imojie/test/carb_1/best_results.txt \
(64.70/45.60/53.50, 33.30, 63.80/45.80/53.30)

models/ba/test/carb_1/best_results.txt \
(Prec/Rec/F1-Optimal, AUC, Prec/Rec/F1-Last) \
(63.50/45.80/53.20, 33.10, 60.40/46.30/52.40)

models/be/test/carb_3/best_results.txt \
(Prec/Rec/F1-Optimal, AUC, Prec/Rec/F1-Last) \
(59.50/45.50/51.60, 32.80, 52.90/46.70/49.60)

## Resources

下载预训练的模型：
```
zenodo_get 3779954
```

Downloading the data:
```
zenodo_get 3775983
```

Downloading the results:
```
zenodo_get 3780045
```

### Citing
If you use this code in your research, please cite:

```
@inproceedings{kolluru&al20,
    title = "{IM}o{JIE}: {I}terative {M}emory-{B}ased {J}oint {O}pen {I}nformation {E}xtraction",
    author = "Kolluru, Keshav  and
      Aggarwal, Samarth  and
      Rathore, Vipul and
      Mausam, and
      Chakrabarti, Soumen",
    booktitle = "The 58th Annual Meeting of the Association for Computational Linguistics (ACL)",
    month = July,
    year = "2020",
    address = {Seattle, U.S.A}
}
```

## Contact
In case of any issues, please send a mail to
```keshav.kolluru (at) gmail (dot) com``` 


