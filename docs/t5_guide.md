<!--
# Copyright (c) 2022, NVIDIA CORPORATION. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
-->

# FasterTransformer T5 Triton Backend

The FasterTransformer T5 implementation are in [t5_guide.md](https://github.com/NVIDIA/FasterTransformer/blob/main/docs/t5_guide.md).

## Table Of Contents
 
- [FasterTransformer T5 Triton Backend](#fastertransformer-t5-triton-backend)
  - [Table Of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Setup Environment](#setup-environment)
    - [How to set the model configuration](#how-to-set-the-model-configuration)
    - [Prepare Triton T5 model store](#prepare-triton-t5-model-store)
      - [From HuggingFace](#from-huggingface)
      - [From Nemo](#from-nemo)
  - [Run Serving on Single Node](#run-serving-on-single-node)
    - [Run serving directly](#run-serving-directly)
    - [Issue request directly](#issue-request-directly)
  - [Run Triton server on multiple nodes](#run-triton-server-on-multiple-nodes)
    - [Prepare Triton model store for multi-node setup](#prepare-triton-model-store-for-multi-node-setup)
    - [Run on cluster with Enroot/Pyxis support](#run-on-cluster-with-enrootpyxis-support)
  - [Run p/prompt tuning on T5](#run-pprompt-tuning-on-t5)
    - [Setup](#setup)
    - [Run 220m NeMo T5](#run-220m-nemo-t5)
    - [Run 3B NeMo T5](#run-3b-nemo-t5)
    - [Run Nemo model converted from HF](#run-nemo-model-converted-from-hf)
  - [Build an entire processing pipeline with Triton](#build-an-entire-processing-pipeline-with-triton)

## Introduction

This document describes how to serve the `T5` model by FasterTransformer Triton backend. This backend is only an interface to call FasterTransformer in Triton. All implementation are in [FasterTransformer repo](https://github.com/NVIDIA/FasterTransformer). There is also a `T5-Encoder` version, that only runs the encoding part of the model and doesn't produce tokens.

## Setup Environment

Follow the guide in [`README.md`](../README.md) to setup the environment and prepare docker image. We assume users already build the docker here.

### How to set the model configuration

In T5 triton backend, the serving configuration is controlled by `config.pbtxt`. We provide an example in `all_models/t5/fastertransformer/config.pbtxt`. It contains the input parameters, output parameters, some other settings like `tensor_para_size` and `model_checkpoint_path`. 

We use the `config.ini` in the `model_checkpoint_path` to control the model hyper-parameters like head number, head size and transformer layers. We also provide an example in `all_models/t5/fastertransformer/1/config.ini` which assume we set the `model_checkpoint_path` as `all_models/t5/fastertransformer/1/`. The `config.ini` will generated by checkpoint converter when user convert model. User can also change the setting to run tests on custom model size to benchmark.  

The following table shows the details of these settings:

* Settings in config.pbtxt

| Classification |              Name               |              Tensor/Parameter Shape              |      Data Type      |                                                                                         Description                                                                                         |
| :------------: | :-----------------------------: | :----------------------------------------------: | :-----------------: | :-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|     input      |                                 |                                                  |                     |                                                                                                                                                                                             |
|                |           `input_ids`           |          [batch_size, max_input_length]          |       uint32        |                                                                                input ids after tokenization                                                                                 |
|                |        `sequence_length`        |                   [batch_size]                   |       uint32        |                                                                             real sequence length of each input                                                                              |
|                |         `runtime_top_k`         |                   [batch_size]                   |       uint32        |                                                                         **Optional**. candidate number for sampling                                                                         |
|                |         `runtime_top_p`         |                   [batch_size]                   |        float        |                                                                       **Optional**. candidate threshold for sampling                                                                        |
|                |  `beam_search_diversity_rate`   |                   [batch_size]                   |        float        |                                             **Optional**. diversity rate for beam search in this [paper](https://arxiv.org/pdf/1611.08562.pdf)                                              |
|                |          `temperature`          |                   [batch_size]                   |        float        |                                                                             **Optional**. temperature for logit                                                                             |
|                |          `len_penalty`          |                   [batch_size]                   |        float        |                                                                           **Optional**. length penalty for logit                                                                            |
|                |      `repetition_penalty`       |                   [batch_size]                   |        float        |                                                                         **Optional**. repetition penalty for logit                                                                          |
|                |          `random_seed`          |                   [batch_size]                   |       uint64        |                                                                           **Optional**. random seed for sampling                                                                            |
|                |      `is_return_log_probs`      |                   [batch_size]                   |        bool         |                                                            **Optional**. flag to return the log probs of generated token or not.                                                            |
|                |        `max_output_len`         |                   [batch_size]                   |       uint32        |                                                                          **Optional**. max output sequence length                                                                           |
|                |          `beam_width`           |                   [batch_size]                   |       uint32        |                                                           **Optional**. beam size for beam search, using sampling if setting to 1                                                           |
|                |        `bad_words_list`         |          [batch_size, 2, word_list_len]          |        int32        |                          **Optional**. List of tokens (words) to never sample. Should be generated with FasterTransformer/examples/pytorch/gpt/utils/word_list.py                           |
|                |        `stop_words_list`        |          [batch_size, 2, word_list_len]          |        int32        |                         **Optional**. List of tokens (words) that stop sampling. Should be generated with FasterTransformer/examples/pytorch/gpt/utils/word_list.py                         |
|                | `prompt_learning_task_name_ids` |                   [batch_size]                   |       uint32        |                                                                  **Optional**. task_name_id for each sequence in one batch                                                                  |
|                |    `request_prompt_lengths`     |                  [batch_size],                   |       uint32        |                               **Optional**. Length of prefix soft prompt embedding. This describes how many tokens of soft prompt embedding in each sentence.                               |
|                |   `request_prompt_embedding`    |  [batch_size, max_prompt_length, hidden_units]   | float/half/bfloat16 | **Optional**. FT will concat them with results of embedding lookup kernel. For prefix soft prompt embedding, the type must be float; while for p/prompt tuning, the type is same to weight. |
|                |           `ia3_tasks`           |                   [batch_size]                   |        int32        |                                                                         **Optional**. select ia3 task per-sequence.                                                                         |
|                |          `top_p_decay`          |                   [batch_size]                   |        float        |                                                                **Optional**. decay values for top_p factual-nucleus sampling                                                                |
|                |           `top_p_min`           |                   [batch_size]                   |        float        |                                                              **Optional**. min top_p values for top p factual-nucleus sampling                                                              |
|                |        `top_p_reset_ids`        |                   [batch_size]                   |       uint32        |                                                    **Optional**. reset ids for reseting top_p values for top p factual-nucleus sampling                                                     |
|     output     |                                 |                                                  |                     |                                                                                                                                                                                             |
|                |          `output_ids`           |           [batch_size, beam_width, -1]           |       uint32        |                                                                              output ids before detokenization                                                                               |
|                |        `sequence_length`        |                   [batch_size]                   |       uint32        |                                                                             real sequence length of each output                                                                             |
|                |         `cum_log_probs`         |             [batch_size, beam_width]             |        float        |                                                                 **Optional**. cumulative log probability of output sentence                                                                 |
|                |       `output_log_probs`        | [batch_size, beam_width, request_output_seq_len] |        float        |                                                      **Optional**. It records the log probability of logits at each step for sampling.                                                      |
|   parameter    |                                 |                                                  |                     |                                                                                                                                                                                             |
|                |       `tensor_para_size`        |                                                  |         int         |                                                                           parallelism ways in tensor parallelism                                                                            |
|                |      `pipeline_para_size`       |                                                  |         int         |                                                                          parallelism ways in pipeline parallelism                                                                           |
|                |           `data_type`           |                                                  |       string        |                                                             infernce data type: fp32 = float32, fp16 = float16, bf16 = bfloat16                                                             |
|                |   `enable_custom_all_reduce`    |                                                  |        bool         |                                                                               use custom all reduction or not                                                                               |
|                |          `model_type`           |                                                  |       string        |                                                                                        must use `T5`                                                                                        |
|                |     `model_checkpoint_path`     |                                                  |       string        |                                                                     the path to save `config.ini` and weights of model                                                                      |


For `T5-Encoder`, the list of inputs and outputs is reduced compared to `T5`. The model parameters stay the same from plain `T5`.

| Classification |          Name          |                         Tensor/Parameter Shape                          |   Data Type    |                          Description                           |
| :------------: | :--------------------: | :---------------------------------------------------------------------: | :------------: | :------------------------------------------------------------: |
|     input      |                        |                                                                         |                |                                                                |
|                |      `input_ids`       |                     [batch_size, max_input_length]                      |     uint32     |                  input ids after tokenization                  |
|                |   `sequence_length`    |                              [batch_size]                               |     uint32     |               real sequence length of each input               |
|                | `is_return_attentions` |                              [batch_size]                               |      bool      |       **Optional**. flag to return the attention scores.       |
|                |      `ia3_tasks`       |                              [batch_size]                               |     int32      |          **Optional**. select ia3 task per-sequence.           |
|     output     |                        |                                                                         |                |                                                                |
|                | `output_hidden_states` |               [batch_size, max_input_length, hidden_size]               | FP32/FP16/BF16 |         embeddings produced by the model's last layer.         |
|                |  `output_attentions`   | [batch_size, num_layers, num_heads, max_input_length, max_input_length] | FP32/FP16/BF16 | **Optional**. Intermediate attention scores for self attention |

* Note that the data type of `input_hidden_state` and `output_hidden_state` are determined by `data_type` of parameter automatically. 

A sample `.pbtxt` file is given in `all_models/t5-encoder/fastertransformer`. Note that `model_checkpoint_path` may point towards a full T5 model, but only the encoder part of the model will be instanciated.

### Prepare Triton T5 model store

Following the guide [#setup](../README.md#setup) to prepare the docker image.

#### From HuggingFace

Download T5 model checkpoint:

```shell
docker run -it --rm --gpus=all --shm-size=1g --ulimit memlock=-1 -v ${WORKSPACE}:${WORKSPACE} -w ${WORKSPACE} ${TRITON_DOCKER_IMAGE} bash
# now in docker
export WORKSPACE=$(pwd)

git lfs clone https://huggingface.co/t5-small
git clone https://github.com/NVIDIA/FasterTransformer.git # To convert checkpoint
python3 FasterTransformer/examples/pytorch/t5/utils/huggingface_t5_ckpt_convert.py \
        -in_file t5-small/ \
        -saved_dir ${WORKSPACE}/all_models/t5/fastertransformer/1/ \
        -inference_tensor_para_size 1
```

We need to convert to format handled by FasterTransformer. 
If you want to run the model with tensor parallel size 4 and pipeline parallel size 2,
you should convert checkpoints with `-inference_tensor_para_size = [tensor_para_size], i.e. -inference_tensor_para_size = 4`. 
We will convert it directly to directory structure which later we'd use as Triton model store.

Then we will get the model weights (`xxx.bin`) and the config file of model (`config.ini`) in the `${WORKSPACE}/all_models/t5/fastertransformer/1/1-gpu/`. The `config.ini` file contains the hyper-parameters of both encoder and decoder. Note that user need to change the `model_checkpoint_path` to `${WORKSPACE}/all_models/t5/fastertransformer/1/1-gpu/`.

#### From Nemo

You can also convert a `.nemo` file with a script provided by FasterTransformer. For example:
```bash
PTYHONPATH="." python3 examples/pytorch/t5/utils/nemo_t5_ckpt_convert.py -i <path_to_nemo_file> -o <path_to_FT_weights> -i_g 1
```

Additionally, you can augment your model with IA3 adapters. For that, you can convert, as a second step, the IA3 nemo file:
```bash
PYTHONPATH="." python3 ./examples/pytorch/t5/utils/nemo_t5_ia3.py -i <path_to_ia3_nemo_file> -o <path_to_FT_weights> -i_g 1
```
Repeat this operation with multiple IA3 nemo files (for different down-stream tasks). Then use the `ia3_tasks` input to select the IA3 weights.

If you prefer to specialize your model for a single IA3 task by pre-multiplying weights, you can use the `--in-place` option of the `nemo_t5_ia3.py` script.

## Run Serving on Single Node

### Run serving directly

Before launching server, we suggest run the gemm test first like what we mention [here](https://github.com/NVIDIA/FasterTransformer/blob/main/docs/t5_guide.md#translation-process). The gemm test program is put at `/workspace/build/fastertransformer_backend/build/bin/t5_gemm`.

```bash
/workspace/build/fastertransformer_backend/build/bin/t5_gemm 8 4 32 512 8 64 2048 512 8 64 2048 32128 0 1 1
CUDA_VISIBLE_DEVICES=0 mpirun -n 1 --allow-run-as-root /opt/tritonserver/bin/tritonserver  --model-repository=${WORKSPACE}/all_models/t5/ &
python3 tools/t5_utils/t5_end_to_end_test.py --batch_size 32
```

The results would be like

```bash
[INFO] ft_triton translates 24 batches taking 8.94 sec to translate 61374 tokens, BLEU score: 27.26, 6862 tokens/sec.
```

* Note: If user encounter `[ERROR] world_size (4) should equal to tensor_para_size_ * pipeline_para_size_ (1 * 1 here)`, please check that the GPU number of your device and set the GPUs you want to use by `CUDA_VISIBLE_DEVICES`. 
* Recommend modifying the SERVER_TIMEOUT of common/util.sh to longer time

### Issue request directly

```bash
python3 ${WORKSPACE}/tools/issue_request.py tools/requests/sample_request_single_t5.json
```
or
```bash
python3 ${WORKSPACE}/tools/issue_request.py tools/requests/sample_request_single_t5_encoder.json
```

## Run Triton server on multiple nodes

### Prepare Triton model store for multi-node setup

For this experiment you need to [prepare Triton t5 model store](#prepare-triton-t5-model-store):
- properly convert the checkpoint to FasterTransformer format
- update Triton model configuration

We do suggest:
- `tensor_para_size` = number of gpus in one node (e.g. 8 for DGX A100)
- `layer_para_size` = number of nodes

Other Triton model configuration parameters should be updated as for single node setup.

Model store should be placed on network file system available for all cluster nodes on which Triton will run.

### Run on cluster with Enroot/Pyxis support

First allocate two nodes:

```bash
salloc -A account_name -t 10:00:00 -N 2
```

Then run the script shown below to start two nodes' server.
-N and -n should be equal to the number of nodes because we start one process per node. If you need to run on two nodes, then -N 2 and -n 2.
Remember to change `tensor_para_size` and `pipeline_para_size` as suggested in [MPI Launching with Tensor Parallel size/ Pipeline Parallel Size Setting](../README.md#mpi-launching-with-tensor-parallel-size-and-pipeline-parallel-size-setting) if you run on multiple nodes. 

```bash
WORKSPACE="/workspace" # the dir you build the docker
IMAGE="github_or_gitlab/fastertransformer/multi-node-ft-triton-backend:latest"
CMD="/opt/tritonserver/bin/tritonserver --model-repository=$WORKSPACE/fastertransformer_backend/all_models/t5"
srun -N 2 -n 2 --mpi=pmix -o inference_server.log \
               --container-mounts /home/account/your_network_shared_space/triton:/workspace \
               --container-name multi-node-ft-triton \
               --container-image $IMAGE \
               bash -c "$CMD"
```

Then, you need to run the server on the background since it will not detach by itself. You can enter and commands `ctrl D` and `bg` or run the script above with `sbatch`.

Next, enter the master triton node (the node where MPI_Rank = 0, normally it is the allocated node with the smallest id) when servers have been started shown in the inference log:

```bash
srun -w master-node-name --overlap \
                         --container-name multi-node-ft-triton \
                         --container-mounts /home/account/your_network_shared_space/triton:/workspace \
                         --pty bash # --overlap may not be needed in your slurm environment
```

Finally, run the client in the master triton node:

```bash
python3 fastertransformer_backend/tools/t5_utils/t5_end_to_end_test.py --batch_size 32
```

You can refer to `inference_server.log` on the login-node for the inference server log.

## Run p/prompt tuning on T5

### Setup

```bash
git clone https://github.com/NVIDIA/NeMo
cd NeMo
git checkout 66c7677cd4a68d78965d4905dd1febbf5385dff3
cd -
sudo pip install ./NeMo[nlp]
```

### Run 220m NeMo T5

The original NeMo models are put in `/home/BigNLP/models/T5/NeMo/220m_p_tuning/megatron_t5_220m_f16.nemo` and `/home/BigNLP/models/T5/NeMo/220m_p_tuning/p_tuning_boolq_t5_220m.nemo`.

The converted FT models from NeMo are put int `all_models/t5/fastertransformer/1/`.

If you use converted FT model, you can skip the step to convert model.

Remember to change the `model_checkpoint_path` of `config.pbtxt` before launching serving.

```bash
# convert NeMo model to FT
rm ./all_models/t5/fastertransformer/1/1-gpu/ -rf
PYTHONPATH=build/_deps/repo-ft-src/:${PYTHONPATH} python3 build/_deps/repo-ft-src/examples/pytorch/t5/utils/nemo_t5_ckpt_convert.py --saved-dir all_models/t5/fastertransformer/1/ --in-file /home/BigNLP/models/T5/NeMo/220m_p_tuning/megatron_t5_220m_f16.nemo  --infer-gpu-num 1 --model-name t5 --prompt-in-file /home/BigNLP/models/T5/NeMo/220m_p_tuning/p_tuning_boolq_t5_220m.nemo --prompt-saved-dir all_models/t5/fastertransformer/1/ -p 1 --vocab-path /home/BigNLP/models/T5/NeMo/220m_p_tuning/vocab.txt

# launch serving
CUDA_VISIBLE_DEVICES=0 mpirun -n 1 --allow-run-as-root /opt/tritonserver/bin/tritonserver  --model-repository=./all_models/t5/ &

# run test
python3 tools/t5_utils/boolq_test.py --batch_size 32 --ckpt_path ./all_models/t5/fastertransformer/1/1-gpu/

python3 tools/t5_utils/boolq_test.py --batch_size 32 --use_request_prompt_embedding --ckpt_path ./all_models/t5/fastertransformer/1/1-gpu/
```

The model is finetuned by 50 epoches, and the accuracy on both FT and NeMo are 77.8%. 

### Run 3B NeMo T5

The original NeMo models are put in `/home/BigNLP/models/T5/NeMo/t5-3b/megatron_t5-tp2--val_los-1.09-step-999999-consumed-samples-2159846144.0.nemo` and `/home/BigNLP/models/T5/NeMo/t5-3b/p_tuning_boolq_t5_3B.nemo`.

The converted FT models from NeMo are put int `/home/BigNLP/models/T5/NeMo/t5-3b/c-models/`.

If you use converted FT model, you can skip the step to convert model.

Remember to change the `model_checkpoint_path` of `config.pbtxt` before launching serving.

```bash
# convert NeMo model to FT
PYTHONPATH=build/_deps/repo-ft-src/:${PYTHONPATH} python3 build/_deps/repo-ft-src/examples/pytorch/t5/utils/nemo_t5_ckpt_convert.py --saved-dir /home/BigNLP/models/T5/NeMo/t5-3b/c-models/ --in-file /home/BigNLP/models/T5/NeMo/t5-3b/megatron_t5-tp2--val_los-1.09-step-999999-consumed-samples-2159846144.0.nemo  --infer-gpu-num 1 --model-name t5 --prompt-in-file /home/BigNLP/models/T5/NeMo/t5-3b/p_tuning_boolq_t5_3B.nemo --prompt-saved-dir /home/BigNLP/models/T5/NeMo/t5-3b/c-models/ -p 1 --vocab-path /home/BigNLP/models/T5/NeMo/220m_p_tuning/vocab.txt

# launch serving
CUDA_VISIBLE_DEVICES=0 mpirun -n 1 --allow-run-as-root /opt/tritonserver/bin/tritonserver  --model-repository=./all_models/t5/ &

# run test
python3 tools/t5_utils/boolq_test.py --batch_size 32 --ckpt_path /home/BigNLP/models/T5/NeMo/t5-3b/c-models/1-gpu

python3 tools/t5_utils/boolq_test.py --batch_size 32 --use_request_prompt_embedding --ckpt_path /home/BigNLP/models/T5/NeMo/t5-3b/c-models/1-gpu
```

The model is finetuned by 5 epoches, and the accuracy on both FT and NeMo are 80.8%. 


### Run Nemo model converted from HF

```bash
rm ./all_models/t5/fastertransformer/1/1-gpu/ -rf
PYTHONPATH=build/_deps/repo-ft-src/:${PYTHONPATH} python3 build/_deps/repo-ft-src/examples/pytorch/t5/utils/nemo_t5_ckpt_convert.py --saved-dir ./all_models/t5/fastertransformer/1/ --in-file /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/nemo_t5v1_1_base_lm_adapt_converted_weights.nemo --infer-gpu-num 1 --tokenizer-model-path /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/t5_tokenizer.model --prompt-in-file /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/p_tuning_boolq_t5_220m.nemo  --prompt-saved-dir ./all_models/t5/fastertransformer/1/  --model-name t5v1_1_base_lm_adapt
# PYTHONPATH=build/_deps/repo-ft-src/:${PYTHONPATH} python3 build/_deps/repo-ft-src/examples/pytorch/t5/utils/nemo_t5_ckpt_convert.py --saved-dir /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/c-models/ --in-file /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/nemo_t5v1_1_base_lm_adapt_converted_weights.nemo --infer-gpu-num 1 --tokenizer-model-path /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/t5_tokenizer.model --prompt-in-file /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/p_tuning_boolq_t5_220m.nemo  --prompt-saved-dir /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/c-models/  --model-name t5v1_1_base_lm_adapt --verbose

CUDA_VISIBLE_DEVICES=0 mpirun -n 1 --allow-run-as-root /opt/tritonserver/bin/tritonserver  --model-repository=./all_models/t5/ &

python3 tools/t5_utils/boolq_test_hf.py --batch_size 32 --ckpt_path /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/c-models/1-gpu/

python3 tools/t5_utils/boolq_test_hf.py --batch_size 32 --use_request_prompt_embedding --ckpt_path /home/BigNLP/models/T5/NeMo/nemo_t5v1_1_base_lm_adapt/c-models/1-gpu # Note that we cannot use request prompt embedding under bf16 because triton server does not support bf16 data type
```

The model is finetuned by 5 epoches, and the accuracy on both FT and NeMo are 61%.

* Note that the `bos` of config.ini may be wrong, need to set start_id to 0 on client side.

## Build an entire processing pipeline with Triton

For T5-Encoder, there exists an example of tokenization in `all_models/t5-encoder/tokenizer`. This python model accepts sentences and convert them to token lists. It can be integrated in a Triton [ensemble model](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/architecture.md#ensemble-models) together with the `fastertransformer` model. You may also consult GPT [documentation](./gpt_guide.md).
