# End-to-End Pipeline for the BioCreative VI Precision Medicine Relation Extraction Task

This code constitutes an end-to-end system for the purpose of extracting interacting protein-protein pairs affected by a mutation. It includes three primary components in the pipeline: a supervised named entity recognition (NER) model, a gene normalization (GN), and a supervised relation classification (RC) model. 

The challenge/task description can be found here:
http://www.biocreative.org/tasks/biocreative-vi/track-4/

# Requirements

~~~
numpy
argparse
pandas
fileinput
sklearn
tensorflow 1.0.0 with tensorflow-fold
~~

# Training

This will train on the pre-formatted data provided in the `corpus_train` directory. Note that this data is compiled from multiple sources including an *augmented* version of the original training data; please see the paper below for more details on the training data composition.

## Named Entity Recognition (NER)

From the `ner_model` directory, run:

`python train.py --datapath=../corpus_train`

This will create a folder `saved_model_autumn` in `corpus_train` where the trained models will be saved (each part of an ensemble). A word vocabulary `word_vocab.ner.txt` will also be created in the same folder. These files will be loaded during the annotation process.

## Relation Classification (RC)

From the `rc_model` directory, run:

`python train.py --datapath=../corpus_train`

This will similarly create a `saved_model_ppi` folder and word vocabulary file.

# Annotating the Test Set

All intermediate outputs of the pipeline will be saved to the `pipeline_test` folder. Let's assume we want to annotate articles from the file `Final_Gold_Relation.json` corresponding to the test set. Keep in mind that this contains the articles and their groundtruth annotations. First, we will run the following command from inside the `pipeline_test` directory:

`python generate-pipeline-feed.py Final_Gold_Relation.json`

This will create a new file `pipeline_feed.txt` serving as input to our pipeline. The input should only include the PMID and the article title/abstract text of each article, one article per line. No groundtruth annotations are fed into the pipeline.

Once a feed file is generated, from the root directory, we can run a series of commands to take the feed and process it through each component in the pipeline. For convenience, we provide a bash script named `run_pipeline_test.sh` to handle this aspect of the system: 

~~~
PIPELINE=pipeline_test
INPUT=$PIPELINE/pipeline_feed.txt
PTOKEN=$PIPELINE/pipeline1.tokenized.txt
PNER=$PIPELINE/pipeline2.ner.txt
PNER2=$PIPELINE/pipeline2.5.ner.txt
PGNORM=$PIPELINE/pipeline3.gnorm.txt
OUTPUT=$PIPELINE/pipeline_output.txt
DATAPATH=corpus_train
GNCACHE=gene_normalization

CUDA_VISIBLE_DEVICES=""
python -u $PIPELINE/tokenize_input.py < $INPUT > $PTOKEN
python -u ner_model/annotate.py --datapath=$DATAPATH < $PTOKEN > $PNER
python -u ner_correction/annotate.py --datapath=$DATAPATH < $PNER > $PNER2
python -u gn_model/annotate.py --datapath=$DATAPATH --cachepath=$GNCACHE < $PNER2 > $PGNORM
python -u rc_model/extract.py --datapath=$DATAPATH < $PGNORM > $OUTPUT
echo "Output saved to $OUTPUT"
~~~

It is sufficient to run the following command from the root directory:

`bash run_pipeline_test.sh`

This will produce an output file at `pipeline_feed/pipeline_output.txt`. The last step is to take these predictions and put it in a format that can be processed by the official evaluation script. A simple way to conform to the required format is to take the JSON test file and replace all groundtruth relation annotations with our own predictions and save the result as a new JSON file. Here we use a script utility named `stitch-results.py` to accomplish this:

`python stitch-results.py Final_Gold_Relation.json`

This will produce a nicely formatted JSON file with our predicted annotations: `PMtask_results.json`.

# Evaluating Predictions

The official evaluation script is found here:
https://github.com/ncbi-nlp/BC6PM

To run the evaluation script for this dataset:

`python eval_json.py relation Final_Gold_Relation.json PMtask_results.json`