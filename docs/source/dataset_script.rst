Create a dataset loading script
===============================

Write a dataset script to load and share your own datasets. It is a Python file that defines the different configurations and splits of your dataset, as well as how to download and process the data.

Any dataset script, for example ``my_dataset.py``, can be placed in a folder or a repository named ``my_dataset`` and be loaded with:

.. code-block::

    >>> from datasets import load_dataset
    >>> load_dataset("path/to/my_dataset")

The following guide includes instructions for dataset scripts for how to:

* Add dataset metadata.
* Download data files.
* Generate samples.
* Test if your dataset was generated correctly.
* Create a Dataset card.
* Upload a dataset to the Hugging Face Hub or GitHub.

Open the `SQuAD dataset loading script <https://github.com/huggingface/datasets/blob/master/datasets/squad/squad.py>`_ template to follow along on how to share a dataset.

.. tip::

   To help you get started, try beginning with the dataset loading script `template <https://github.com/huggingface/datasets/blob/master/templates/new_dataset_script.py>`_!


Add dataset attributes
----------------------

The first step is to add some information, or attributes, about your dataset in :func:`datasets.DatasetBuilder._info`. The most important attributes you should specify are:

1. :obj:`datasets.DatasetInfo.description` provides a concise description of your dataset. The description informs the user what's in the dataset, how it was collected, and how it can be used for a NLP task.

2. :obj:`datasets.DatasetInfo.features` defines the name and type of each column in your dataset. This will also provide the structure for each example, so it is possible to create nested subfields in a column if you want. Take a look at :class:`datasets.Features` for a full list of feature types you can use.

.. code-block::

   datasets.Features(
       {
           "id": datasets.Value("string"),
           "title": datasets.Value("string"),
           "context": datasets.Value("string"),
           "question": datasets.Value("string"),
           "answers": datasets.Sequence(
               {
                   "text": datasets.Value("string"),
                   "answer_start": datasets.Value("int32"),
               }
           ),
       }
   )

3. :obj:`datasets.DatasetInfo.homepage` contains the URL to the dataset homepage so users can find more details about the dataset.

4. :obj:`datasets.DatasetInfo.citation` contains a BibTeX citation for the dataset.

After you've filled out all these fields in the template, it should look like the following example from the SQuAD loading script:

.. code-block::

   def _info(self):
       return datasets.DatasetInfo(
           description=_DESCRIPTION,
           features=datasets.Features(
               {
                   "id": datasets.Value("string"),
                   "title": datasets.Value("string"),
                   "context": datasets.Value("string"),
                   "question": datasets.Value("string"),
                   "answers": datasets.features.Sequence(
                       {"text": datasets.Value("string"), "answer_start": datasets.Value("int32"),}
                   ),
               }
           ),
           # No default supervised_keys (as we have to pass both question
           # and context as input).
           supervised_keys=None,
           homepage="https://rajpurkar.github.io/SQuAD-explorer/",
           citation=_CITATION,
       )

Multiple configurations
^^^^^^^^^^^^^^^^^^^^^^^

In some cases, your dataset may have multiple configurations. For example, the `SuperGLUE <https://huggingface.co/datasets/super_glue>`_ dataset is a collection of 5 datasets designed to evaluate language understanding tasks. 🤗 Datasets provides :class:`datasets.BuilderConfig` which allows you to create different configurations for the user to select from.

Let's study the `SuperGLUE loading script <https://github.com/huggingface/datasets/blob/master/datasets/super_glue/super_glue.py>`_ to see how you can define several configurations.

1. Create a :class:`datasets.BuilderConfig` subclass with attributes about your dataset. These attributes can be the features of your dataset, label classes, and a URL to the data files.

.. code-block::

   class SuperGlueConfig(datasets.BuilderConfig):
       """BuilderConfig for SuperGLUE."""

   def __init__(self, features, data_url, citation, url, label_classes=("False", "True"), **kwargs):
       """BuilderConfig for SuperGLUE.

       Args:
       features: `list[string]`, list of the features that will appear in the
           feature dict. Should not include "label".
       data_url: `string`, url to download the zip file from.
       citation: `string`, citation for the data set.
       url: `string`, url for information about the data set.
       label_classes: `list[string]`, the list of classes for the label if the
           label is present as a string. Non-string labels will be cast to either
           'False' or 'True'.
       **kwargs: keyword arguments forwarded to super.
       """
       # Version history:
       # 1.0.2: Fixed non-nondeterminism in ReCoRD.
       # 1.0.1: Change from the pre-release trial version of SuperGLUE (v1.9) to
       #        the full release (v2.0).
       # 1.0.0: S3 (new shuffling, sharding and slicing mechanism).
       # 0.0.2: Initial version.
       super(SuperGlueConfig, self).__init__(version=datasets.Version("1.0.2"), **kwargs)
       self.features = features
       self.label_classes = label_classes
       self.data_url = data_url
       self.citation = citation
       self.url = url

2. Create instances of your config to specify the values of the attributes of each configuration. This gives you the flexibility to specify all the name and description of each configuration. These sub-class instances should be listed under :obj:`datasets.DatasetBuilder.BUILDER_CONFIGS`:

.. code-block::

   class SuperGlue(datasets.GeneratorBasedBuilder):
       """The SuperGLUE benchmark."""

       BUILDER_CONFIGS = [
           SuperGlueConfig(
               name="boolq",
               description=_BOOLQ_DESCRIPTION,
               features=["question", "passage"],
               data_url="https://dl.fbaipublicfiles.com/glue/superglue/data/v2/BoolQ.zip",
               citation=_BOOLQ_CITATION,
               url="https://github.com/google-research-datasets/boolean-questions",
           ),
           ...
           ...
           SuperGlueConfig(
               name="axg",
               description=_AXG_DESCRIPTION,
               features=["premise", "hypothesis"],
               label_classes=["entailment", "not_entailment"],
               data_url="https://dl.fbaipublicfiles.com/glue/superglue/data/v2/AX-g.zip",
               citation=_AXG_CITATION,
               url="https://github.com/rudinger/winogender-schemas",
           ),
       

3. Now, users can load a specific configuration of the dataset with the configuration ``name``:

.. code-block::

   >>> from datasets import load_dataset
   >>> dataset = load_dataset('super_glue', 'boolq')

Default configurations
^^^^^^^^^^^^^^^^^^^^^^

Users must specify a configuration name when they load a dataset with multiple configurations. Otherwise, 🤗 Datasets will raise a ``ValueError``, and prompt the user to select a configuration name. You can avoid this by setting a default dataset configuration with the :attr:`datasets.DatasetBuilder.DEFAULT_CONFIG_NAME` attribute:

.. code-block::

   class NewDataset(datasets.GeneratorBasedBuilder):

   VERSION = datasets.Version("1.1.0")

   BUILDER_CONFIGS = [
       datasets.BuilderConfig(name="first_domain", version=VERSION, description="This part of my dataset covers a first domain"),
       datasets.BuilderConfig(name="second_domain", version=VERSION, description="This part of my dataset covers a second domain"),
   ]

   DEFAULT_CONFIG_NAME = "first_domain"

.. important::

   Only use a default configuration when it makes sense. Don't set one because it may be more convenient for the user to not specify a configuration when they load your dataset. For example, multi-lingual datasets often have a separate configuration for each language. An appropriate default may be an aggregated configuration that loads all the languages of the dataset if the user doesn't request a particular one.

Download data files and organize splits
---------------------------------------

After you've defined the attributes of your dataset, the next step is to download the data files and organize them according to their splits. 

1. Create a dictionary of URLs in the loading script that point to the original SQuAD data files:

.. code-block::

   _URL = "https://rajpurkar.github.io/SQuAD-explorer/dataset/"
       _URLS = {
           "train": _URL + "train-v1.1.json",
           "dev": _URL + "dev-v1.1.json",
       }

.. tip::
    If the data files live in the same folder or repository of the dataset script, you can just pass the relative paths to the files instead of URLs.

2. :func:`datasets.DownloadManager.download_and_extract` takes this dictionary and downloads the data files. Once the files are downloaded, use :class:`datasets.SplitGenerator` to organize each split in the dataset. This is a simple class that contains:

  * The :obj:`name` of each split. You should use the standard split names: :obj:`datasets.Split.TRAIN`, :obj:`datasets.Split.TEST`, and :obj:`datasets.Split.VALIDATION`.

  * :obj:`gen_kwargs` provides the file paths to the data files to load for each split.

Your :obj:`datasets.DatasetBuilder._split_generator()` should look like this now:

.. code-block::

   def _split_generators(self, dl_manager: datasets.DownloadManager) -> List[datasets.SplitGenerator]:
       urls_to_download = self._URLS
       downloaded_files = dl_manager.download_and_extract(urls_to_download)

       return [
           datasets.SplitGenerator(name=datasets.Split.TRAIN, gen_kwargs={"filepath": downloaded_files["train"]}),
           datasets.SplitGenerator(name=datasets.Split.VALIDATION, gen_kwargs={"filepath": downloaded_files["dev"]}),
       ]

Generate samples
----------------

At this point, you have:

* Added the dataset attributes.
* Provided instructions for how to download the data files.
* Organized the splits.

The next step is to actually generate the samples in each split. 

1. :obj:`datasets.DatasetBuilder._generate_examples` takes the file path provided by :obj:`gen_kwargs` to read and parse the data files. You need to write a function that loads the data files and extracts the columns.

2. Your function should yield a tuple of an ``id_``, and an example from the dataset.

.. code-block::

   def _generate_examples(self, filepath):
       """This function returns the examples in the raw (text) form."""
       logger.info("generating examples from = %s", filepath)
       with open(filepath) as f:
           squad = json.load(f)
           for article in squad["data"]:
               title = article.get("title", "").strip()
               for paragraph in article["paragraphs"]:
                   context = paragraph["context"].strip()
                   for qa in paragraph["qas"]:
                       question = qa["question"].strip()
                       id_ = qa["id"]

                       answer_starts = [answer["answer_start"] for answer in qa["answers"]]
                       answers = [answer["text"].strip() for answer in qa["answers"]]

                       # Features currently used are "context", "question", and "answers".
                       # Others are extracted here for the ease of future expansions.
                       yield id_, {
                           "title": title,
                           "context": context,
                           "question": question,
                           "id": id_,
                           "answers": {"answer_start": answer_starts, "text": answers,},
                       }

Testing data and checksum metadata
----------------------------------

We strongly recommend adding testing data and checksum metadata to your dataset to verify and test its behavior. This ensures the generated dataset matches your expectations.
Testing data and checksum metadata are mandatory for Canonical datasets stored in the GitHub repository of the 🤗 Datasets library.

.. important::

   Make sure you run all of the following commands **from the root** of your local ``datasets`` repository.

Dataset metadata
^^^^^^^^^^^^^^^^

1. Run the following command to create the metadata file, ``dataset_infos.json``. This will also test your new dataset loading script and make sure it works correctly.

.. code::

   datasets-cli test datasets/<your-dataset-folder> --save_infos --all_configs

2. If your dataset loading script passed the test, you should now have a ``dataset_infos.json`` file in your dataset folder. This file contains information about the dataset, like its ``features`` and ``download_size``.

Dummy data
^^^^^^^^^^

Next, you need to create some dummy data for automated testing. There are two methods for generating dummy data: automatically and manually. 

Automatic
"""""""""

If your data file is one of the following formats, then you can automatically generate the dummy data:

* txt
* csv
* tsv
* jsonl
* json
* xml

Run the command below to generate the dummy data:

.. code::

   datasets-cli dummy_data datasets/<your-dataset-folder> --auto_generate

Manual
""""""

If your data files are not among the supported formats, you will need to generate your dummy data manually. Run the command below to output detailed instructions on how to create the dummy data:

.. code-block::

   datasets-cli dummy_data datasets/<your-dataset-folder>

   ==============================DUMMY DATA INSTRUCTIONS==============================
   - In order to create the dummy data for my-dataset, please go into the folder './datasets/my-dataset/dummy/1.1.0' with `cd ./datasets/my-dataset/dummy/1.1.0` .

   - Please create the following dummy data files 'dummy_data/TREC_10.label, dummy_data/train_5500.label' from the folder './datasets/my-dataset/dummy/1.1.0'

   - For each of the splits 'train, test', make sure that one or more of the dummy data files provide at least one example

   - If the method `_generate_examples(...)` includes multiple `open()` statements, you might have to create other files in addition to 'dummy_data/TREC_10.label, dummy_data/train_5500.label'. In this case please refer to the `_generate_examples(...)` method

   - After all dummy data files are created, they should be zipped recursively to 'dummy_data.zip' with the command `zip -r dummy_data.zip dummy_data/`

   - You can now delete the folder 'dummy_data' with the command `rm -r dummy_data`

   - To get the folder 'dummy_data' back for further changes to the dummy data, simply unzip dummy_data.zip with the command `unzip dummy_data.zip`

   - Make sure you have created the file 'dummy_data.zip' in './datasets/my-dataset/dummy/1.1.0'
   ===================================================================================

.. tip::

   Manually creating dummy data can be tricky. Make sure you follow the instructions from the command ``datasets-cli dummy_data datasets/<your-dataset-folder>``. If you are still unable to succesfully generate dummy data, open a `Pull Request <https://github.com/huggingface/datasets/pulls>`_ and we will be happy to help you out!

There should be two new files in your dataset folder:

* ``dataset_infos.json`` stores the dataset metadata including the data file checksums, and the number of examples required to confirm the dataset was generated properly.

* ``dummy_data.zip`` is a file used to test the behavior of the loading script without having to download the full dataset.


Test
^^^^

The last step is to actually test dataset generation with the real and dummy data. Run the following command to test the real data:

.. code::

   RUN_SLOW=1 pytest tests/test_dataset_common.py::LocalDatasetTest::test_load_real_dataset_<your_dataset_name>

Test the dummy data:

.. code::

   RUN_SLOW=1 pytest tests/test_dataset_common.py::LocalDatasetTest::test_load_dataset_all_configs_<your_dataset_name>

If both tests pass, your dataset was generated correctly!
