---
layout: tutorial_hands_on
enable: false

title: CRISPR screen analysis
zenodo_link: https://zenodo.org/record/5658600
questions:
- What are the steps to process CRISPR screen data?
- How to identify differentially enriched guides across multiple experimental conditions?
objectives:
- Check quality of raw reads
- Trim sequencing adapters
- Count guide sequences in samples
- Evaluate the quality of count results
- Test for differential enrichment of guides across conditions
time_estimation: 2H
key_points:
- CRISPR screen data can be analysed using MAGeCK and standard read quality tools
contributors:
- mblue9
- kenjifujihara
- twishigulati
requirements:
  -
    type: "internal"
    topic_name: sequence-analysis
    tutorials:
      - quality-control
  -
    type: "internal"
    topic_name: galaxy-interface
    tutorials:
      - collections
      - upload-rules

---


# Introduction
{:.no_toc}

The **C**lustered **R**egularly **I**nterspaced **S**hort **P**alindromic **R**epeats (CRISPR) Type II system is a bacterial immune system that has been modified for genome engineering. CRISPR consists of two components: a guide RNA (gRNA) and a non-specific CRISPR-associated endonuclease (Cas9). The gRNA is a short synthetic RNA composed of a scaffold sequence necessary for Cas9-binding (trRNA) and ~20 nucleotide spacer or targeting sequence which defines the genomic target to be modified (crRNA). The ease of generating gRNAs makes CRISPR one of the most scalable genome editing technologies and has been recently utilized for genome-wide screens.
Cas9 induces double-stranded breaks (DSB) within the target DNA. The resulting DSB is then repaired by either error-prone Non-Homologous End Joining (NHEJ) pathway or less efficient but high-fidelity Homology Directed Repair (HDR) pathway. The NHEJ pathway is the most active repair mechanism and it results in small nucleotide insertions or deletions (indels) at the DSB site. This results in in-frame amino acid deletions, insertions or frameshift mutations leading to premature stop codons within the open reading frame (ORF) of the targeted gene. Ideally, the end result is a loss-of-function mutation within the targeted gene; however, the strength of the knockout phenotype for a given mutant cell is ultimately determined by the amount of residual gene function. These days, pooled whole-genome knockout, inhibition and activation CRISPR libraries and CRISPR sub-library pools are commonly screened.


![Illustration of CRISPR Screen Method](../../images/crispr-screen/crispr_screen.jpg "CRISPR knockout and activation methods (from {% cite Joung2016 %})")


> ### Agenda
>
> In this tutorial, we will cover:
>
> 1. TOC
> {:toc}
>
{: .agenda}

# Preparing the reads

## Data upload

Here we will demonstrate analysing CRISPR screen using data from {% cite Fujihara2020 %}. We will use FASTQ files containing 1% of reads from the original samples to demonstrate the read processing steps.

> ### {% icon hands_on %} Hands-on: Retrieve CRISPR screen fastq datasets
>
> 1. Create a new history for this tutorial
> 2. Import the files from Zenodo:
>
>    - Open the file {% icon galaxy-upload %} __upload__ menu
>    - Click on __Rule-based__ tab
>    - *"Upload data as"*: `Collection(s)`
>    - Copy the following tabular data, paste it into the textbox and press <kbd>Build</kbd>
>
>      ```
>      T0-Control https://zenodo.org/api/files/646f76fa-dc6b-404a-9a74-c371a43aacd5/T0-Control.fastq.gz
>      T8-APR-246 https://zenodo.org/api/files/646f76fa-dc6b-404a-9a74-c371a43aacd5/T8-APR-246.fastq.gz
>      T8-Vehicle https://zenodo.org/api/files/646f76fa-dc6b-404a-9a74-c371a43aacd5/T8-Vehicle.fastq.gz
>      ```
>  
>    ![Rule-based Uploader](../../images/crispr-screen/crispr_rule_uploader.png)
>
>    - From **Rules** menu select `Add / Modify Column Definitions`
>       - Click `Add Definition` button and select `List Identifier(s)`: column `A`
>
>         > ### {% icon tip %} Can't find *List Identifier*?
>         > Then you've chosen to upload as a 'dataset' and not a 'collection'. Close the upload menu, and restart the process, making sure you check *Upload data as*: **Collection(s)**
>         {: .tip}
>
>       - Click `Add Definition` button and select `URL`: column `B`
>
>    - Click `Apply` 
>    - In the Name: box type `fastqs` and press <kbd>Upload</kbd>
>      
>    ![Rule-based Editor](../../images/crispr-screen/crispr_rule_editor.png)
>    
{: .hands_on}

## Raw reads QC

First we'll check the quality of the raw read sequences with [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) and aggregate the reports from the multiple samples with [MultiQC](https://multiqc.info/) ({% cite ewels2016multiqc %}). We will check if the base quality is good and for presence of adapters. For more details on quality control and what the FastQC plots mean see the ["Quality control" tutorial]({% link topics/sequence-analysis/tutorials/quality-control/tutorial.md %}).


> ### {% icon hands_on %} Hands-on: Quality control
>
> 1. {% tool [FastQC](toolshed.g2.bx.psu.edu/repos/devteam/fastqc/fastqc/0.72+galaxy1) %} with the following parameters:
>    - {% icon param-collection %} *"Short read data from your current history"*: fastqs (click "Dataset collection" button on left-side of this input field)
>
> 2. Inspect the webpage output of **FastQC** for the T0 sample
>
>    > ### {% icon question %} Questions
>    >
>    > What is the read length?
>    >
>    > > ### {% icon solution %} Solution
>    > >
>    > > The read length is 75 bp.
>    > >
>    > {: .solution}
>    >
>    {: .question}
>
> 3. {% tool [MultiQC](toolshed.g2.bx.psu.edu/repos/iuc/multiqc/multiqc/1.9+galaxy1) %} with the following parameters to aggregate the FastQC reports:
>     - In *"Results"*
>       - *"Which tool was used generate logs?"*: `FastQC`
>       - In *"FastQC output"*
>         - *"Type of FastQC output?"*: `Raw data`
>         - {% icon param-collection %} *"FastQC output"*: `Raw data` files (output of **FastQC**)
>
> 4. Inspect the webpage output from MultiQC for each FASTQ
>
>    > ### {% icon question %} Questions
>    >
>    > What do you think of the quality of the sequences?
>    >
>    > > ### {% icon solution %} Solution
>    > >
>    > > The quality seems good for the 3 files.
>    > >
>    > {: .solution}
>    >
>    {: .question}
>
{: .hands_on}


## Trim adapters

We need to trim the adapters to leave just the 20bp guide sequences.  We'll trim these sequences using [Cutadapt](https://cutadapt.readthedocs.io/en/stable/guide.html) ({% cite marcel2011cutadapt %}) and its linked adapter format `MY_5PRIME_ADAPTER...MY_3PRIME_ADAPTER`.


> ### {% icon details %} Adapter trimming
> 
> In this dataset the adapters are at different positions in the reads, as shown below.
>
> ![Adapter sequences dataset](../../images/crispr-screen/adapter_sequences_dataset.png "Reads from the T8-APR-246 sample with one of the guide sequences highlighted in blue (controlguide1 from Brunello library file). The adapter sequences are directly adjacent to the guide on the right and left.")
>
> MAGeCK count can trim adapters around the guide sequences. However, the adapters need to be at the same position in each read, requiring the same trimming length, as described on the MAGeCK website [here](https://sourceforge.net/p/mageck/wiki/advanced_tutorial/). An example for what MAGeCK expects is shown below. If you used MAGeCK count trimming with the dataset in this tutorial it wouldn't be able to trim the adapters properly and you would only get ~60% reads mapping instead of >80%.
>
> ![Adapters MAGeCK can trim](../../images/crispr-screen/adapter_sequences_mageck.png "Example showing what MAGeCK count expects to be able to auto-detect and trim adapters. Guide sequence is in blue, with the adapter sequences directly adjacent on the right and left.")
>
> So for this dataset, as the adapters are not at the same position in each read, we need to trim the adapters before running MAGeCK count. To trim, we could run Cutadapt twice, first trimming the 5' adapter sequence, then trimming the 3' adapter. Alternatively, we can run Cutadapt just once using the linked adapter format `MY_5PRIME_ADAPTER...MY_3PRIME_ADAPTER`, as discussed [here](https://github.com/marcelm/cutadapt/issues/261#issue-261019127).
{: .details}


> ### {% icon hands_on %} Hands-on: Trim adapters
>
> 1. {% tool [Cutadapt](toolshed.g2.bx.psu.edu/repos/lparsons/cutadapt/cutadapt/3.4+galaxy1) %} with the following parameters:
>    - *"Single-end or Paired-end reads?"*: `Single-end`
>        - {% icon param-collection %} *"FASTQ/A file #1"*: all fastq.gz files 
>        - In *"Read 1 Options"*:
>            - In *"3' (End) Adapters"*:
>                - {% icon param-repeat %} *"Insert 3' (End) Adapters"*
>                    - *"Source"*: `Enter custom sequence`
>                        - *"Enter custom 3' adapter sequence"*: `TGTGGAAAGGACGAAACACCG...GTTTTAGAGCTAGAAATAGCAAG`
>    - In *"Filter Options"*:
>        - *"Minimum length (R1)"*: `20`
>    - In *"Read Modification Options"*:
>        - *"Quality cutoff"*: `20`
>    - *"Outputs selector"*: 
>        - *"Report"*: tick
>
> 2. Inspect the report output from Cutadapt for the T8-APR sample
>
>    > ### {% icon question %} Questions
>    >
>    > What % of reads had adapters?
>    >
>    > > ### {% icon solution %} Solution
>    > >
>    > > 99.9% 
>    > >
>    > {: .solution}
>    >
>    {: .question}
>
{: .hands_on}


# Counting

For the rest of the CRISPR screen analysis, counting and testing, we'll use MAGeCK ({% cite Li2014 %}, {% cite Li2015 %}).

To count how many guides we have for each gene, we need a library file that tells us which guide sequence belongs to which gene. The guides used here are from the Brunello library ({% cite Doench2016 %}) which contains 77,441 sgRNAs, an average of 4 sgRNAs per gene, and 1000 non-targeting control sgRNAs. The library file must be tab-separated and contain no spaces within the target names. If necessary, there are tools in Galaxy that can format the file removing spaces and converting commas to tabs.

> ### {% icon hands_on %} Hands-on: Count guides per gene
> 1. Import the library file 
>    ```
>    https://zenodo.org/api/files/646f76fa-dc6b-404a-9a74-c371a43aacd5/brunello.tsv
>    ```
>
>    {% snippet faqs/galaxy/datasets_import_via_link.md %}
>
> 2. {% tool [MAGeCK count](toolshed.g2.bx.psu.edu/repos/iuc/mageck_count/mageck_count/0.5.9.2.4) %} with the following parameters:
>    - *"Reads Files or Count Table?"*: `Separate Reads files`
>        - {% icon param-collection %} *"Sample reads"*: the `Read 1 Output` (outputs of **Cutadapt**)
>    - {% icon param-file %} *"sgRNA library file"*: the `brunello.tsv` file
>    - In *"Output Options"*:
>        - *"Output Count Summary file"*: `Yes`
>        - *"Output plots"*: `Yes`
>
> 3. Inspect the Count Summary file
>
> 4. We have been using 1% of reads from the samples. Import the count summary file for the full dataset so you can see what the values look like.
>    ```
>    https://zenodo.org/api/files/646f76fa-dc6b-404a-9a74-c371a43aacd5/kenji_mageck_count_summary.tsv
>    ```
>
{: .hands_on}

The contents of the count summary file is explained on the MAGeCK website [here](https://sourceforge.net/p/mageck/wiki/output/#count_summary_txt), also shown below. The columns are as follows. **To help you evaluate the quality of the data, recommended values are shown in bold.**

Column | Content
--- | ---
File | The fastq (or the count table) file used.
Label | The label of that fastq file assigned.
Reads | Total number reads in the fastq file. **(Recommended: 100~300 times the number of sgRNAs)**
Mapped | Total number of reads that can be mapped to library
Percentage | Mapped percentage, calculated as Mapped/Reads **(Recommended: at least 60%)**
TotalsgRNAs | Total number of sgRNAs in the library
Zerocounts | Total number of missing sgRNAs (sgRNAs that have 0 counts) **(Recommended: no more than 1%)**
GiniIndex | The Gini Index of the read count distribution. A smaller value indicates more eveness of the count distribution. **(Recommended: around 0.1 for plasmid or initial state samples, and around 0.2-0.3 for negative selection samples )**


> ### {% icon question %} Questions
>
> Is the data quality good for the 3 samples? Use the count summary file for the full dataset, and the recommended values in the table above, to answer these questions.
>
> 1. Have we sequenced enough reads?
> 2. Is the mapped percentage good?
> 3. Is the sgRNA zero count value good?
> 4. Is the Gini Index good?
>
> > ### {% icon solution %} Solution
> >
> > 1. The number of reads is ok. The lowest number of reads we have for a sample is 17,855,968 (T8-Vehicle), we have 77,441 guides so we have ~230 reads per guide (17,855,968/77,441). A minimum of 100 reads per guide, preferably 300, is recommended.
> > 2. Yes, it is >80% in all 3 samples.
> > 3. T0-Control has 0.69% (535/77441 * 100) which is good. The T8 samples are just slightly high at 2.1% (1659/77441 * 100) and 2.7% (2102/77441 * 100).
> > 4. The Gini Index is 0.09 for T0-Control (initial state) which is good. The T8 samples are 0.12 and 0.13 which is good (not too high) as this is a negative selection experiment.
> >
> {: .solution}
>
{: .question}


The paper by {% cite Li2015 %} has more information on MAGeCK quality control.

# Testing

We have been using 1% of reads from the samples in the original dataset to save time as FASTQ files are large. As counts files are small, here we will import and use the MAGeCK counts file generated using all the reads for the samples.

## Two conditions

If we want to compare the drug treatment (T8-APR-246) to the vehicle control (T8-Vehicle) we can use MAGeCK test. MAGeCK test uses a robust ranking aggregation (RRA) algorithm ({% cite Li2014 %}).

We could specify them using their names, which must match the names used in the columns of the counts file, but hypens aren't allowed. We can also specify by their positions in the counts file with the first sample column being 0.

> ### {% icon hands_on %} Hands-on: Test for enrichment
> 1. Import the count file from the full dataset [Zenodo]({{ page.zenodo_link }}) or the Shared Data library (if available):
>    ```
>    https://zenodo.org/api/files/646f76fa-dc6b-404a-9a74-c371a43aacd5/kenji_mageck_counts.tsv
>    ```
>
> 2. {% tool [MAGeCKs test](toolshed.g2.bx.psu.edu/repos/iuc/mageck_test/mageck_test/0.5.9.2.1) %} with the following parameters:
>    - {% icon param-file %} *"Counts file"*: the `kenji_mageck_counts.tsv` file
>    - *"Specify Treated samples or Control"*: `Treated samples`
>        - *"Treated Sample Labels (or Indexes)"*: `0`
>    - *"Control Sample Labels (or Indexes)"*: `1`
>    - In *"Output Options"*:
>        - *"Output normalized counts file"*: `Yes`
>        - *"Output plots"*: `Yes`
>
>    > ### {% icon details %} Normalization
>    >
>    > We are using MAGeCK's default normalization method "median" which is more robust to outliers.
>    > Figure M1 from {% cite Li2014 %} shows a comparison of median ("median") versus total ("total") normalization for two CRISPR screen datasets. 
>    > The distribution of the read counts of significant sgRNAs (FDR=1%) was compared with the mean read count distribution of all sgRNAs (“all”, black). The distribution of the significant sgRNAs should be similar to the distribution of all sgRNAs if the normalization method is unbiased. The difference is small for the leukemia dataset. However, in the melanoma dataset, where a few sgRNAs have very large read counts, the difference is larger, as “total” normalization will prefer sgRNAs with higher read-counts. In contrast, the distribution after “median” normalization is closer to the distribution of all sgRNAs.
>    > 
>    > ![Median versus Total normalization](../../images/crispr-screen/median_vs_total.png)
>    >
>    {: .details}
>
> 3. Inspect the PDF Report output.
>
>    > ### {% icon question %} Questions
>    >
>    > What are the top 3 genes?
>    >
>    > > ### {% icon solution %} Solution
>    > >
>    > > ESD, MTHFD1L and SHMT2 which are part of the glutathione pathway. That was the main pathway found to be altered in the published paper for this dataset.
>    > >
>    > {: .solution}
>    >
>    {: .question}
>
{: .hands_on}

MAGeCK test outputs a gene summary file that contains the columns described below.

Column number | Column name | Content
--- | --- | ---
1 | id | Gene ID
2 | num | The number of targeting sgRNAs for each gene
3 | neg\|score | The RRA lo value of this gene in negative selection
4 | neg\|p-value | The raw p-value (using permutation) of this gene in negative selection
5 | neg\|fdr | The false discovery rate of this gene in negative selection
6 | neg\|rank  | The ranking of this gene in negative selection
7 | neg\|goodsgrna | The number of "good" sgRNAs, i.e., sgRNAs whose ranking is below the alpha cutoff (determined by the --gene-test-fdr-threshold option), in negative selection.
8 | neg\|lfc | The log2 fold change of this gene in negative selection. The way to calculate gene lfc is controlled by the --gene-lfc-method option
9 | pos\|score | The RRA lo value of this gene in positive selection
10 | pos\|p-value | The raw p-value (using permutation) of this gene in positive selection
11 | pos\|fdr | The false discovery rate of this gene in positive selection
12 | pos\|rank  | The ranking of this gene in positive selection
13 | pos\|goodsgrna | The number of "good" sgRNAs, i.e., sgRNAs whose ranking is below the alpha cutoff (determined by the --gene-test-fdr-threshold option), in positive selection.
14 | pos\|lfc | The log fold change of this gene in positive selection

Genes are ranked by the p.neg field (by default). If you need a ranking by the p.pos, you can use the **Sort** data in ascending or descending order tool in Galaxy.

We can create a volcano plot to visualise the output, plotting the magnitude of change for drug treatment versus vehicle control (lfc) versus significance (p-value). As we have two columns for lfc and p-value, one for negative selection and one for positive, we first combine these into one column for each using the **awk** tool. If the `neg|p-value` is smaller than the `pos|p-value` the gene is negatively selected. If the `neg|p-value` is larger than the `pos|p-value` the gene is positively selected. Then we create the plot using the **Volcano plot** tool.


> ### {% icon hands_on %} Hands-on: Create volcano plot
> 1. {% tool [Text reformatting](toolshed.g2.bx.psu.edu/repos/bgruening/text_processing/tp_awk_tool/1.1.2) %} with the following parameters:
>    - {% icon param-file %} *"File to process"*: `MAGeCK test Gene Summary`
>    - *"AWK Program"*: Copy and paste the text in the grey box below into this field
>
>    ```
>    # Print new header for first line
>    NR == 1 { print "gene", "pval", "fdr", "lfc" }
>
>    # Only process lines after first
>    NR > 1 {
>        # check if neg pval < pos pval
>        if ($4 < $10){
>           # if it is, print negative selection values
>            print $1, $4, $5, $8
>        } else {
>           # if it's not, print positive selection values
>            print $1, $10, $11, $14
>        }
>    }
>    ```
>  2. Inspect the file output. It should look like below.
> ```
> gene    pval        fdr       lfc
> ESD     1.7977e-05  0.279703  -1.9404
> MTHFD1L 2.7827e-05  0.279703  -0.90207
> SHMT2   8.9391e-05  0.454208  -1.1307
> FLI1    9.0376e-05  0.454208  -0.085192
> ```
>
> 3. {% tool [Volcano Plot](toolshed.g2.bx.psu.edu/repos/iuc/volcanoplot/volcanoplot/0.0.5) %} to create a volcano plot
>    - {% icon param-file %} *"Specify an input file"*: the Text reformatting output file
>    - {% icon param-file %} *"File has header?"*: `Yes`
>    - {% icon param-select %} *"FDR (adjusted P value)"*: `Column 3`
>    - {% icon param-select %} *"P value (raw)"*: `Column 2`
>    - {% icon param-select %} *"Log Fold Change"*: `Column 4`
>    - {% icon param-select %} *"Labels"*: `Column 1`
>    - {% icon param-select %} *"Points to label"*: `Significant`
>        - {% icon param-text %} *"Only label top most significant"*: `10`
>
> 4. Inspect the plot in the PDF output.
>
>    > ### {% icon question %} Questions
>    >
>    > What is the most significant gene?
>    >
>    > > ### {% icon solution %} Solution
>    > >
>    > > ATP5E as it is the gene nearest the top of the plot.
>    > >
>    > > ![Volcano plot](../../images/crispr-screen/volcanoplot.png)
>    > >
>    > {: .solution}
>    >
>    {: .question}
>
> For more details on using the volcano plot tool, see the [tutorial here]({% link topics/transcriptomics/tutorials/rna-seq-viz-with-volcanoplot/tutorial.md %}). For how to customise the volcano plot tool output using R, see the [tutorial here]({% link topics/transcriptomics/tutorials/rna-seq-viz-with-volcanoplot-r/tutorial.md %}).
>
{: .hands_on}



> ### {% icon tip %} Tip: Getting help
>
> For questions about using Galaxy, you can ask in the [Galaxy help forum](https://help.galaxyproject.org/). For questions about MAGeCK, you can ask in the [MAGeCK Google group](https://groups.google.com/g/mageck).
>
{: .tip}


# Conclusion
{:.no_toc}

CRISPR Screen reads can be assessed for quality using standard sequencing tools such as FASTQC, MultiQC and trimmed of adapters using Cutadapt. The detection of enriched guides can be performed using MAGeCK.
