```shell
sudo apt update

sudo apt install -y gcc
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
conda create -n llm python=3.12

pip install numpy
pip install torch
pip install sentencepiece
pip install pyyaml
pip install safetensors
pip install transformers
pip install -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple numpy torch sentencepiece pyyaml safetensors transformers
```