# Load external tfrecord with TFDS


If you have a `tf.train.Example` proto (inside `.tfrecord`, `.riegeli`,...),
which has been generated by third party tools, that you would like to directly
load with tfds API, then this page is for you.

In order to load your `.tfrecord` files, you only need to:

*   Follow the TFDS naming convention.
*   Add metadata files (`dataset_info.json`, `features.json`) along your
    tfrecord files.

Limitations:

*   `tf.train.SequenceExample` is not supported, only `tf.train.Example`.
*   You need to be able to express the `tf.train.Example` in terms of
    `tfds.features` (see section bellow).

## File naming convention

In order for your `.tfrecord` files to be detected by TFDS, they need to follow
the following naming convention:
`<dataset_name>-<split_name>.<file-extension>-xxxxx-of-yyyyy`

For example, MNIST has the
[following files](https://console.cloud.google.com/storage/browser/tfds-data/datasets/mnist/3.0.1):

*   `mnist-test.tfrecord-00000-of-00001`
*   `mnist-train.tfrecord-00000-of-00001`


## Add metadata

### Provide the feature structure

For TFDS to be able to decode the `tf.train.Example` proto, you need to provide
the `tfds.features` structure matching your specs. For example:

```python
features = tfds.features.FeaturesDict({
    'image': tfds.features.Image(shape=(256, 256, 3)),
    'label': tfds.features.ClassLabel(names=['dog', 'cat'])
    'objects': tfds.features.Sequence({
        'camera/K': tfds.features.Tensor(shape=(3,), dtype=tf.float32),
    }),
})
```

Corresponds to the following `tf.train.Example` specs:

```python
{
    'image': tf.io.FixedLenFeature(shape=(), dtype=tf.string),
    'label': tf.io.FixedLenFeature(shape=(), dtype=tf.int64),
    'objects/camera/K': tf.io.FixedLenSequenceFeature(shape=(3,), dtype=tf.int64),
}
```

Specifying the features allow TFDS to automatically decode images, video,...
Like any other TFDS datasets, features metadata (e.g. label names,...) will be
exposed to the user (e.g. `info.features['label'].names`).

#### If you control the generation pipeline

If you generate datasets outside of TFDS but still control the generation
pipeline, you can use `tfds.features.FeatureConnector.serialize_example` to
encode your data from `dict[np.ndarray]` to `tf.train.Example` proto `bytes`:

```python
with tf.io.TFRecordWriter('path/to/file.tfrecord') as writer:
  for ex in all_exs:
    ex_bytes = features.serialize_example(data)
    f.write(ex_bytes)
```

This will ensure feature compatibility with TFDS.

Similarly, a `feature.deserialize_example` exists to decode the proto
([example](https://www.tensorflow.org/datasets/features#serializedeserialize_to_proto))

#### If you don't control the generation pipeline

If you want to see how `tfds.features` are represented in a `tf.train.Example`,
you can examine this in colab:

*   To translate `tfds.features` into the human readable structure of the
    `tf.train.Example`, you can call `features.get_serialized_info()`.
*   To get the exact `FixedLenFeature`,... spec passed to
    `tf.io.parse_single_example`, you can use the following code snippet:

    ```python
    example_specs = features.get_serialized_info()
    nested_feature_specs = tfds.core.example_parser._build_feature_specs(example_specs)
    feature_specs = tfds.core.utils.flatten_nest_dict(nested_feature_specs)
    ```

Note: If you're using custom feature connector, make sure to implement
`to_json_content`/`from_json_content` and test with `self.assertFeature` (see
[feature connector guide](https://www.tensorflow.org/datasets/features#create_your_own_tfdsfeaturesfeatureconnector))

### Get statistics on splits


TFDS requires to know the exact number of examples within each shard. This is
required for features like `len(ds)`, or the
[subplit API](https://www.tensorflow.org/datasets/splits):
`split='train[75%:]'`.

*   If you have this information, you can explicitly create a list of
    `tfds.core.SplitInfo` and skip to the next section:

    ```python
    split_infos = [
        tfds.core.SplitInfo(
            name='train',
            shard_lengths=[1024, ...],  # Num of examples in shard0, shard1,...
            num_bytes=0,  # Total size of your dataset (if unknown, set to 0)
        ),
        tfds.core.SplitInfo(name='test', ...),
    ]
    ```

*   If you do not know this information, you can compute it using the
    `compute_split_info.py` script (or in your own script with
    `tfds.folder_dataset.compute_split_info`). It will launch a beam pipeline
    which will read all shards on the given directory and compute the info.


### Add metadata files

To automatically add the proper metadata files along your dataset, use
`tfds.core.write_metadata`:

```python
tfds.folder_dataset.write_metadata(
    data_dir='/path/to/my/dataset/1.0.0/',
    features=features,
    # Pass the `out_dir` argument of compute_split_info (see section above)
    # You can also explicitly pass a list of `tfds.core.SplitInfo`
    split_infos='/path/to/my/dataset/1.0.0/',

    # Optionally, additional DatasetInfo metadata can be provided
    # See:
    # https://www.tensorflow.org/datasets/api_docs/python/tfds/core/DatasetInfo
    description="""Multi-line description."""
    homepage='http://my-project.org',
    supervised_keys=('image', 'label'),
    citation="""BibTex citation.""",
)
```

Once the function has been called once on your dataset directory, metadata files
( `dataset_info.json`,...) have been added and your datasets are ready to be
loaded with TFDS (see next section).

## Load dataset with TFDS

### Directly from folder

Once the metadata have been generated, datasets can be loaded using
`tfds.builder_from_directory` which returns a `tfds.core.DatasetBuilder` with
the standard TFDS API (like `tfds.builder`):

```python
builder = tfds.builder_from_directory('~/path/to/my_dataset/3.0.0/')

# Metadata are avalailable as usual
builder.info.splits['train'].num_examples

# Construct the tf.data.Dataset pipeline
ds = builder.as_dataset(split='train[75%:]')
for ex in ds:
  ...
```

### Folder structure (optional)

For better compatibility with TFDS, you can organize your data as
`<data_dir>/<dataset_name>[/<dataset_config>]/<dataset_version>`. For example:

```
data_dir/
    dataset0/
        1.0.0/
        1.0.1/
    dataset1/
        config0/
            2.0.0/
        config1/
            2.0.0/
```

This will make your datasets compatible with the `tfds.load` / `tfds.builder`
API, simply by providing `data_dir/`:

```python
ds0 = tfds.load('dataset0', data_dir='data_dir/')
ds1 = tfds.load('dataset1/config0', data_dir='data_dir/')
```
