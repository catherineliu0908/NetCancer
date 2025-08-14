# NetCancer

**NetCancer** is a computational pipeline designed to predict early-stage lymph node metastasis in OSCC using network-based methods. It retrieves TCGA bulk RNA-seq data, retireves features from DE analysis and literture review, constructs sample-specific interaction networks (SINs) using the SWEET framework, and applies an graph neural network (GNN)-based representation learning approach, Pretraining on PanCancer and Fine-tuning on OSCC.

This model integrates **contrastive learning** to capture structural similarities across patient-specific networks and **supervised learning** for accurate metastasis prediction. By jointly optimizing both objectives, the framework learns biologically relevant patterns from high-dimensional omics data represented as graphs.

---

## Features

- RNA-seq data preprocessing for early-stage OSCC samples  
- Sample-specific network construction via SWEET  
- Joint contrastive and supervised GNN training  
- End-to-end prediction of lymph node metastasis  (change to pretrain and finetuning)

---

## Dependencies

---

## Usage
### Step 1: Prepare and proprocess bulk RNA-seq Data
a. Download TCGA RNA-seq data and filter for early-stage colorectal cancer samples.

```bash
Rscript data/download_TCGA.R
```

```Outputs : 
'data/others/gene_map_TCGA.csv'
(in each cancer type: "data/GDCdata/'cancer type'/")
'clinical_table.csv' 
'filtered_raw.csv'
'filtered_tpm_log2.csv'
```

b. get DE genes

```bash
Rscript data/DESeq2.R
```

```Outputs: 
'DESeq2/{cancer type}_DEs.csv'
'DESeq2/Distinct_DEGs_All_Cancers.csv'
```


### Step 2: Get LNM-related features (including all DE genes)

a. download EMT related genes from dbEMT 2.0 https://bioinfo-minzhao.org/dbemt/download.cgi
(data/others/dbemt2.txt)

b. Lymphangiogenesis-related genes from https://www.informatics.jax.org/vocab/gene_ontology/GO:0001946
(data/others/GO_0001946.txt)

c. HALLMARK_ANGIOGENESIS genes from https://www.gsea-msigdb.org/gsea/msigdb/human/geneset/HALLMARK_ANGIOGENESIS
(data/others/HALLMARK_ANGIOGENESIS.v2024.1.Hs.gmt)

d. from literature:
"Alitalo, K., & Detmar, M. (2012). Mechanisms of lymphatic metastasis. Nature Reviews Cancer, 12(9), 639–652. PMC3938272"
[VEGFC, VEGFD, VEGFR3, PDGFBB, IGF1, IGF2...] save in (data/others/LNM_associated_genes.txt)

*others/ 
-  reactome source from:
    https://data.broadinstitute.org/gsea-msigdb/msigdb/release/2024.1.Hs/ for enrichment and mapping

- ENSG & gene symbol mapping: 'data/others/gene_map_TCGA.csv' from TCGA

### Step 3: Construct Sample-specific Interaction Networks (SINs)
Use the SWEET framework to calculate patient-specific edge scores and contruct network with networkX.

```bash
Rscript data/SWEET/0.formatting.R # features = all DE genes + STEP 2: a.~d.
python3 data/SWEET/1.SWEET_sample_weight_calculating.py
python3 data/SWEET/1.SWEET_sample_weight_calculating.py \
-f data/SWEET/target_net/expression.txt \
-s data/SWEET/target_net/weight.txt

python3 data/SWEET/2.SWEET_edge_score_calculating.py
python3 data/SWEET/2.SWEET_edge_score_calculating.py \
-f data/SWEET/target_net/expression.txt \
-w data/SWEET/target_net/weight.txt \
-p data/SWEET/target_net/patient.txt \
-g data/SWEET/target_net/gene.txt \
-s data/SWEET/target_net

python3 data/SWEET/3-4.SWEET_mean_std_zscore_filter.py 
python3 data/SWEET/3-4.SWEET_mean_std_zscore_filter.py \
-p data/SWEET/target_net/patient.txt \
-l data/SWEET/target_net \
-s data/SWEET/target_net/mean_std.txt 

python3 data/SWEET/5.construct_network.py
python3 data/SWEET/5.construct_network.py --project target
```
Output: 
'dataset/PanCancer/raw/PanCancer_networks.pkl'
'dataset/PanCancer/raw/PanCancer_clinical_table.csv'
 
'dataset/target/raw/target_networks.pkl'
'dataset/target/raw/target_clinical_table.csv'


### Step 4: compute CBDI scores and update network (only keep high CBDI genes local structure + concat DE genes)
keep the high CBDI value ( >1.0) local structure, filter out all the other node and edges that dont connect in the network

```bash
python3 master/CBDI.py
```

### Step 4: Train Graph Neural Network
```bash
python3 master/main_pretrain.py --hyperparameter_tuning # this set true for later hyperparameter tuning

python3 master/main_finetune.py --hyperparameter_tuning # tuning the best model
