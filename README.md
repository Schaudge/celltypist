# Overview of Celltypist
Celltypist is an automated cell type annotation tool for scRNA-seq datasets on the basis of logistic regression classifiers optimized by the stochastic gradient descent algorithm. Celltypist provides several different models for predictions, with a current focus on immune sub-populations, in order to assist in the accurate classification of different cell types and subtypes.

# Install the development version
```console
pip install celltypist-dev
```

# Usage

## 1. Use in the Python environment

### 1.1. Import the module
```python
import celltypist
from celltypist import models
```

### 1.2. Download all available models
The models serve as the basis for cell type predictions. Each model is on average 3 megabytes (MB). We thus encourage the users to download all of them.
```python
#Download all the available models from the remote Sanger server.
models.download_models()
#Update all models by re-downloading the latest versions if you think they may be outdated.
models.download_models(force_update=True)
#Show the local directory storing these models.
models.models_path
```

### 1.3. Overview of the models
All models are serialized in a binary format by [pickle](https://docs.python.org/3/library/pickle.html).
```python
#Get an overview of what these models represent and their names.
models.models_description()
```

### 1.4. Inspect the model of interest
To take a look at a given model, load the model as an instance of the `Model` class as defined in the Celltypist.
```python
#Select the model from the above list. If not provided, will default to `Immune_All_Low.pkl`.
model = models.load(model = 'Immune_All_Low.pkl')
#Examine cell types contained in the model.
model.cell_types
#Examine genes/features contained in the model.
model.features
#The stochastic gradient descent logistic regression classifier within the model.
model.classifier
#The standard scaler within the model (used to scale the input data).
model.scaler
```

### 1.5. Celltyping based on the input of count table 
Celltypist accepts the input data as a count table (cell-by-gene or gene-by-cell) in the format of `.txt`, `.csv`, `.tsv`, `.tab`, `.mtx` or `.mtx.gz`. A raw count matrix (reads or UMIs) is required.
```python
#Get a demo test data. This is a UMI count csv file with cells as rows and genes as columns.
input_file = celltypist.samples.get_sample_csv()
```
Assign the cell type labels within the model to the input test cells using the `annotate` function.
```python
#Predict the cell identity of each input cell.
predictions = celltypist.annotate(input_file, model = 'Immune_All_Low.pkl')
```
If your input file is in a gene-by-cell format (genes as rows and cells as columns), pass in the `transpose_input = True` argument. In addition, if the input is provided in the `.mtx` format, you will also need to specify the `gene_file` and `cell_file` arguments as the files containing names of genes and cells, respectively.
```python
#In case your input file is a gene-by-cell table.
predictions = celltypist.annotate(input_file, model = 'Immune_All_Low.pkl', transpose_input = True)
#In case your input file is a gene-by-cell mtx file.
predictions = celltypist.annotate(input_file, model = 'Immune_All_Low.pkl', transpose_input = True, gene_file = '/path/to/gene/file.txt', cell_file = '/path/to/cell/file.txt')
```
Similarly, if the `model` argument is not provided, Celltypist will by default use the `Immune_All_Low.pkl` model.
The `annotate` function will return an instance of the `AnnotationResult` class as defined in the Celltypist.
```python
#Examine the predicted cell types.
predictions.predicted_labels
#Examine the matrix representing the decision score of each cell belonging to a given cell type.
predictions.decision_matrix
#Examine the matrix representing the probability each cell belongs to a given cell type (transformed from decision matrix by the sigmoid function).
predictions.probability_matrix
```
The resulting `AnnotationResult` can be also transformed to an `AnnData` which stores the expression matrix in the log1p normalized format (to 10,000 counts per cell) by `to_adata`. The predicted cell type labels can be inserted to this `AnnData` as well by specifying `insert_labels = True` (which is the default).
```python
#Get an `AnnData` with predicted labels embedded into the observation metadata column.
adata = predictions.to_adata(insert_labels = True)
#Inspect this column (`predicted_labels`).
adata.obs.predicted_labels
```
In addition, you can insert the decision matrix into the 'AnnData' by passing in `insert_decision = True`, shwoing the decision scores of each cell type distributed across the input cells. Alternativley, setting `insert_probability = True` will insert the probability matrix into the `AnnData`. The former is the recommended way as not all test datasets converge to a meaningful range of probability distributions.
```python
#Get an `AnnData` with predicted labels and decision matrix (recommended).
adata = predictions.to_adata(insert_labels = True, insert_decision = True)
#Get an `AnnData` with predicted labels and probability matrix.
adata = predictions.to_adata(insert_labels = True, insert_probability = True)
```
You can then manipulate this object with any functions/modules applicable to `AnnData`. Actually, Celltypist provides a quick function `to_plots` to visualize your `AnnotationResult` and store the figures without the need to explicitly transform it into an `AnnData`.
```python
#Visualize the predicted cell types overlaid onto the UMAP.
predictions.to_plots(folder = '/path/to/a/folder', prefix = '')
```
UMAP coordinates will be generated for this dataset using a canonical Scanpy(https://scanpy.readthedocs.io/en/stable/) piepline. If you also would like to inspect the decision score and probability distributions for each cell type involved in the model, pass in the `plot_probability = True`.
```python
#Visualize the decision scores and probabilities of each cell type overlaid onto the UMAP as well.
predictions.to_plots(folder = '/path/to/a/folder', prefix = '', plot_probability = True)
```
N.B. Non-expressed genes (if you are sure of their expression absence in your data) are suggested to be included in the input table, as they point to the negative transcriptomic signatures when compared with the model used.

### 1.6. Celltyping based on Scanpy h5ad data
Celltypist also accepts the input data as an [AnnData](https://anndata.readthedocs.io/en/latest/) generated from for example [Scanpy](https://scanpy.readthedocs.io/en/stable/).
Since the expression of each gene will be centered and scaled by matching with the mean and standard deviation of that gene in the provided model, Celltypist requires a logarithmized and normalized expression matrix stored in the `AnnData` (log1p normalized expression to 10000 counts per cell). Celltypist will try the `adata.X` first, and if it does not suffice, try the `adata.raw.X`. If none of them fit into the desired data type or the expression matrix is not properly normalized, an error will be raised.
```python
#Provide the input as a Scanpy object.
predictions = celltypist.annotate('/path/to/input/adata', model = 'Immune_All_Low.pkl')
#Examine the predicted cell types.
predictions.predicted_labels
#Examine the matrix representing the probability each cell belongs to a given cell type.
predictions.probability_table
#Export the above two results to an Excel table.
predictions.write_excel('/path/to/output.xlsx')
```

### 1.7. Use a majority voting classifier combined with celltyping 
By default, Celltypist will only do the prediction job to infer the identities of input cells, which renders the prediction of each cell independent. To combine the cell type predictions with the cell-cell transcriptomic relationships, Celltypist offers a majority voting approach based on the idea that similar cell subtypes are more likely to form a (sub)cluster regardless of their individual prediction outcomes.
To turn on the majority voting classifier in addition to the Celltypist predictions, pass in `majority_voting = True` to the `annotate` function.
```python
#Turn on the majority voting classifier as well.
predictions = celltypist.annotate(input_file, model = 'Immune_All_Low.pkl', majority_voting = True)
```
During the majority voting, to define cell-cell relations, Celltypist will use a heuristic over-clustering approach according to the size of the input data with the aid of a canonical Leiden clustering pipeline. Users can also provide their own over-clustering result to the `over_clustering` argument. This argument can be specified in several ways:
   1) an input plain file with the over-clustering result of one cell per line.
   2) a string key specifying an existing metadata column in the `AnnData` (pre-created by the user).
   3) a Python list, Numpy 1D array, or Pandas series indicating the over-clustering result of all cells.
   4) if none of the above is provided, will use a heuristic over-clustering approach, noted above.
```python
#Add your own over-clustering result.
predictions = celltypist.annotate(input_file, model = 'Immune_All_Low.pkl', majority_voting = True, over_clustering = '/path/to/over_clustering/file')
```
Again, an instance of the `AnnotationResult` class will be returned.
```python
#Inspect the result.
predictions.predicted_labels
predictions.probability_table
#Export the result.
predictions.write_excel('/path/to/output.xlsx')
```

## 2. Use as the command line

### 2.1. Check the command line options
```bash
celltypist --help
```

### 2.2. Download all available models
```bash
celltypist --update-models
```
This will download the latest models from the remote server.

### 2.3. Overview of the models
```bash
celltypist --show-models
```

### 2.4. Celltyping based on the input of count table
See `1.5.` for the format of the desired count matrix.
```bash
celltypist --indata /path/to/input/file --model Immune_All_Low.pkl --outdir /path/to/outdir
```
You can add a different model to be used in the `--model` option. If the `--model` is not provided, Celltypist will by default use the `Immune_All_Low.pkl` model. The output directory will be set to the current working directory if `--outdir` is not specified.
If your input file is in a gene-by-cell format (genes as rows and cells as columns), add the `--transpose-input` option.
```bash
celltypist --indata /path/to/input/file --model Immune_All_Low.pkl --outdir /path/to/outdir --transpose-input
```
If the input is provided in the `.mtx` format, you will also need to specify the `--gene-file` and `--cell-file` options as the files containing names of genes and cells, respectively.
Other options that control the output files of Celltypist include `--prefix` which adds a custom prefix and `--xlsx` which merges the output files into one xlsx table. Check `celltypist --help` for more details.

### 2.5. Celltyping based on Scanpy h5ad data
See `1.6.` for the requirement of the Scanpy expression data.
```bash
celltypist --indata /path/to/input/adata --model Immune_All_Low.pkl --outdir /path/to/outdir
```

### 2.6. Use a majority voting classifier combined with celltyping
See `1.7.` for how the majority voting classifier works.
```bash
celltypist --indata /path/to/input/file --model Immune_All_Low.pkl --outdir /path/to/outdir --majority-voting
```
During the majority voting, to define cell-cell relations, Celltypist will use a heuristic over-clustering approach according to the size of the input data with the aid of a canonical clustering pipeline. Users can also provide their own over-clustering result to the `--over-clustering` argument. This argument can be specified in several ways:
   1) an input plain file with the over-clustering result of one cell per line.
   2) a string key specifying an existing metadata column in the `AnnData` (pre-created by the user).
   3) if none of the above is provided, will use a heuristic over-clustering approach, noted above.
```bash
celltypist --indata /path/to/input/file --model Immune_All_Low.pkl --outdir /path/to/outdir --majority-voting --over-clustering /path/to/over_clustering/file
```
