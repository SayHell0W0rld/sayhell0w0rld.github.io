---
title: Stable Diffusion 体验：从零到能写提示词
date: 2022-12-15 10:00:00
tags:
  - AI
  - Stable Diffusion
  - 图像生成
categories:
  - 技术实践
---

## 前言


2022年8月，Stable Diffusion发布当天我就装上了。那时候还没有现在这么多教程和插件，官方release里就一个Python脚本和一行命令。我跑的第一张图是一只猫，用了默认提示词，出来的效果一般——毛发糊成一团，背景乱七八糟。但那个瞬间很明确：这东西能用。

<!-- more -->
后来几个月我花了不少时间研究提示词怎么写，怎么调，踩了不少坑。这篇记录一下当时的理解和实践经验。

## Stable Diffusion的基本工作方式

简单说：你写一段文字描述（prompt），模型把这段文字编码成语义向量，然后用这个向量引导扩散过程（diffusion process）一步步去噪，最终生成一张图片。

所以提示词的质量直接决定了出图质量。模型不理解你的意图，它只看你写了什么词，然后在训练数据里找最接近的分布去生成。

## 提示词的基本结构

一个好的提示词通常由这几类关键词组成：

1. **主体（Subject）**：你想画什么。越具体越好。"a sorceress"和"Emma Watson as a powerful mysterious sorceress, casting lightning magic, detailed clothing"出来的结果完全不同。
2. **画风（Medium）**：用什么媒介。digital painting、oil painting、photography，这些对最终风格影响很大。
3. **风格（Style）**：艺术流派。hyperrealistic、fantasy、art nouveau等等。
4. **画家/平台（Artist/Website）**：引用特定画家或平台的视觉风格。比如"by Alphonse Mucha"、"trending on artstation"。
5. **分辨率/细节（Resolution）**：highly detailed、sharp focus、8k。
6. **色调和光照（Color/Lighting）**：iridescent gold、cinematic lighting、rim lighting。

一个完整的提示词可能是这样的：

```
Emma Watson as a powerful mysterious sorceress, casting lightning magic,
detailed clothing, digital painting, hyperrealistic, fantasy, Surrealist,
full body, by Stanley Artgerm Lau and Alphonse Mucha, artstation,
highly detailed, sharp focus, iridescent gold, cinematic lighting
```

不需要每类都填。画家、网站和风格之间有重叠，根据需要组合就行。

## 提示词顺序很重要

写提示词的时候，顺序代表优先级。靠前的词权重大于靠后的词。所以最重要的内容要放在前面。

比如你想画一个戴帽子的女孩，"1girl, hat"比"hat, 1girl"的效果更稳定——因为"1girl"锚定了主体，"hat"作为附加元素叠加。

模型的标准token限制是75个。超过75的部分可能被截断。AUTOMATIC1111的WebUI会自动把提示词分成75-token一组处理，但如果一个概念（比如"blue hair"）被切到两个组里，效果就会出问题。可以用BREAK关键词手动分割。

## 负面提示词：告诉模型不要画什么

负面提示词（Negative Prompt）的作用是引导采样过程远离特定特征。不是把元素从画面里移除，而是在无条件采样阶段让模型往相反的方向走。

四个主要用途：

1. **移除不想要的物体**：比如脸上有胡子，负面提示词加mustache。想更狠一点就加权重：(mustache:1.3)。
2. **微调已生成的图片**：想减少飘动的头发，负面提示词加windy。
3. **关键词切换**：采样中途切换负面提示词内容，比如the: (ear:1.9): 0.5。
4. **改变整体风格**：负面提示词加blur让画面更清晰，加painting或cartoon让画面更写实。

通用的负面提示词模板：

```
ugly, tiling, poorly drawn hands, poorly drawn feet, poorly drawn face,
out of frame, extra limbs, disfigured, deformed, body out of frame,
bad anatomy, watermark, signature, cut off, low contrast,
underexposed, overexposed, bad art, beginner, amateur,
distorted face, blurry, draft, grainy
```

## 权重控制：微调每个词的影响

提示词里每个关键词都可以调整权重。

**WebUI语法：**
- `(keyword:1.5)` — 加强keyword的效果，1.5倍
- `(keyword:0.5)` — 减弱keyword的效果，0.5倍
- `(keyword)` — 等价于 `(keyword:1.1)`，加强10%
- `[keyword]` — 等价于 `(keyword:0.9)`，减弱10%
- `((keyword))` — 1.1 × 1.1 = 1.21倍

权重的范围一般是0.1到100，但实际用到超过2倍就很少见了。太高的权重容易导致画面崩坏。

**注意：WebUI和NovelAI的语法不同。** WebUI用()是加强（1.1倍），NovelAI用{}是加强（1.05倍）。不要混用。

## 提示词混合：在同一张图里融合两个概念

有一种比较高级的技巧：在采样过程中途切换关键词。

语法：`[keyword1 : keyword2 : factor]`

比如 `[apple : fire : 0.5]` — 前一半步数生成苹果的特征，后一半步数生成火焰的特征。最终出来的图是两者的混合体。

这里有个关键点：**第一个关键词决定整体构图**。第二个关键词只影响细节。所以如果想保持某种构图，第一个词要写清楚。

这个技巧也可以用来融合人脸。比如：

```
(Emma Watson:0.5), (Tara Reid:0.9), (Ana de Armas:1.2)
```

多个名人名字加权重混合，生成一张具有混合特征的脸。

## 提示词矩阵和调度

**Prompt Matrix（提示词矩阵）：** 用|分隔关键词，WebUI会自动生成所有关键词组合的图片。比如：

```
a busy city street | illustration | cinematic lighting
```

会生成4张图：单独street、street+illustration、street+cinematic lighting、street+illustration+cinematic lighting。适合快速测试不同关键词的效果。

**Prompt Schedule（步数调度）：** 在采样过程中动态改变提示词。

语法：`[from : to : when]`

- `[fantasy : cyberpunk : 16]` — 第16步之前用fantasy，之后切换为cyberpunk
- `[to : when]` — 到when步之后才加入to
- `[from :: when]` — 到when步之后移除from

## 我遇到的问题

1. **风格冲突**：有些标签本身自带固定风格。比如loli标签有特定的画风，加flatcolor会冲突。解决办法是降低冲突标签的权重，或者直接换掉。

2. **训练数据的隐含关联**：blue eyes经常生成白人面孔，因为训练数据里蓝眼睛和白人的关联很强。brown eyes则能生成更多种族多样性。这个在写提示词的时候要注意。

3. **模型差异**：换了checkpoint模型之后，之前好用的风格关键词可能完全失效。每个模型的训练数据不同，对同一个词的理解也不同。换了模型一定要重新测试关键词效果。

4. **过多关键词反而不好**：不是写得越多越好。堆太多词会稀释每个词的效果，模型会随机从一堆词里选。与其写20个词，不如精选5-6个最准确的词。

## 关于Danbooru标签体系

SD的早期版本（尤其是基于NovelAI训练的模型）大量使用Danbooru的标签体系。Danbooru是一个动画图片托管网站，有标准化的标签系统。这些标签被直接用于模型训练，所以用Danbooru标签来描述画面效果往往很精确。

比如：
- `1girl` — 一个女孩
- `solo` — 画面只有一个人
- `long hair, blue eyes` — 长发、蓝眼
- `looking at viewer` — 看向观众
- `outdoors, beach` — 室外、海滩

这套标签体系的好处是标准化程度高，模型对每个标签的理解比较一致。缺点是需要专门学习标签词汇，不太适合用自然语言描述。

后来AUTOMATIC1111有了Tag Autocomplete插件，输入时会自动补全标签。还有人做了中文翻译版的标签库，降低了上手门槛。

## 一些实用工具

- **Danbooru标签超市** (tags.novelai.dev)：标签浏览和组合工具，支持标签权重调整和混合组配置。
- **Tag Autocomplete插件**：在WebUI里自动补全标签，支持中文翻译。
- **MagicPrompt**：基于GPT-2训练的提示词补全工具，输入几个关键词就能扩展成完整提示词。

## 最后

写提示词这件事没有标准答案。模型在变，工具在变，写法也在变。核心思路是：明确你想要什么，用模型能理解的词汇去描述它，然后通过权重和负面提示词去微调。

我现在回头看早期写的那些提示词，很多都很粗糙。但那时候没有教程，就是一次次试错，看效果，调参数，再试。这个过程本身挺有意思的。
