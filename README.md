# 多模态具身智能大模型（OpenVLA）复现与优化实践

以 GPT 为代表的 Decoder-Only 架构大模型在泛化性、zero-shot 能力上取得巨大进展，随后的 [Clip](https://github.com/openai/CLIP.git)、[LLaVA](https://github.com/haotian-liu/LLaVA.git) 的等多模态的工作让大模型有了理解世界的能力，使得使用 VLM 控制机器人成为可能。

我深入研究视觉-语言-动作（VLA）领域最新的模型 [OpenVLA](https://github.com/openvla/openvla) ，通过全面复现、创新微调与优化部署，实现机器人智能控制系统提升：

1. **精准复现与仿真验证**：采用 PyTorch 框架精准复现 [OpenVLA](https://github.com/openvla/openvla) 模型架构，搭建起包含视觉感知、语言指令和动作规划的完整端到端训练与验证平台，成功验证模型在复杂环境中的泛化能力与鲁棒性，具备出色的零样本迁移性能。
2. **高效的参数微调方案**：针对预训练数据与 [LIBERO](https://libero-project.github.io/datasets) 任务数据模态差异大、数据规模有限的挑战，运用 HuggingFace 工具链设计了 LoRA 低秩微调策略。通过冻结原主干网络，仅优化极少量的低秩适配参数（0.8%），使任务准确率提升23%。
3. **高效云端部署与量化优化**：探索工程部署方案，将模型通过 INT8 量化技术压缩至原大小的 28% ，部署到阿里云GPU服务器（A10），在保持 92.3% 原始精度基础上，推理速度提升了 3.2 倍 ，大幅降低推理延迟，展现出良好的实际应用潜力。



## 实际效果展示(以 LIBERO-Spatial为例)

|                            测试1                             |                            测试2                             |                            测试3                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![19](resources/19--successTrue--taskpick_up_the_black_bowl_on_the_wooden_cabinet_and_p.gif) | ![2](resources/2--successTrue--taskpick_up_the_black_bowl_between_the_plate_and_the_r.gif) | ![3](resources/3--successTrue--taskpick_up_the_black_bowl_next_to_the_ramekin_and_pla.gif) |
|       拿起放在木质橱柜上的那个黑色碗，并将它放到盘子中       |       拿起放在小烤碗旁边的那个黑色碗，并将它放在盘子中       |           拿起小烤碗旁边的黑色碗，并把它放到盘子中           |
|                          **测试4**                           |                          **测试5**                           |                          **测试6**                           |
| ![14](resources/14--successTrue--taskpick_up_the_black_bowl_next_to_the_cookie_box_and_.gif) | ![8](resources/8--successTrue--taskpick_up_the_black_bowl_on_the_cookie_box_and_place.gif) | ![10](resources/10--successTrue--taskpick_up_the_black_bowl_in_the_top_drawer_of_the_wo.gif) |
|           拿起饼干盒旁边的黑色碗，并将它放到盘子中           |          拿起放在饼干盒上的黑色碗，并把它放到盘子中          |      拿起木质橱柜最上面抽屉里的黑色碗，并把它放到盘子中      |



以上的六个测试用例都是成功的测试用例，接下来再展示几个失败的

|                          失败测试1                           |                          失败测试2                           |                          失败测试3                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![7](resources/7--successFalse--taskpick_up_the_black_bowl_on_the_cookie_box_and_place.gif) | ![15](resources/15--successFalse--taskpick_up_the_black_bowl_on_the_stove_and_place_it_o.gif) | ![20](resources/20--successFalse--taskpick_up_the_black_bowl_on_the_wooden_cabinet_and_p.gif) |
|          拿起放在饼干盒上的黑色碗，并将它放到盘子中          |             拿起炉子上的黑色碗，并将它放到盘子中             |           拿起木质橱柜上的黑色碗，并将它放到盘子中           |
| **失败原因**：虽然成功的放到了盘子中，但是模型没有停止输出，导致仿真环境的 Step 达到最大次数，发生了截断 | **失败原因**：对炉子上的黑色碗的位置判断不够准确，机械臂还没有移动到准确的位置时，就执行了抓取动作 | **失败原因**：虽然成功的放到了盘子中，但是模型没有停止输出，导致仿真环境的 Step 达到最大次数，发生了截断 |



测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试







## 数据集 LIBERO

LIBERO，Lifelong Robot Learning Benchmark，是一个专为终身机器人学习研究设计的基准数据集，旨在促进机器人在长期学习过程中知识转移的研究。 

#### 数据集内容：

- **图像数据**：包括来自工作区和手腕相机的RGB图像，提供机器人视觉感知所需的信息。 

- **本体感觉数据**：记录机器人的关节状态、末端执行器的位置和方向等，帮助机器人了解自身状态。 

- **语言任务规范**：为每个任务提供语言描述，明确任务目标和要求，辅助机器人理解需要完成的具体操作。 



#### 任务套件：

- **LIBERO-Spatial**：包含10个任务，侧重于物体空间位置的变化，研究机器人对空间关系的理解和适应能力。 
- **LIBERO-Object**：包含10个任务，主要关注操作对象的变化，例如不同形状、大小或类型的物体，以考察机器人对不同物体的操作和认知能力。 
- **LIBERO-Goal**：包含10个任务，着重于任务目标的改变，检验机器人在不同目标下的规划和执行能力。 
- **LIBERO-100**：由100个任务组成，其中LIBERO-90和LIBERO-10可分别用于预训练和评估长期学习性能，涵盖更广泛的任务类型和变化，全面评估机器人的终身学习能力。 



如果你想了解更多关于 LIBERO 数据集，请参考：https://libero-project.github.io/datasets



## 数据集格式 RLDS









