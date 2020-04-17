# 刀具剩余寿命预测
[工业大数据创新竞赛链接](http://www.industrial-bigdata.com/competition/competitionAction!showDetail34.action?competition.competitionId=3)
数据集我也没有，大家不要再发邮件咨询了。
## 背景介绍 ##
　　此次竞赛提供了刀具全生命周期的实验数据，数据分为控制器数据（plc）和振动采集数据（sensor），任务是给定一段时间刀具的数据，对其寿命进行预测。
PHM的全称为Prognostics and Health Management，其目的在于不需要了解太多的领域知识的前提下，构建基于数据驱动的模型来实现相应的任务。PHM的主要任务可分为四类：健康评估(Health Assessment)、故障检测(Fault Detction）、故障诊断(Fault Diagnosis)以及预测(Prognosis)。其中本次比赛会涉及到了PHM的健康评估和预测两方面的问题。

## 思路分析 ##
　　控制器数据可以用于做工况的模式匹配，提取振动采集数据中有用特征。

#### 1.单一工况
 　　在工况单一的情况下，常见的做法是：基于已有的数据为刀具构建一个虚拟健康指标（health indicator）。然后以健康指标为特征，剩余寿命占比（RULR）作为训练标签，把预测问题转化为一个回归问题。关于RULR，它是一个0~1的值，表示的是当前剩余寿命占总寿命的比值。
     由于产品在出厂时的品质存在差异，因此即使在完全相同的工况下，刀具的最终寿命也不尽相同。而RULR比时间更能反映刀具的内在的健康状态，我们把RULR作为我们训练的预测目标。  
   **这类做法的关键在于：如何构建一个健康模型？**  
　　本项目中截取了刀具全生命周期的前10%左右的数据作为其健康数据，并假设特征符合多元高斯分布，理论上健康数据将会是一个超球面，我们从而得到球心坐标。接着计算剩余的90%的数据到超球面的距离，我们称之为“偏移距离”。偏移距离就是我们要求的健康指标。
本项目的代码实现了上述方法。

#### 2.复杂工况
　　若刀具的整个生命周期中，经历的工况不止一种，那么想要直接去构建一个健康指标就不那么容易了。这时候我们需要获取更多的信息——刀具的工况信息。
　　在本项目中，工况信息包括：走刀轨迹(x,y,z)和主轴负载(spindle_load)。因为数据都是时间序列，使用tsfresh提取统计特征，再基于统计特征做无监督的聚类。从而我们得到“工况模式”这一特征。
把“工况模式”这一特征和其他传感器特征拼接在一起，送入回归学习器进行训练。
本项目实现了探索“工况模式”代码，只需要稍加修改，便可以进行训练。
此外，在数据探索过程中发现，主轴负载(spindle_load)具有比较好的趋势性。因此有一种很简便但是不够精确的方法是：直接基于‘spindle_load’特征，训练回归模型。之所以这样可以做的原因是：相对于其他的特征来说，只有‘spindle_load’具有较好的长期趋势性，可见工况等因素对它的影响较小。  
 <div align=center><img  src="https://github.com/ultimatejoe/rul_of_cutter/blob/master/splendid_load%E8%B6%8B%E5%8A%BF%E6%80%A7.PNG"/>
 </div>  
 ‘spindle_load’具有较好的长期趋势性
 
#### 3.不需要考虑工况
如果现有的数据库中已经保存了大量的刀具的历史数据并记录了寿命标签，我们可以直接把测试集的时间序列片段同历史数据做相似度匹配，相似度最高的一部分数据的寿命值去做加权平均就是测试集的预测寿命。

#### 4.总结
处理工业中数据，我们常常会遇到如下困难：

 - 情况复杂多变，比如工况的分类我们就没法给出一个具体的数。再比如，机器的种类型号繁多，我们不可能对每一种机器单独设计模型。因此，寻找一种相对通用的解决办法显得很有意义。
 - 工业数据的质量偏低。可能受采集环境、人工干预等因素的影响。
 - 有用的数据较少。这个很好理解，工厂里的机器出故障的时间远远小于机器正常工作的时间。很多情况下，我们面对的可能都是新进的机器，因此可以拿来训练的标签少，进行有监督学习的难度很大。
 - 缺乏领域知识。单纯地依靠基于数据驱动的方法来构建一个通用的模型，其效果往往不够理想，仍然需要引入更多的领域知识。
 
 


