## 背景

[PyCaret](https://pycaret.org/) 是一款Python中的开源低代码（low-code）机器学习库，
支持在「低代码」环境中训练和部署有监督以及无监督的机器学习模型，提升机器学习实验的效率。
PyCaret本质上是Python的包装器，它围绕着多个机器学习库和框架，
例如scikit-learn，XGBoost，Microsoft LightGBM，spaCy等。

## 模块
PyCaret中目前支持以下有监督和无监督模型：

- 有监督学习模型
1. 分类[Classification](https://pycaret.org/clf101/)

    1.1 分类指标：
   - Accuracy：预测对的样本占所有样本的百分比
   - AUC(Area under the curve),用于评价二值分类器，值越大，表示分类效果越好
   - Recall：召回率：表示原样本中的正样本有多少被预测正确了
   - Precision：正确率
   - F1: 正确率和召回率的调和平均数，最大为1，最小为0。F1分数认为召回率和精确率同等重要
   - Kappa：多分类模型准确度的评估，值越高代表模型实现的分类准确度越高；
   - MCC：马修斯相关系数，衡量不平衡数据集的指标比较好
2. 回归 [Regression](https://pycaret.org/reg101/)

- 无监督学习模型
1. 聚类 [Clustering](https://pycaret.org/clu101/)
2. 异常检测 [Anomaly Detection](https://pycaret.org/ano101/)
3. 自然语言处理[Natural Language Processiong](https://pycaret.org/nlp101/)
4. 关联规则挖掘 [Association Rule Mining](https://pycaret.org/arul101/)

## PyCaret 模型训练过程
1. get data
2. setup()
所有的数据预处理步骤都在setup()中执行，比如缺失值插补，分类变量编码，标签编码（将yes或no转换为1或0）和训练、测试集拆分（train-test-split）等。
3. compare model
这是在有监督的机器学习实验（分类或回归）中建议的第一步。此功能训练模型库中的所有模型，并使用k倍交叉验证（默认10倍）比较通用评估指标。
使用的评估指标是
- 分类：Accuracy（准确度），AUC，Recall（召回率），Precision（精确度），F1，Kappa
- 回归：MAE，MSE，RMSE，R2，RMSLE，MAPE
4. create model
仅接受一个参数，即作为字符串输入传递的模型名称。此函数返回具有k倍交叉验证分数和训练有素的模型对象的表格。
5. tune model
tune_model功能用于机器学习模型的自动调整超参数。PyCaret 在预定义的搜索空间上使用随机网格搜索。此函数返回具有k倍交叉验证分数和训练有素的模型对象的表格。
6. ensemble model
默认情况下，“Bagging”方法用于ensembling，可使用ensemble_model函数中的method参数将其更改为“Boosting” 。
PyCaret还提供blend_models和stack_models功能来集成多个训练过的模型
7. plot model
可以使用plot_model函数对经过训练的机器学习模型进行性能评估和诊断。它使用训练有素的模型对象和作图的类型作为plot_model函数中的字符串输入。
或者，可以使用评估模型（evaluate_model）函数通过botebook中的用户界面查看作图效果。
8. interpret model
PyCaret 使用interpret_model函数实现SHAP（SHapley Additive exPlanations）
9. predict model
10. deploy model
11. save model
12. automl


### flask-pycaret容器
用python的web框架，将pycaret提供的功能，封装成rest api的形式，方便其余模块的调用。

该容器提供的api如下（待细化）：
- set up
- compare model
- create model
- tune model
- predict model
- get log
- get model status
- deploy model
- save model
- ...

### 数据集管理
用于机器学习训练的数据集，后端可以通过在digits中来实现。包括以下功能：
- 数据集的上传，
- 数据集列表查询，
- 数据集详情
- 数据集删除


### Pycaret Usage 

#### 环境安装

1. 创建conda虚拟环境

```markdown
conda create --name yourenvname python=3.6
conda activate yourenvname
pip install pycaret==2.0
#create notebook kernel connected with the conda environment
python -m ipykernel install --user --name yourenvname --display-name "display-name"
```
2. 
```markdown
yum install python-pip -y
安装python3环境 
安装virtualenv
//创建python3 virtualvenv
virtualenv -p python3 pycaret-venv3
//进入虚拟环境
source pycaret-venv3/bin/activate
//install pycaret
# pip install pycaret==2.0
//退出虚拟环境
deactivate
```

### docker image build

```markdown
# cat Dockerfile
FROM python:3.7-slim
WORKDIR /app
ADD . /app
RUN apt-get update && apt-get install -y libgomp1
RUN pip install --trusted-host pypi.python.org -r requirements.txt
CMD pytest //TODO

# cat Dockerfile
FROM python:3.7-slim
WORKDIR /app
ADD . /app
RUN apt-get update && apt-get install -y libgomp1
RUN pip install --trusted-host pypi.python.org -r requirements.txt
CMD pytest //TODO
```