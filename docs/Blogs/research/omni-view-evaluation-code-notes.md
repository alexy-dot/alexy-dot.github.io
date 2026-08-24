---
title: Omni-View 测评代码阅读笔记：VSI-Bench 推理流程与 BAGEL 调用链
description: 从 eval_und.sh、evaluate_vsibench.py 到 BAGEL model.chat 的测评推理代码阅读笔记
icon: material/code-braces
comments: true
---

# Omni-View 测评代码阅读笔记

先建立论文整体认识；

学推理；

学评测；

再学训练；

最后把训练设计与论文方法对应起来。

## 代码结构与学习顺序

我按照代码调用链的顺序进行学习，先看测评推理流程：

1. eval_und.sh
   torchrun 启动多个 GPU 进程

2. evaluate_vsibench.py 主流程
   解析参数
   → 初始化多 GPU 通信
   → 加载 model、tokenizer、特殊 token
   → 创建图片预处理器

3. VQADataset
   根据 idx 取出一道题
   → 找到对应视频
   → 均匀抽取 8～32 帧
   → 转换成 PIL 图片列表
   → 调整图片尺寸

4. 构造 prompt
   读取原始 question
   → 根据题型添加选项或回答要求
   → 添加 `<image>` 标记
   → 得到 conversation 字符串

5. DataLoader 和 Sampler
   Sampler 给不同 GPU 分配不同样本编号
   → DataLoader 读取完整样本
   → collate_fn 整理成 batch

6. model.chat()
   图片经过视觉模型
   → 视觉信息进入 past_key_values
   → tokenizer 把问题转换成 token ID
   → 问题信息也进入 past_key_values
   → generate_text() 逐 token 生成回答
   → tokenizer.decode() 转回字符串

7. 保存结果
   模型回答放入 pred
   → 保存到 pred_response
   → 汇总多个 GPU 的 outputs
   → 0 号进程写入 results/vsibench.json

### 1. eval_und.sh

每条命令测评一组任务，共五组，代码

```
torchrun \使用 PyTorch 的分布式方式启动下面的 Python 程序，类似命令python **.py，不过是启动多GPU进程的
  --nproc_per_node=4 \torchrun在当前节点启动 4 个推理进程，通常每个进程对应一张 GPU
  --master_port=12345 \进程通信端口
  -m eval.vlm.eval.vqa.evaluate_vsibench \运行python模块，把/替换成.是因为 Python 把文件夹层级写成了模块路径
  --model-path ./pretrained_model/BAGEL-7B-MoT/ \模型配置文件位置，tokenizer位置
  --safetensor-path model.safetensors \模型主要权重文件
  --datasets vsibench   数据集
```

### 2. evaluate_vsibench.py

输入：

- 数据集名称
- 模型目录
- 权重路径
- 输出目录

主要工作：

1. 解析命令行参数
2. 初始化多 GPU 通信
3. 加载模型、tokenizer 和图片预处理器
4. 调用 evaluate_chat_model()

输出：

- results/vsibench.json

关键限制：

- batch_size 必须为 1

torchrun从 此文件开始执行,直接来看main函数。

```python
parser = argparse.ArgumentParser()#通常为命令行参数解析器
parser.add_argument('--datasets', type=str, default='vsibench')#程序可以接受的参数，default是默认值
parser.add_argument('--out-dir', type=str, default='results')#输出为results
args = parser.parse_args()#读取程序启动时收到的所有命令行参数，并把结果放进 args
if not os.path.exists(args.out_dir):
        os.makedirs(args.out_dir, exist_ok=True)#若没有results文件夹则创建
args.datasets = args.datasets.split(',')#这里在分割数据集名称
torch.distributed.init_process_group(
        backend='nccl',#使用 NVIDIA 的 GPU 通信后端
        world_size=int(os.getenv('WORLD_SIZE', '1')),#总进程数
        rank=int(os.getenv('RANK', '0')),#当前进程的全局编号，分别为 0、1、2、3
        timeout=datetime.timedelta(seconds=12000)#设定报错时间
    )#初始化多GPU通信
torch.cuda.set_device(int(os.getenv('LOCAL_RANK', 0)))#为每个进程指定GPU
model, tokenizer, new_token_ids = load_model_and_tokenizer(args, args.safetensor_path)#加载模型⭐️很重要的一段代码,函数在utils.py实现
    model = model.to(torch.bfloat16)#转化模型参数，精度降低每个数字占2字节，显存减半且GPU能更快进行矩阵运算
    image_transform = build_transform()#自定义函数，在utils.py,用于处理图片,图片预处理工具

    total_params = sum(p.numel() for p in model.parameters()) / 1e9#遍历模型所有参数统计参数量
    print(f'[test] total_params: {total_params}B')

    evaluate_chat_model()#🌟这个函数正式开始测评
```

shell是传输参数，python parser是接受参数，有默认值为默认值除非shell传入其他覆盖默认值

#### 详细解释evaluate_chat_model():推理流程:

VQADataset 定义一条样本
        ↓
InferenceSampler 决定当前 GPU 处理哪些编号
        ↓
DataLoader 根据编号调用 Dataset.__getitem__()
        ↓
collate_fn 整理成一个 batch
        ↓
model.chat() 生成回答
        ↓
当前 GPU 的 outputs 列表
        ↓
all_gather_object 汇总四个 GPU
        ↓
0 号进程保存 results/vsibench.json

```python
def evaluate_chat_model():
    random.seed(args.seed)

    for ds_name in args.datasets:#遍历数据集,args.datasets是["vsibench"]
        dataset = VQADataset(
            test=ds_collections[ds_name]['test'],
        )#根据类VQADataset创建对象,等价于dataset = VQADataset(test="./dataset/eval/VSI-Bench")
        dataloader = torch.utils.data.DataLoader(#创建对象
            dataset=dataset,
            sampler=InferenceSampler(len(dataset)),#给不同GPU分数据/当前进程应该取哪些样本编号
            batch_size=args.batch_size,
            num_workers=args.num_workers,
            pin_memory=True,
            drop_last=False,
            collate_fn=collate_fn,#
        )

        outputs = []#空列表
        for _, (dataset_names, question_ids, question_types, video_ids, questions, images, conversations, answers) in tqdm(enumerate(dataloader)):
            pred = model.chat(#生成回答
                tokenizer,
                new_token_ids,
                image_transform,
                images=images[0], # batch=1
                prompt=conversations[0], # batch=1
                max_length=ds_collections[ds_name]['max_new_tokens'],
            )
            preds = [pred]

            for question, question_id, question_type, pred, answer, video_id in zip(questions, question_ids, question_types, preds, answers, video_ids):
                outputs.append({
                    "dataset": ds_name,
                    "sample_id": question_id,
                    "prompt": question,
                    "pred_response": pred,#这里是回答储存的位置
                    "gt_response": answer,#数据集提供的标准答案
                    "model_id": "BAGEL-7B",
                    "question_type": question_type,
                    "scene": video_id,
                })

        torch.cuda.empty_cache()#清理pytorch里没有继续使用GPU的缓存显存
        torch.distributed.barrier()#所有进程在这里等待

        world_size = torch.distributed.get_world_size()#4
        merged_outputs = [None for _ in range(world_size)]#[None,None,None,None]
        torch.distributed.all_gather_object(merged_outputs, json.dumps(outputs))#json.dumps()把当前进程列表转化为JSON字符串,all_gather_object()收集所有进程的结果

        merged_outputs = [json.loads(_) for _ in merged_outputs]#将字符串恢复为列表
        merged_outputs = [_ for _ in itertools.chain.from_iterable(merged_outputs)]#合并为一个列表

        if torch.distributed.get_rank() == 0:
            print(f'Evaluating {ds_name} ...')
            results_file = f'{ds_name}.json'#文件名
            results_file = os.path.join(args.out_dir, results_file)#与输出目录拼接
            json.dump(merged_outputs, open(results_file, 'w'))#写文件
            print('Results saved to {}'.format(results_file))

        torch.distributed.barrier()
```

### 待确认：

- eval_und.sh 使用 --dataset，但 Python 定义的是 --datasets
- model.safetensors 是否是正确 BAGEL 权重
- image_transform 在 model.chat() 内部如何使用
- 四个 GPU 之间如何分配样本

#### VQADataset

一个场景视频+一个问题+问题类型+可选选项+标准答案

```
class VQADataset(torch.utils.data.Dataset):#定义一个名叫 VQADataset 的类,并继承 PyTorch 的 Dataset 类
    def __init__(self, test):
            self.bench_path = test#数据集目录路径
        self.bench = load_dataset(self.bench_path)["test"]#load_dataset 是 Hugging Face datasets 库提供的函数,此时主要加载数据表和元信息#["test"]：从数据集中选择测试集 split
    def __len__(self):#测试集共有多少条
    def __getitem__(self, idx):
        images = read_video_images(scene_path)#读视频并抽成多张图片
        现均匀的抽取帧数,抽取在8-32范围内
        传入的mp4视频,返回__getitem__()PIL图片
        传回后继续调整尺寸,不改变原比例,缩放为高400,注意这里有一个小问题🍄
```

待确认问题🍄:
PIL.Image.size 返回 (width, height)，
当前 H, W 的赋值可能导致图像比例错误。

##### 将问题加工成模型能看到的prompt

代码先定义了两组题型：

MCA_QUESTION_TYPES = [...]选择题

NA_QUESTION_TYPES = [...]简答题

### 3. bagel.py

```python
    # for evaluation
    @torch.no_grad()#装饰器语法,写在函数上面,每次调用函数都会加这个限制,这里的话是关闭pytorch梯度,因为用于推理而不是推理,有减少显存占用,减少额外计算,防止意外执行反向传播的好处
       def chat(#🌲对应前面的modal.chat()
        self,
        tokenizer,
        new_token_ids,
        image_transform,
        images,
        prompt,
        max_length: int,
        do_sample: bool = False,
        temperature: float = 1.0,
    ):
        device = next(self.parameters()).device#self.parameters()遍历模型参数,self就是bagel模型,next定位第一个参数,再找参数对应的device
                #这里看是不是tensor,是tensor将转移到GPU,但是实际用不到这段代码因为值是整数
        if isinstance(new_token_ids, dict):#这个是在判断是不是字典
            for k, v in new_token_ids.items():
                if torch.is_tensor(v):
                    new_token_ids[k] = v.to(device)
        elif torch.is_tensor(new_token_ids):
            new_token_ids = new_token_ids.to(device)
         # prefill
        past_key_values = NaiveCache(self.config.llm_config.num_hidden_layers)#建立上下文缓存
        newlens = [0]#像是在创建变量,这个是上下文长度,下一行是下一部分内容的位置编号
        new_rope = [0]

        # add images
        for image in images:
            generation_input, newlens, new_rope = self.prepare_vit_images(
                curr_kvlens=newlens,
                curr_rope=new_rope,
                images=[image],
                transforms=image_transform,
                new_token_ids=new_token_ids,
            )
            for k, v in generation_input.items():
                if torch.is_tensor(v):
                    generation_input[k] = v.to(device)
            with torch.amp.autocast("cuda", enabled=True, dtype=torch.bfloat16):#GPU计算尽量使用bfloat16可以提升计算效率
                past_key_values = self.forward_cache_update_vit(past_key_values, '''generation_input表示把字典展开为关键字参数''')#把图片送进视觉模型并更新缓存,🌵这里模型已读取视觉信息即视频内容
                # add text
        generation_input, newlens, new_rope = self.prepare_prompts(#🪴函数会text_ids = tokenizer.encode(prompt)将prompt编码
            curr_kvlens=newlens,
            curr_rope=new_rope,
            prompts=[prompt],#batchsize=1,prompt是之前生成的字符串
            tokenizer=tokenizer,
            new_token_ids=new_token_ids,
        )#🌷generation_input为语言模型可以处理的输入,包含token id,token位置编码等
        for k, v in generation_input.items():
            if torch.is_tensor(v):
                generation_input[k] = v.to(device)
        with torch.amp.autocast("cuda", enabled=True, dtype=torch.bfloat16):
            past_key_values = self.forward_cache_update_text(past_key_values, **generation_input)#让语言模型读取问题，并把问题信息加入原来的缓存
            '''这里执行以后past_key_values= 所有视频帧的视觉信息+ 问题文字信息'''
         # decode
        generation_input = self.prepare_start_tokens(newlens, new_rope, new_token_ids)
        for k, v in generation_input.items():
            if torch.is_tensor(v):
                generation_input[k] = v.to(device)
        with torch.amp.autocast("cuda", enabled=True, dtype=torch.bfloat16):
            unpacked_latent = self.generate_text(#🌿逐个生成问题回答(一串tokenID),根据图片、问题和已经生成的文字→ 预测下一个 token→ 把新 token 加入回答→ 再预测下一个 token
                past_key_values=past_key_values,
                max_length=max_length,#最大回答token数
                do_sample=do_sample,#这里=false,不随机抽取,选概率最高token
                temperature=temperature,
                end_token_id=new_token_ids['eos_token_id'],
                **generation_input,
            )
        output = tokenizer.decode(unpacked_latent[:,0])#0指的是取第一个样本,但是batchsize=1所以无所谓就是取那个
        output = output.split('<|im_end|>')[0].split('<|im_start|>')[1]#删除特殊标记

        return output
```

**完整推理流程已经结束。**
