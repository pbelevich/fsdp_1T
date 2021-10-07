# Run fairseq-train with SLURM srun/sbatch

Clone this repo
```bash
git clone https://github.com/pbelevich/fair_seq_fsdp_slurm.git
cd fair_seq_fsdp_slurm
```
Create conda env
```bash
conda create -yn fairseq_fsdp python=3.8
conda activate fairseq_fsdp

conda install pytorch cudatoolkit=11.1 -c pytorch -c nvidia
```
Clone and install fairscale from source
```bash
git clone git@github.com:facebookresearch/fairscale.git
cd fairscale
pip install -e .
```
Clone and install pbelevich/fairseq from branch `fsdp_1T_2`
```bash
git clone -b fsdp_1T_2 git@github.com:pbelevich/fairseq.git pbelevich-fairseq
cd pbelevich-fairseq
pip install -e .
```
Install deepspeed
```bash
pip install deepspeed
```
Clone and build NVIDIA/apex
```bash
git clone https://github.com/NVIDIA/apex
cd apex
pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" \
  --global-option="--deprecated_fused_adam" --global-option="--xentropy" \
  --global-option="--fast_multihead_attn" ./
```
[No need if you use fsdp_1T_2@pbelevich/fairseq] Quick fix [fairseq-deepspeed issue](https://github.com/pytorch/fairseq/issues/3810):
Open fairseq/optim/cpu_adam.py and add `, False` to [the line 116](https://github.com/pytorch/fairseq/blob/1f7ef9ed1e1061f8c7f88f8b94c7186834398690/fairseq/optim/cpu_adam.py#L116)

[Preprocess the data for RoBERTa](https://github.com/pytorch/fairseq/blob/master/examples/roberta/README.pretraining.md#1-preprocess-the-data)
```bash
wget https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-103-raw-v1.zip
unzip wikitext-103-raw-v1.zip
```
```bash
mkdir -p gpt2_bpe
wget -O gpt2_bpe/encoder.json https://dl.fbaipublicfiles.com/fairseq/gpt2_bpe/encoder.json
wget -O gpt2_bpe/vocab.bpe https://dl.fbaipublicfiles.com/fairseq/gpt2_bpe/vocab.bpe
for SPLIT in train valid test; do \
    python -m examples.roberta.multiprocessing_bpe_encoder \
        --encoder-json gpt2_bpe/encoder.json \
        --vocab-bpe gpt2_bpe/vocab.bpe \
        --inputs wikitext-103-raw/wiki.${SPLIT}.raw \
        --outputs wikitext-103-raw/wiki.${SPLIT}.bpe \
        --keep-empty \
        --workers 60; \
done
```
```bash
wget -O gpt2_bpe/dict.txt https://dl.fbaipublicfiles.com/fairseq/gpt2_bpe/dict.txt
fairseq-preprocess \
    --only-source \
    --srcdict gpt2_bpe/dict.txt \
    --trainpref wikitext-103-raw/wiki.train.bpe \
    --validpref wikitext-103-raw/wiki.valid.bpe \
    --testpref wikitext-103-raw/wiki.test.bpe \
    --destdir data-bin/wikitext-103 \
    --workers 60
```
Run fairseq-train with SLURM sbatch (output to the file slurm-XXXXX.out)
```bash
sbatch fairseq_fsdp_sbatch.sh
```
To see the log:
```bash
tail -f -n +1 slurm-XXXXX.out
```
Run fairseq-train with SLURM srun (output to the screen)
```bash
./fairseq_fsdp_interactive.sh
```
