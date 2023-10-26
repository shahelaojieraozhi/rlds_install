# RLDS

RLDS stands for Reinforcement Learning Datasets and it is an ecosystem of tools
to store, retrieve and manipulate episodic data in the context of Sequential
Decision Making including Reinforcement Learning (RL), Learning for
Demonstrations, Offline RL or Imitation Learning.

This repository includes a library for manipulating RLDS compliant datasets. For
other parts of the pipeline please refer to:

*   [EnvLogger](http://github.com/deepmind/envlogger) to create synthetic
    datasets
*   [RLDS Creator](http://github.com/google-research/rlds-creator) to create
    datasets where a human interacts with an environment.
*   [TFDS](http://www.tensorflow.org/datasets/catalog/overview) for existing RL
    datasets.

Learn more about the RLDS ecosystem in the
[Google AI Blog](https://ai.googleblog.com/2021/12/rlds-ecosystem-to-generate-share-and.html)
and the [arXiv paper](https://arxiv.org/abs/2111.02767).

## QuickStart & Colabs

See how to use RLDS in this
[tutorial](https://colab.research.google.com/github/google-research/rlds/blob/main/rlds/examples/rlds_tutorial.ipynb).

You can find more examples, including performance best practices in the
[examples page](docs/examples.md). Besides, the
[transformations page](docs/transformations.md) provides an
overview of the RLDS library.

## Available datasets

This is a non-exhaustive list of datasets that are compatible with RLDS:

*   **[D4RL](https://www.tensorflow.org/datasets/catalog/overview#d4rl)**:
    subset of the [D4RL suite](https://github.com/rail-berkeley/d4rl) with
    Mujoco, Adroit and AntMaze tasks.
*   **[RL Unplugged](https://www.tensorflow.org/datasets/catalog/overview#rl_unplugged)**:
    subset of the
    [RL Unplugged suite](https://github.com/deepmind/deepmind-research/tree/master/rl_unplugged)
    that includes DMLab, Atari, Real World RL, Locomotion and Control Suite
    datasets.
*   **[Robosuite](https://www.tensorflow.org/datasets/catalog/robosuite_panda_pick_place_can)**:
    three [Robosuite](https://robosuite.ai/) datasets generated with the RLDS
    tools.
*   **[Robomimic](https://www.tensorflow.org/datasets/catalog/overview#robomimic)**:
    [subset of the Robomimic suite](https://arise-initiative.github.io/robomimic-web/).
*   **[MuJoCo Locomotion](https://www.tensorflow.org/datasets/catalog/locomotion)**
    datasets created with a SAC agent trained on the environment reward of
    MuJoCo locomotion tasks. These datsets were generated with the RLDS tools.
*   **Robotics**:
    *   [MT Opt dataset](https://www.tensorflow.org/datasets/catalog/mt_opt)

If you want to add your dataset to this list, let us know!

## Dataset Format

The dataset is retrieved as a `tf.data.Dataset` of Episodes where each episode
contains a `tf.data.Dataset` of steps.

![drawing](docs/images/rlds.png "RLDS Diagram")

*   **Episode**: dictionary that contains a `tf.data.Dataset` of Steps, and
    metadata.

    The metadeta fields are user-defined. While no names or types are
    prescribed, we propose a set of optional fields that are generic and useful.
    * Metadata optional fields:

      * `episode_id`: Unique identifier of the episode within the dataset.
        The episode ID should also be unique with high probability across
        datasets so different datasets can be merged easily on the fly.
      * `agent_id`: Unique identifier of the agent(s) that generated the
        episode. In a multi-agent setting, this could be for example a tensor
        of size Nx2 where N is the number of agents and where each pair
        represents the agent name in the environment and the ID of the agent
        that actually generated the episode.
      * `environment_config`: Configuration of the environment that was used
        to generate the episode.
      * `experiment_id`: Identifier of an experiment when the episode was
        generated as part of an experiment.
      * `invalid`: Flag to signal invalid episodes, which in general
        should be discarded at read time. Since episodes are in general
        recorded step by step, there are a few scenarios where an episode
        might be incomplete: e.g. machine preemption. This flag is usually
        used in dtaasets that have just been created and not polished for
        sharing.

*   **Step**: dictionary that contains:

    * Mandatory fields:

      *   `is_first`: if this is the first step of an episode that contains the
          initial state.
      *   `is_last`: if this is the last step of an episode, that contains the
          last observation. When true, `action`, `reward` and `discount`, and
          other cutom fields subsequent to the observation are considered invalid.
    * Optional fields:
      *   `observation`: current observation
      *   `action`: action taken in the current observation
      *   `reward`: return after appyling the action to the current observation
      *   `is_terminal`: if this is a terminal step
      *   `discount`: discount factor at this step.
      *   extra metadata

    When `is_terminal = True`, the `observation` corresponds to a final state,
    so `reward`, `discount` and `action` are meaningless. Depending on the
    environment, the final `observation` may also be meaningless.

    If an episode ends in a step where `is_terminal = False`, it means that this
    episode has been truncated. In this case, depending on the environment, the
    action, reward and discount might be empty as well.

    Note: Although some fields of the steps are optional, all the steps in the
    same dataset are required to have the same fields.

## How to create a dataset

Although you can read datasets with the RLDS format even if they were not
created with our tools (for example, by adding them to [TFDS](#load-with-tfds)),
we recommend the use of [EnvLogger] and [RLDS Creator] as they ensure that the
data is stored in a lossless fashion and compatible with RLDS.

### Synthetic datasets

Envlogger provides a [dm_env] `Environment` class wrapper that records
interactions between a real environment and an agent.

```
env = envlogger.EnvLogger(
      environment,
      data_directory=`/tmp/mydataset`)
```

Besides, two callbacks can be passed to the `EnvLogger` constructor to
store per-step metadata and per-episode metadata. See the [EnvLogger]
documentation for more details.

Note that per-session metadata can be stored but is currently ignored when
loading the dataset.

NOTE: We recommend to use the TFDS Envlogger backend in order to get datasets
that can be read directly with TFDS. See an example in
[this colab](https://colab.research.google.com/github/google-research/rlds/blob/main/rlds/examples/rlds_tfds_envlogger.ipynb).

Note that the Envlogger follows the [dm_env] convention. So considering:

*   `o_i`: observation at step `i`
*   `a_i`: action applied to `o_i`
*   `r_i`: reward obtained when applying `a_i` in `o_i`
*   `d_i`: discount for reward `r_i`
*   `m_i`: metadata for step `i`

Data is generated as:

```none
    (o_0, _, _, _, m_0) → (o_1, a_0, r_0, d_0, m_1)  → (o_2, a_1, r_1, d_1, m_2) ⇢ ...
```

But loaded with RLDS as:

```none
    (o_0,a_0, r_0, d_0, m_0) → (o_1, a_1, r_1, d_1, m_1)  → (o_2, a_2, r_2, d_2, m_2) ⇢ ...
```

### Human datasets

If you want to collect data generated by a human interacting with an
environment, check the [RLDS Creator].

## How to load a dataset

RL datasets can be loaded with [TFDS](#load-with-tfds)
and they are retrieved
with the canonical [RLDS dataset format](#dataset-format).

### Load with TFDS

Note: In TFDS you can load the nested dataset as a batched sequence instead of a
`tf.data.Dataset`. See the [FAQ](#faq) for details.

#### Datasets created with Envlogger and the TFDS backend

These datasets can be loaded directly with:

```py
tfds.builder_from_directory('path').as_dataset(split='all')
```

or from a list of paths:

```py
tfds.builder_from_directories(paths).as_dataset(split='all')
```

See more examples in
[this colab](https://colab.research.google.com/github/google-research/rlds/blob/main/rlds/examples/rlds_tfds_envlogger.ipynb).

#### Datasets in the TFDS catalog

These datasets can be loaded directly with:

```py
tfds.load('dataset_name').as_dataset()['train']
```

This is how we load the datasets in the
[tutorial](https://colab.research.google.com/github/google-research/rlds/blob/main/rlds/examples/rlds_tutorial.ipynb).

See the full documentation and the catalog in the [TFDS] site.

#### Datasets in your own repository

Datasets can be implemented with TFDS both inside and outside of the TFDS
repository. See examples
[here](https://www.tensorflow.org/datasets/external_tfrecord?hl=en#load_dataset_with_tfds).

## How to add your dataset to TFDS

This is only necessary when your dataset is not already in TFDS format or if you
want to add it to the TFDS catalog. See more details in
[this page](docs/tfds-add.md).

## Performance best practices

As RLDS exposes RL datasets in a form of Tensorflow's
[tf.data](https://www.tensorflow.org/api_docs/python/tf/data), many Tensorflow's
[performance hints](https://www.tensorflow.org/guide/data_performance#optimize_performance)
apply to RLDS as well. It is important to note, however, that RLDS datasets are
very specific and not all general speed-up methods work out of the box. Advice
on improving performance might not result in expected outcome.

RLDS provides an optimized
[library of transformations](docs/transformations.md), but
to get a better understanding on how to use RLDS datasets effectively we
recommend going through this
[colab](https://colab.research.google.com/github/google-research/rlds/blob/main/rlds/examples/rlds_performance.ipynb).

## FAQ

### Processing steps in random order

While by default the order of episodes in RLDS datasets is randomized and there
is no need to randomize them again when loading the dataset, some algorithms
operate on steps/n-step transitions. There are different ways to interleave
steps across multiple episodes - for example:

* Shuffle steps using [tf.data.Dataset.shuffle](https://www.tensorflow.org/api_docs/python/tf/data/Dataset#shuffle).
Note that obtaining perfect shuffling this way involves specifying `buffer_size`
which can accomodate entire dataset and can result in high memory usage for big datasets.

* Interleave `N` copies of the dataset using [tf.data.Dataset.interleave](https://www.tensorflow.org/api_docs/python/tf/data/Dataset#interleave):

```
def ds_loader():
  episode_dataset = tfds.load(...)
  step_dataset = episode_dataset.flat_map(lambda x: x[rlds.STEPS])
  return step_dataset

dataset = Dataset.range(1, N).interleave(ds_loader, cycle_length=..., block_length=...)
```

Each copy of the dataset shuffles input partitions independently, so consecutive steps
returned by the resulting dataset come from unrelated episodes. It is important to note,
however, that this way each step will be loaded `N` times. To avoid duplicates,
it is possible to construct each dataset using disjoint [splits](https://www.tensorflow.org/datasets/splits).

See one example of randomized access in the
[Atari colab](https://colab.research.google.com/github/google-research/rlds/blob/main/rlds/examples/tfds_rlu_atari.ipynb).

### Processing random episodes in multiple readers.

Sometimes, users read multiple copies of the dataset in separate processes. For
example, to emulate a multiple-actor single learner scenario, where the actors
get the offline data from the same dataset. In these situations, it is important
that the different processes don't get the same sequence of episodes.

When the number of readers is known, the easiest way is to use the
[`split` API](https://www.tensorflow.org/datasets/splits#slicing_api) from TFDS
to ensure that each of the reader takes a different set of episodes from the
dataset. Note that if one of the reader dies, its portion of the dataset will
not be processed.

Another option is to ensure that the datasets are read in a non-deterministic
way. This can be achieved by setting `shuffle_files=True` and by tuning the
`ReadConfig` options in `tfds.load` or in `builder.as_dataset`. You can find
more details in the [TFDS documentation about determinism]. In this case, if a
reader dies, the full dataset can still be processed. However, with this option,
some episodes may appear more than once.

### Reducing memory usage

To improve throughput of loading datasets, by default TFDS loads multiple partitions
of the dataset in parallel. In the case of datasets with big episodes that can result
in high memory usage. If you run into high memory usage problems, it is worth playing
around with `read_config` provided to [tfds.load](https://www.tensorflow.org/datasets/api_docs/python/tfds/load).

### Loading the steps as a batch instead of a nested dataset.

If using TFDS you can load the nested dataset as a batched sequence instead of a
nested `tf.data.Dataset`. You can do it by using `SkipDecoding`:

```py
ds = tfds.load('d4rl_mujoco_halfcheetah/v0-medium', decoders={rlds.STEPS: tfds.decode.SkipDecoding()}, split='train')
```

To decode the steps as a dataset, you can use
`tf.data.Dataset.from_tensor_slices`.

```py

for e in ds:
 print(tf.data.Dataset.from_tensor_slices(e[rlds.STEPS]))
 break

```

When using `tfds.builder_from_directories` or `tfds.builder_from_directory`, the
`decoder` argument can be passed to `as_dataset`.

## Who uses RLDS

### Publications
Below is a sample of publications using RLDS:

*   [Hyperparameter Selection for Imitation Learning](https://arxiv.org/abs/2105.12034).
    L. Hussenot et al., ICML 2021.
*   [Continuous Control with Action Quantization from Demonstrations](https://arxiv.org/pdf/2110.10149.pdf),R.
    R. Dadashi et al., Deep RL Workshop @ NeurIPS 2021.
*   [What Matters for Adversarial Imitation Learning?](https://arxiv.org/pdf/2106.00672.pdf)
    M. Orsini et al., NeurIPS 2021.
*   [MT-Opt: Continuous Multi-Task Robotic Reinforcement Learning at Scale](https://arxiv.org/abs/2104.08212)
    D. Kalashnikov et al.
*   [Offline Reinforcement Learning with Pseudometric Learning](https://arxiv.org/abs/2103.01948)
    R. Dadashi et al., ICML 2021.
*   [Offline Reinforcement Learning as Anti-Exploration](https://arxiv.org/abs/2106.06431)
    S. Rezaheifar et al.

## Citation

If you use RLDS, please cite the [RLDS paper](https://arxiv.org/abs/2111.02767)
as

```
@misc{ramos2021rlds,
      title={RLDS: an Ecosystem to Generate, Share and Use Datasets in Reinforcement Learning},
      author={Sabela Ramos and Sertan Girgin and Léonard Hussenot and Damien Vincent and Hanna Yakubovich and Daniel Toyama and Anita Gergely and Piotr Stanczyk and Raphael Marinier and Jeremiah Harmsen and Olivier Pietquin and Nikola Momchev},
      year={2021},
      eprint={2111.02767},
      archivePrefix={arXiv},
      primaryClass={cs.LG}
}
```

## Acknowledgements

We greatly appreciate all the support from the
[TF-Agents](https://github.com/tensorflow/agents) team in setting up building
and testing for EnvLogger.

## Disclaimer

This is not an officially supported Google product.

[EnvLogger]: http://github.com/deepmind/envlogger
[RLDS Creator]: http://github.com/google-research/rlds-creator
[dm_env]: http://github.com/deepmind/dm_env/blob/master/docs/index.md
