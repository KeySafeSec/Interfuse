InterFuse: Scalable Detection of Memory Safety Violations via Differential Reasoning
Task Definition
Given a C/C++ function as input, the task is to perform binary classification (0/1), where 1 represents a code fragment containing memory safety vulnerabilities (e.g., UAF, Double-Free, Buffer Overflow) and 0 represents non-vulnerable code. Models are evaluated primarily by F1-score, along with Accuracy and MCC.
Directory Structure
src: Core source code of InterFuse, including the hierarchical context fusion model and GA-based path sampling scripts.
dataset: Placeholder for the D2A and MVD datasets.
output: Results of all experiments, including model checkpoints, prediction files, and the snapvuln_comparison_summary.txt.
parserTool: The static analysis tool based on Tree-sitter for ICFG construction and feature extraction.
paper: The PDF version of the manuscript: "InterFuse: Scalable Detection of Memory Safety Violations via Differential Reasoning and Hierarchical Context Fusion".
Prepare Requirements
Python: 3.9
Pytorch: 1.12.1+cu113
transformers: 4.21.0
networkx: 2.8.5
scikit-learn: 1.1.1
tree-sitter: 0.20.0
Tree-sitter (Building Parser)
If the pre-built library in parserTool/my-languages.so is incompatible with your OS, please rebuild it using the following commands:
cd parserTool
bash build.sh
cd ..
Dataset
Our study utilizes the D2A (Differential Analysis to Attention) dataset and the MVD (Multi-Vulnerability Dataset). We provide processed versions where functions are annotated with execution path information and danger-op embeddings.
The processed datasets can be downloaded from Google Drive and should be placed in the dataset folder.
Data Format
`train.train.jsonl, valid.jsonl, and test.jsonl are stored in JSONLines format. Each line contains:
func: The source code of the function.
target: Label (0 for benign, 1 for vulnerable).
idx: Unique index of the sample.
paths: Sampled execution paths (generated via GA).
Data Statistics
Dataset	#Examples	#Vul	#Non-Vul
D2A	2,670	1,148	1,522
MVD	673	312	361
Train
We use NVIDIA RTX 3090 GPUs for fine-tuning. The following command starts the training using the **UniUniXcoder backbone with Hierarchical Fusion.
python src/run.py \
    --output_dir=./output --model_type=unixcoder \
    --tokenizer_name=microsoft/unixcoder-base \
    --model_name_or_path=microsoft/unixcoder-base \
    --do_train --train_data_file=./dataset/train.jsonl \
    --eval_data_file=./dataset/valid.jsonl \
    --epoch 10 --block_size 512 --train_batch_size 16 --learning_rate 2e-5 \
    --evaluate_during_training --seed 123456
Inference
To evaluate the model on the test set:
python src/run.py \
    --output_dir=./output --model_type=unixcoder \
    --do_eval --do_test \
    --test_data_file=./dataset/test.jsonl \
    --model_name_or_path=./output/checkpoint-best/pytorch_model.bin \
    --eval_batch_size 32
Evaluation
To calculate detailed metrics (Accuracy, Precision, Recall, F1):
python ./src/evaluator.py -a ./dataset/test.jsonl -p ./output/predictions.txt
The script of InterFuse and Variants
Main InterFuse Execution (GA + Hierarchical Fusion)
cd src
bash ./run_interfuse_main.sh
Different Path Sampling Methods (Random vs. Heuristic vs. GA)
bash ./test_sampling_methods.sh
Weight Sensitivity Analysis (GA Fitness Function)
bash ./test_ga_weights.sh
Result
Experimental results on the D2A Test Set (compared with the slicing-based baseline):
Methods	Precision	Recall	F1-Score	Construction Time
SnapVuln	78.4	83.2	80.7	18.7s
InterFuse	83.1	86.4	84.7	1.6s
Additional Note for the dataset/README.md
Inside the dataset folder, you should also place a small file to guide users to the external link:
**File: `dataset/File: dataset/README.md
# Dataset Download Link
Due to the large size of the ICFG graphs and processed path embeddings, please download the datasets from the following Google Drive link:
url?sa=E&source=gmail&q=https://drive.google.com/drive/folders/YOUR_LINK_HERE) and should be placed in the dataset folder.
Data Format
train.jsonl, valid.jsonl, and test.jsonl are stored in JSONLines format. Each line contains:
func: The source code of the function.
target: Label (0 for benign, 1 for vulnerable).
idx: Unique index of the sample.
paths: Sampled execution paths (generated via GA).
Data Statistics
Dataset	#Examples	#Vul	#Non-Vul
D2A	2,670	1,148	1,522
MVD	673	312	361
Train
We use NVIDIA RTX 3090 GPUs for fine-tuning. The following command starts the training using the UniXcoder backbone with Hierarchical Fusion.
python src/run.py \
    --output_dir=./output --model_type=unixcoder \
    --tokenizer_name=microsoft/unixcoder-base \
    --model_name_or_path=microsoft/unixcoder-base \
    --do_train --train_data_file=./dataset/train.jsonl \
    --eval_data_file=./dataset/valid.jsonl \
    --epoch 10 --block_size 512 --train_batch_size 16 --learning_rate 2e-5 \
    --evaluate_during_training --seed 123456
Inference
To evaluate the model on the test set:
python src/run.py \
    --output_dir=./output --model_type=unixcoder \
    --do_eval --do_test \
    --test_data_file=./dataset/test.jsonl \
    --model_name_or_path=./output/checkpoint-best/pytorch_model.bin \
    --eval_batch_size 32
Evaluation
To calculate detailed metrics (Accuracy, Precision, Recall, F1):
python ./src/evaluator.py -a ./dataset/test.jsonl -p ./output/predictions.txt
The script of InterFuse and Variants
Main InterFuse Execution (GA + Hierarchical Fusion)
cd src
bash ./run_interfuse_main.sh
Different Path Sampling Methods (Random vs. Heuristic vs. GA)
bash ./test_sampling_methods.sh
Weight Sensitivity Analysis (GA Fitness Function)
bash ./test_ga_weights.sh
Result
Experimental results on the D2A Test Set (compared with the slicing-based baseline):
Methods	Precision	Recall	F1-Score	Construction Time
SnapVuln	78.4	83.2	80.7	18.7s
InterFuse	83.1	86.4	84.7	1.6s
Additional Note for the dataset/README.md
Inside the dataset folder, you should also place a small file to guide users to the external link:
File: dataset/README.md
# Dataset Download Link
Due to the large size of the ICFG graphs and processed path embeddings, please download the datasets from the following Google Drive link:
[https://drive.google.com/drive/folders/YOUR_LINK_HERE](https://drive.google.com/drive/folders/YOUR_LINK_HERE)

Once downloaded, extract the files so that `train.jsonl` and `test.jsonl` are located directly in this directory.
