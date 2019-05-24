# 监督学习

- 回归算法
- 线性回归
- 逻辑回归
- 神经网络
- 支持向量机

# 非监督学习

- 聚类算法
- 降维算法
  - PCA（主成分算法） ：线性代数运算
- 推荐算法
- 协同过滤算法
- 梯度下降法
- BP算法

#### 机器学习流程

A：数据导入
B：数据预处理
C：特征工程
D：拆分
E：训练模型
F：评估模型
G：预估新数据

#### 机器学习平台

scikit-learn
TensorFlow
Caffe
PyTorch
Keras
Spark MLlib是Spark常见机器学习的库

# Python数据分析工具简介

## NumPy

数组分析的核心基础库， 类似Matlab矩阵

## SciPy

- 科学计算库
- 插值：scipy.interpolate
- 统计：[scipy.stats](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.expon.html)
- 优化: scipy.optimize
- 积分: scipy.integrate quad
- 线代: scipy.linalg

> 特征值分解， SVD分解, 正定矩阵， 特征值

## matplotlib 

- 可视化基础工具

> 学习参考官网demo

## Pandas数据分析

- 基础数据结构
  -- 一维Series
  -- 二维DataFrame
  -- 三维Panel
- 字典和Numpy数组的结合体µ
- 提供类似SQL语言的支持
- 集合处理多种数据格式的功能
  -- h5py，json，csv，pickle
- Pandas数据分析
  -- 对读写多种数据的良好支持

## Scikit-Learn

- 统一使用方法： .fit, .predict
- 分类: SVM, K近邻, 决策树
- 回归: SVR, 岭回归
- 聚类： K均值， 
- 集成： 随机森林， Adaboost， GBDT
- 降维： PCA

### 学习资料

Scipy-Lecture
[vim plugin](https://realpython.com/vim-and-python-a-match-made-in-heaven/#auto-complete)

## 支持向量机 SVM

性能强大且广泛应用的学习算法. SVM中，我们优化的目标是*最大*分类间隔。此处间隔是两个分离的超平面（决策边界）

> 感知器: 最简单的单层神经网络

### 算法思想

基于训练集D在样本空间中找到一个划分超平面，将不同类别的样本分开，并且最大化间隔.
wTx + b = 0

间隔(margin)：两个异类变量支持向量到超平面的距离之和

### 核函数

利用"Kernel技巧"来解决`非线性`可分问题

最为广使用的核函数就是径向基函数核（Radial Basis
 Function kernel, RBF kernel) 或高斯核（Gaussian kernel）

```python
 	svm = SVC(kernel='rbf', random_state=0, gamma=0.10, C=10.0)
 	svm.fit(X_xor, y_xor)
```

> `c` 和 `gamma`是控制参数，影响泛化能力.

