# Pretraining Representations For Data-Efficient Reinforcement Learning

*Max Schwarzer, Nitarshan Rajkumar, Michael Noukhovitch, Ankesh Anand, Laurent Charlin, Devon Hjelm, Philip Bachman & Aaron Courville*

This repo provides code for implementing SGI.

* [📦 Install ](#install) -- Install relevant dependencies and the project
* [🔧 Usage ](#usage) -- Commands to run different experiments from the paper

## Install 
To install the requirements, follow these steps:
```bash
# PyTorch
export LANG=C.UTF-8
# Install requirements
pip install torch==1.8.1+cu102 torchvision==0.9.1+cu102 -f https://download.pytorch.org/whl/torch_stable.html
pip install -r requirements.txt
# Finally, install the project
pip install --user -e .
```

## Usage:
The default branch for the latest and stable changes is `release`. 

0.  Download dataset

```bash
# Install gsutil (https://cloud.google.com/storage/docs/gsutil_install#deb)
sudo apt-get install apt-transport-https ca-certificates gnupg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-cli

# download DQN dataset
mkdir data/
#gsutil -m cp -R gs://atari-replay-datasets/dqn/Seaquest dataset/
#gsutil -m cp -R gs://atari-replay-datasets/dqn/DemonAttack dataset/
```

To run SGI:

```bash
# args are game, seed, and number of dataloader
bash ./scripts/experiments/sgim_pretrain.sh demon_attack 1 8
```

1.  Use the helper script to download and parse checkpoints from the [DQN Replay Dataset](https://research.google/tools/datasets/dqn-replay/); this requires [gsutil](https://cloud.google.com/storage/docs/gsutil_install#install) to be installed. You may want to modify the script to download fewer checkpoints from fewer games, as otherwise this requires significant storage.
    * Or substitute your own pre-training data!  The codebase expects a series of .gz files, one each for observations, actions and terminals.
```bash
bash scripts/download_replay_dataset.sh $DATA_DIR
```
2.  To pretrain with SGI:
```bash
python -m scripts.run public=True model_folder=./ offline.runner.save_every=2500 \
    env.game=pong seed=1 offline_model_save={your model name} \
    offline.runner.epochs=10 offline.runner.dataloader.games=[Pong] \
    offline.runner.no_eval=1 \
    +offline.algo.goal_weight=1 \
    +offline.algo.inverse_model_weight=1 \
    +offline.algo.spr_weight=1 \
    +offline.algo.target_update_tau=0.01 \
    +offline.agent.model_kwargs.momentum_tau=0.01 \
    do_online=False \
    algo.batch_size=256 \
    +offline.agent.model_kwargs.noisy_nets_std=0 \
    offline.runner.dataloader.dataset_on_disk=True \
    offline.runner.dataloader.samples=1000000 \
    offline.runner.dataloader.checkpoints='{your checkpoints}' \
    offline.runner.dataloader.num_workers=2 \
    offline.runner.dataloader.data_path={your data dir} \
    offline.runner.dataloader.tmp_data_path=./ 
```
3. To fine-tune with SGI:
```bash
python -m scripts.run public=True env.game=pong seed=1 num_logs=10  \
    model_load={your_model_name} model_folder=./ \
    algo.encoder_lr=0.000001 algo.q_l1_lr=0.00003 algo.clip_grad_norm=-1 algo.clip_model_grad_norm=-1
```

When reporting scores, we average across 10 fine-tuning seeds.

`./scripts/experiments` contains a number of example configurations, including for SGI-M, SGI-M/L and SGI-W, for both pre-training and fine-tuning.
Each of these scripts can be launched by providing a game and seed, e.g., `./scripts/experiments/sgim_pretrain.sh pong 1`.  These scripts are provided primarily to illustrate the hyperparameters used for different experiments; you will likely need to modify the arguments in these scripts to point to your data and model directories.

Data for SGI-R and SGI-E is not included due to its size, but can be re-generated locally.  Contact us for details.

## What does each file do? 

    .
    ├── scripts
    │   ├── run.py                # The main runner script to launch jobs.
    │   ├── config.yaml           # The hydra configuration file, listing hyperparameters and options.
    |   ├── download_replay_dataset.sh  # Helper script to download the DQN replay dataset.
    |   └── experiments           # Configurations for various experiments done by SGI.
    |   
    ├── src                     
    │   ├── agent.py              # Implements the Agent API for action selection 
    │   ├── algos.py              # Distributional RL loss and optimization
    │   ├── models.py             # Forward passes, network initialization.
    │   ├── networks.py           # Network architecture and forward passes.
    │   ├── offline_dataset.py    # Dataloader for offline data.
    │   ├── gcrl.py               # Utils for SGI's goal-conditioned RL objective.
    │   ├── rlpyt_atari_env.py    # Slightly modified Atari env from rlpyt
    │   ├── rlpyt_utils.py        # Utility methods that we use to extend rlpyt's functionality
    │   └── utils.py              # Command line arguments and helper functions 
    │
    └── requirements.txt          # Dependencies
