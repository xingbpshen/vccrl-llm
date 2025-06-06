<h1 align="center">
Verbalized Confidence Calibration in Large Language Models with Reinforcement Learning
</h1>

<p align="center">
  <a href="https://www.python.org/">
    <img src="https://img.shields.io/badge/python-3.10+-blue.svg" alt="Python Version">
  </a>
  <a href="https://pytorch.org/">
    <img src="https://img.shields.io/badge/PyTorch-2.6.0-red.svg" alt="PyTorch Version">
  </a>
</p>


## 0. Before you start
Make sure you have at least **3** NVIDIA GPUs with adequate memory (memory requirement depends on the scale of the LLM you want to use) if you wish to use open-source downstream LLMs. Otherwise, you can use supported closed-source LLMs (see list below and see [**section 1.3**](#13-modifying-config)) as downstream, which only requires at least **2** GPUs.

Supported closed-source LLMs as downstream:
- `gemini-2.0-flash-001`
- `gpt-4o-mini-2024-07-18`
### 0.1 (Recommended) Downloading open-source LLMs beforehand
We recommend to download the open-source LLMs using `huggingface-cli` before you start (make sure you obtained relevant permissions/agreement to download the models from Hugging Face):
```bash
huggingface-cli login
huggingface-cli download {REPO_NAME} --local-dir {SAVE_FOLDER} --local-dir-use-symlinks False
```
For example, the `{REPO_NAME}` can be `meta-llama/Llama-3.1-8B-Instruct` and `{SAVE_FOLDER}` can be `/usr/local/data/Llama-3.1-8B-Instruct`. The downloaded model will be saved in the specified folder `{SAVE_FOLDER}`.

## 1. Preparation
### 1.1 Installation
It is recommended to use a virtual environment (e.g., `venv`) to avoid package conflicts. Here we assume you are using `venv` as your virtual environment. If you are using `conda`, please adjust the commands accordingly.
```bash
git clone https://github.com/xingbpshen/vccrl-llm.git
cd vccrl-llm/
pip install -r requirements.txt
```
### 1.2 Downloading datasets
Create a `data/` folder under the project root `vccrl-llm/`:
```bash
mkdir data/
```
Download the dataset from the MedMCQA repository [here](https://drive.google.com/uc?export=download&id=15VkJdq5eyWIkfb_aoD3oS8i4tScbHYky). Unzip and put all three `.json` files in the newly created `data/` folder.
### 1.3 Modifying config
The config files are located at `config/`. You can modify the parameters according to your needs. The default config file `medmcqa.yml` is for the MedMCQA dataset.

Here are some important parameters you may want to modify:
- `resources.cache_dir` This is where vLLM and other python packages will be cached. Make sure you have enough space.
- `resources.policy_cuda` This is a string of CUDA devices (e.g., `"3,4"` or `"3"`) used for the policy update/training. Make sure you have enough memory on these devices.
- `resources.action_cuda` This is a string of CUDA devices used for the TRL with vLLM serving to sample "actions" (in the context of reinforcement learning). Make sure you have enough memory on these devices.
- `resources.downstream_cuda` This is a string of CUDA devices used for the downstream LLM (to obtain reward). Make sure you have enough memory on these devices. This field is ignored if you are using closed-source LLMs as downstream.
- `model.policy` This is the model name. You can use any repository name supported by Hugging Face or a path to a local model (e.g., `"meta-llama/Llama-3.1-8B-Instruct"` or `"/usr/local/data/Llama-3.1-8B-Instruct"`).
- `model.downstream` This is the model name. You can use any repository name supported by Hugging Face or a path to a local model, or a closed-source model such as `gemini-2.0-flash-001`.
- `api_key.openai` and `api_key.google` If you specified a closed-source LLM as downstream in `model.downstream`, you need to provide the API key for the model. You can obtain the API key from the respective provider. If you are using open-source LLMs, you can leave these fields empty.

Please note that `resources.policy_cuda`, `resources.action_cuda`, and `resources.downstream_cuda` **must not include any overlapping device** to avoid CUDA initialization error.

## 2. Training
### 2.1 About vLLM serving
To enable TRL with vLLM serving, we need to start **2** (or **1** if you are using closed-source LLM as downstream) servers: one for the policy model (to sample action) and one for the downstream LLM to calculate reward.

These servers will be started automatically so you do not need to do anything now.
### 2.2 Training the policy model
By default, the policy model will be trained with GRPO using TRL support. Run the following command to start training:
```bash
python main.py --config {DATASET}.yml --log_folder {LOG_FOLDER} --trial_name {TRIAL_NAME} --train --ni
```
Running the above command once will start:
- **2** (or **1**) detached subprocesses for vLLMs, each corresponding to one of the servers. You can observe the GPU memory usage increasing in the terminal. You can use `nvidia-smi` to check the GPU memory usage for your specified CUDA devices `resources.action_cuda` and `resources.downstream_cuda`.
- **1** foreground engine subprocess for TRL, which will be responsible for the training of the policy model. You can observe the GPU memory usage (on your specified CUDA devices `resources.policy_cuda`) increasing in the terminal.

Runtime related logs will be saved in `{LOG_FOLDER}/{TRIAL_NAME}/` folder.

## 3. Evaluation
Run the following command to evaluate the trained policy model:
```bash
python main.py --config {DATASET}.yml --log_folder {LOG_FOLDER} --trial_name {TRIAL_NAME} --test --ni 
```

To evaluate the entropy-based consistency metric, add the flag `--entropy`.

Please test it out and let me know (by raising a GitHub issue) if you encounter any issue.