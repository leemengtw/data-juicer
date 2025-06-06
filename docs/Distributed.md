# Distributed Data Processing in Data-Juicer

## Overview

Data-Juicer supports large-scale distributed data processing based on [Ray](https://github.com/ray-project/ray) and Alibaba's [PAI](https://www.aliyun.com/product/bigdata/learn).

With a dedicated design, almost all operators of Data-Juicer implemented in standalone mode can be seamlessly executed in Ray distributed mode. We continuously conduct engine-specific optimizations for large-scale scenarios, such as data subset splitting strategies that balance the number of files and workers, and streaming I/O patches for JSON files to Ray and Apache Arrow.

For reference, in our experiments with 25 to 100 Alibaba Cloud nodes, Data-Juicer in Ray mode processes datasets containing 70 billion samples on 6400 CPU cores in 2 hours and 7 billion samples on 3200 CPU cores in 0.45 hours. Additionally, a MinHash-LSH-based deduplication operator in Ray mode can deduplicate terabyte-sized datasets on 8 nodes with 1280 CPU cores in 3 hours.

More details can be found in our paper, [Data-Juicer 2.0: Cloud-Scale Adaptive Data Processing for Foundation Models](arXiv_link_coming_soon).

<img src="https://img.alicdn.com/imgextra/i2/O1CN01EteoQ31taUweAW1UE_!!6000000005918-2-tps-4034-4146.png" align="center" width="600" />

## Implementation and Optimizations

### Ray Mode in Data-Juicer

- For most implementations of Data-Juicer [operators](Operators.md), the core processing functions are engine-agnostic. Interoperability is primarily managed in [RayDataset](../data_juicer/core/ray_data.py) and [RayExecutor](../data_juicer/core/ray_executor.py), which are subclasses of the base `DJDataset` and `BaseExecutor`, respectively, and support both Ray [Tasks](https://docs.ray.io/en/latest/ray-core/tasks.html) and [Actors](https://docs.ray.io/en/latest/ray-core/actors.html).
- The exception is the deduplication operators, which are challenging to scale in standalone mode. We provide these operators named as [`ray_xx_deduplicator`](../data_juicer/ops/deduplicator/).

### Subset Splitting

When dealing with tens of thousands of nodes but only a few dataset files, Ray would split the dataset files according to available resources and distribute the blocks across all nodes, incurring huge network communication costs and reduces CPU utilization. For more details, see [Ray's autodetect_parallelism](https://github.com/ray-project/ray/blob/2dbd08a46f7f08ea614d8dd20fd0bca5682a3078/python/ray/data/_internal/util.py#L201-L205) and [tuning output blocks for Ray](https://docs.ray.io/en/latest/data/performance-tips.html#tuning-output-blocks-for-read).

This default execution plan can be quite inefficient especially for scenarios with large number of nodes. To optimize performance for such cases, we automatically splitting the original dataset into smaller files in advance, taking into consideration the features of Ray and Arrow. When users encounter such performance issues, they can utilize this feature or split the dataset according to their own preferences. In our auto-split strategy, the single file size is set to 128MB, and the result should ensure that the number of sub-files after splitting is at least twice the total number of CPU cores available in the cluster. The corresponding tool can be obtained in tools/data_resplit.py.

### Streaming Reading of JSON Files

Streaming reading of JSON files is a common requirement in data processing for foundation models, as many datasets are stored in JSONL format and in huge sizes. 
However, the current implementation in Ray Datasets, which is rooted in the underlying Arrow library (up to Ray version 2.40 and Arrow version 18.1.0), does not support streaming reading of JSON files.

To address the lack of native support for streaming JSON data, we have developed a streaming loading interface and contributed an in-house [patch](https://github.com/modelscope/data-juicer/pull/515) for Apache Arrow ([PR to the repo](https://github.com/apache/arrow/pull/45084)). This patch helps alleviate Out-of-Memory issues. With this patch, Data-Juicer in Ray mode will, by default, use the streaming loading interface to load JSON files. 
Besides, streaming-read support for CSV and Parquet files is already enabled.


### Deduplication

An optimized MinHash-LSH-based Deduplicator is provided in Ray mode. We implement a multiprocess Union-Find set in Ray Actors and a load-balanced distributed algorithm, [BTS](https://ieeexplore.ieee.org/document/10598116), to complete equivalence class merging. This operator can deduplicate terabyte-sized datasets on 1280 CPU cores in 3 hours. Our ablation study shows 2x to 3x speedups with our dedicated optimizations for Ray mode compared to the vanilla version of this deduplication operator.

## Performance Results

### Data Processing with Varied Scales

We conducted experiments on datasets with billions of samples. We prepared a 560k-sample multimodal dataset and expanded it by different factors (1x to 125000x) to create datasets of varying sizes. The experimental results, shown in the figure below, demonstrate good scalability.

![Overview](https://img.alicdn.com/imgextra/i3/O1CN01JV8wcC1oxn0G2xnBT_!!6000000005292-0-tps-1328-1742.jpg)

### Distributed Deduplication on Large-Scale Datasets

We tested the MinHash-based RayDeduplicator on datasets sized at 200GB, 1TB, and 5TB, using CPU counts ranging from 640 to 1280 cores. As the table below shows, when the data size increases by 5x, the processing time increases by 4.02x to 5.62x. When the number of CPU cores doubles, the processing time decreases to 58.9% to 67.1% of the original time.

| # CPU   | 200GB Time | 1TB Time  | 5TB Time   |
|---------|------------|-----------|------------|
| 4 * 160 | 11.13 min  | 50.83 min | 285.43 min |
| 8 * 160 | 7.47 min   | 30.08 min | 168.10 min |

## Quick Start

Before starting, you should install Data-Juicer and its `dist` requirements:

```shell
pip install -v -e .  # Install the minimal requirements of Data-Juicer
pip install -v -e ".[dist]"  # Include dependencies on Ray and other distributed libraries
```

Then start a Ray cluster (ref to the [Ray doc](https://docs.ray.io/en/latest/ray-core/starting-ray.html) for more details):

```shell
# Start a cluster as the head node
ray start --head

# (Optional) Connect to the cluster on other nodes/machines.
ray start --address='{head_ip}:6379'
```

We provide simple demos in the directory `demos/process_on_ray/`, which includes two config files and two test datasets.

```text
demos/process_on_ray
├── configs
│   ├── demo.yaml
│   └── dedup.yaml
└── data
    ├── demo-dataset.json
    └── demo-dataset.jsonl
```

> [!Important]
> If you run these demos on multiple nodes, you need to put the demo dataset to a shared disk (e.g. NAS) and export the result dataset to it as well by modifying the `dataset_path` and `export_path` in the config files.

### Running Example of Ray Mode

In the `demo.yaml` config file, we set the executor type to "ray" and specify an automatic Ray address.

```yaml
...
dataset_path: './demos/process_on_ray/data/demo-dataset.jsonl'
export_path: './outputs/demo/demo-processed'

executor_type: 'ray'  # Set the executor type to "ray"
ray_address: 'auto'  # Set an automatic Ray address
...
```

Run the demo to process the dataset with 12 regular OPs:

```shell
# Run the tool from source
python tools/process_data.py --config demos/process_on_ray/configs/demo.yaml

# Use the command-line tool
dj-process --config demos/process_on_ray/configs/demo.yaml
```

Data-Juicer will process the demo dataset with the demo config file and export the result datasets to the directory specified by the `export_path` argument in the config file.

### Running Example of Distributed Deduplication

In the `dedup.yaml` config file, we set the executor type to "ray" and specify an automatic Ray address.
And we use a dedicated distributed version of MinHash Deduplicator to deduplicate the dataset.

```yaml
project_name: 'demo-dedup'
dataset_path: './demos/process_on_ray/data/'
export_path: './outputs/demo-dedup/demo-ray-bts-dedup-processed'

executor_type: 'ray'  # Set the executor type to "ray"
ray_address: 'auto'  # Set an automatic Ray address

# process schedule
# a list of several process operators with their arguments
process:
  - ray_bts_minhash_deduplicator:  # a distributed version of minhash deduplicator
      tokenization: 'character'
```

Run the demo to deduplicate the dataset:

```shell
# Run the tool from source
python tools/process_data.py --config demos/process_on_ray/configs/dedup.yaml

# Use the command-line tool
dj-process --config demos/process_on_ray/configs/dedup.yaml
```

Data-Juicer will dedup the demo dataset with the demo config file and export the result datasets to the directory specified by the `export_path` argument in the config file.
