镇楼！
===
![企业微信截图_16271339771728](https://user-images.githubusercontent.com/29834982/126888030-4dd10af5-dccb-4124-8560-95ab63940c5a.png)


std: 方差的意思, 反应镀膜过程中, 机器的工作平稳状态

Model介绍:
===

model1:
---
10层可调thickness 到 sensor ACT_O1_QCMS_THICKNESS_CH1 std 值的映射关系.
[ACT_O1_QCMS_THICKNESS_CH1 std 的维度不等, 因数据清洗的周期性而变化, 我们以一个清洗周期内的所有数据为准]


model2： 
---
14维 ACT_O1_QCMS_THICKNESS_CH1 std 到 81维 lab曲线的映射关系
[为什么14? 因为本批次产品为正背7层膜产品, 分为8个步骤镀膜. 除去step1过程的std值恒为0, 剩下的7个step均有std响应值]



算法流程:
===

model2 part: 
---
1. 运行 data_clean_cycle.py 脚本, 生成各个 data cycle 内的炉号组合, 将其记录在 './part_clean_number{}.txt'中.
2. 使用全量数据训一个base model2.pth, 再针对各个data cycle, 用各个 data cycle 内的所有数据, fine-tune model2.pth. 得到 model2_{}.pth
[run mm_Model2_data_cycle.py 完成.] 
[为什么要得到base model2.pth后再fine-tune? 因为model2也是需要区分 data 周期性的. 但各个 data cycle 内的数据量太少, fine-tune才能训出拟合比较好的model2_{}.pth]
3. 测试阶段, 输入待测试样本的真实std值, 得到各个model2_{}.pth对应的lab结果, 与测试数据的真实lab值计算loss, 选取loss最小的model2_{}.pth 作为最接近状态.
4. 基于挑选出的model2_{}.pth, 进行微调输入端std值, 达到优化lab曲线的目的.
[依次运行 model2.py flag == 0,2, 0阶段为lr,epoch等超参搜索阶段, 2阶段为最佳参数配置下的微调过程]
5. 结束微调过程,记录下模型训练最佳时候, 输入端的值. [即期望得到的lab曲线，所对应的输入std值. 落盘存为'./modified_std.npy', 后续model1阶段需要使用.]

model1 part:
---
1. 基于'./part_clean_number{}.txt', run mm_Model1_data_cycle.py 得到基于各个 data cycle 的model1_{}.pth 
[不同于model2, 不需要base model1.pth 再 fine-tune.]
2. 测试阶段, 输入待测试样本的膜厚设置值, 得到各个model1_{}.pth对应的std值结果, 与测试数据的真实std值计算loss, 选取loss最小的model1_{}.pth 作为最接近状态.
3. 基于挑选出的model1_{}.pth, 将以上model2中得到的'./modified_std.npy', 赋值到model1的输出端, 通过微调输入端的膜厚值, 得到我们想要的,可使lab曲线更优的,std值.
5. 微调阶段可依次运行 model1.py flag == 0,2, 0阶段为lr,epoch等超参搜索阶段, 2阶段为最佳参数配置下的微调过程


附:
===
为什么model1突然work了?
---
剔除正背面的第一个std值, 这俩都是0, 没有拟合必要, 则std从16维变到14维. model1得区分数据清洗周期[data cycle]. 后面发现model2也需要去扽周期性.. [麻烦哦..]

scale 统计值
===
modle1,2的输入都需要进行data normalization. 统计量mean和std, 应该用全量数据的mean,std(包括thickness,sensor std), 而不是每个 data cycle 内各自的 mean,std. 

超参搜索
===
1. 针对model1,2的微调阶段, 可对 lr 和 epoch 进行超参搜索, 记录下最佳lr下最小loss对应的训练epoch次数. 落盘到临时文件. 
2. 将此套超参设置到模型中, 再run一遍模型, 即可得到最佳的输入端偏移量, 自动化优化出更好的lab曲线和sensor修改值. 

部署流程: 
---
run Test.py 中的 modify_std() 和 modify_thickness(). 
test_model1_and_model2(flag=1/2) 为model1+2的正向验证过程, 即通过膜厚设置值, 验证可真实得到测试数据的lab曲线.


项目总结
===
1. 了解工艺背景，熟悉数据，建模需要对数据敏感，抽哪些数据特征作为输入，模型的输出是什么，拟合几个模型？
2. bug free很重要, 不然一直不顺一直调模型换思路, 很崩溃. 要注意代码细节! 
3. 工艺优化项目, 需要注意 机器状态 这个隐含状态[也即机器产生的数据可能存在周期性]. 本蔡司案例, 拆分出的 model1,2 均受到周期性的影响.


实验记录:
===
run Test.py 
1. 正向验证, model12均需要用测试数据fine-tune最佳model12
2. 反向验证, model12均不需要用测试数据fine-tune最佳model12
3. 
