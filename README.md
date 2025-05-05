# ClusSearch

ClusSearch is a pipeline that implements a workflow described in the paper "Stable population structure in Europe during the Iron Age, despite high mobility" by Antonio et al. (2024).  
For given regions, it groups genetically similar individuals into clusters, which are categorized into majority and potential outlier clusters. These clusters are then split into periodic subclusters.  
The resulting subclusters are compared inside and across regions to confirm outlier status and to find potential sources for outlier clusters.

The central tool of this workflow is **Admixtools2 qpadm**, which is applied as one-component models. Given a target `T`, a left component `L`, and a group of reference populations `R`, such a model tests against the null hypothesis that `T` and `L` form a clade in respect to `R`. A resulting p-value above the significance threshold therefore means that `L` and `T` do form a clade. This concept is used for initial clustering of each regions individuals, as well as for confirming outlier status of potential outlier clusters, source detection for confirmed outliers and model competition of multiple potential sources. See detailed description of the pipeline steps below for more detaiis.

---

## Installation

### 1. Download and Extract the Pipeline
1. Go to the [Releases page](https://github.com/YourGitHubUsername/clusSearch/releases) and download the latest ZIP file.
2. Extract the ZIP file into a directory of your choice.

### 2. Make Scripts Executable
To ensure all scripts are executable, run the following commands in the extracted directory:
```bash
chmod +x exec.sh
chmod +x .scripts/*.sh
```
Note: Only `.sh` files in the `.scripts/` directory require execution permissions. Other non-shell files do not need modification.

### 3. Install Dependencies
ClusSearch includes a script to install required dependencies automatically. Run the following command:
```bash
./install_dependencies.sh
```
This script will install necessary Python packages, R libraries, and external tools (e.g., Admixtools2) required for the pipeline.

---

## Execution

To run ClusSearch, execute the `exec.sh` script inside the ClusSearch directory and provide a parameter file:
```bash
~/path/to/clusSearch/exec.sh parameter_file
```

### Important Notes:
- ClusSearch will create a tab-separated dataframe for each region and dynamically add columns with new information as it proceeds through the workflow.
- An example parameter file is provided in the ClusSearch directory, together with all necessary supplementary files for a quick-start example run.
- **CAUTION**: Do not move `exec.sh` individually to another location. It calls scripts from `clusSearch/.scripts/`, and therefore needs to be located in the `clusSearch/` directory when executed.

---

## Parameters

The parameter file is a plain text file specifying mandatory and optional parameters, which can be provided in any order. A parameters name **must be seperated from its value with a colon ':'**. 
It is formatted as follows:
```
eigenstrat_fileset: path/to/fileset_prefix
ids_directory: path/to/directory/storing/regional_ID_lists
rightlist: path/to/rightlist_textfile
...
```

### Mandatory Parameters:
- **`eigenstrat_fileset`**: Prefix of the eigenstrat fileset (.geno, .snp, .ind files) containing all samples to be clustered and reference set individuals.  
  **CAUTION**: Do not have the Plink format (.bed, .bim, .fam files) of the same fileset in the same directory. Otherwise, Admixtools2's `extract_f2` may attempt to use the `.fam` file, which does not have updated population labels, and fail.

- **`ids_directory`**: A directory holding text files, each specifying the individuals of a region, one ID per line. The text file names must end with `_IDs`. The string before this suffix will be considered the region's name (e.g., `Sardinia_IDs`).

- **`rightlist`**: A text file specifying the populations to be used as the "rightlist". One population ID per line.

- **`f2_directory`**: A directory where precomputed f2 stats are saved. If precomputation is turned off, this parameter is not necessary.

- **`python_path`**: Path to the Python installation, necessary for a clustering package.

- **`output_folder`**: A directory where the results are saved.

### Optional Parameters:
- **`dates_file`**:
A tab separated text file that has individual IDs in first column, and date in years BP in second column. The dates are necessary to split clusters into periodic subclusters.
If not provided, date specific annotations will be set as NA.

Example:
I16326  2625
I2446   4215
I13778  3234

- **`periods_file`**:
A tab separated text file where custom periods can be set upon which the clusters are split into periodic subclusters. First column is periods name, second column is older period boundary and third column
is younger period boundary; in years BP. Period boundaries are inclusive, so avoid overlap! If not provided, defaults to clusSearch/.scripts/standard_periods, which divides 10000 to 0 BP into 500 year bins.

Example:
Iron_Age        2750    1950
Roman_Imperial  1949    1550

- **`custom_rightgroups`**:
A tab separated text file to set custom population labels. First column shows individual ID, second column a custom population label. Example:

Bichon.SG       WHG
Canes.SG        WHG
Chan.SG WHG

- **`outlier_threshold_relative`**:
Clusters that contain less than the given fraction of the regions samples are considered potential outliers. Takes values between 0 and 1, defaults to 0.05.

- **`outlier_threshold_absolute`**:
Clusters that have no more than the given number of samples are considered potential outliers. Defaults to 2.

- **`pval_threshold_cluster`**:
The p-value threshold used for initial clustering of individuals. If the one-component model of two individuals of a region yields a p-value above that threshold, they are considered cladal and clustered
together. Takes values between 0 and 1, defaults to 0.05.

- **`pval_threshold_validate`**:
The p-value threshold used for confirming outlier status. If the one-compontent model of a potential outlier cluster and a in-region majority cluster yields a p-value above that threshold, they are considered
cladal and the former is removed from the outlier list. Has to be set between 0 and 1, defaults to 0.01.

- **`pval_threshold_ancestry`**:
The p-value threshold used for searching ancestries to outlier clusters. If the one-component model of a outlier cluster and a majority cluster from a different region yields a p-value above that threshold,
they are considered cladal and the majority is considered a potential source to the outlier. The same threshold is used for model_competition, see description of STEP 15 below for a detailed explanation of
model-competition. Has to be set between 0 and 1, defaults to 0.01.

- **`from_step`**:
Defines what script the pipeline starts at. Takes values between 1 and 17, defaults to 1. See below for detailed description of steps.

- **`to_step`**:
Defines to what step the pipeline will run. Takes values between 1 and 17, defaults to 1. See below for detailed description of steps.


- **`precompute_f2`**:
If set to F, f2 stats will not be precomputed and qpadm will use the eigenstrat fileset as input data, which results in a longer runtime. Also, running qpadm directly on a fileset yields much higher
pvalues than running it on precomputed f2 stats with default parameters, resulting in fewer clusters. Reproduction tests of the results described in the original paper (Antonio et al. 24) showed that using
precomputed f2 stats seems to be much more akin to the approach used by the original authors of this workflow.
Defaults to T, meaning f2 stats are precomputed and read by Admixtools2's extract_f2 and f2_from_precomp functions. The parameters max_mem, max_miss, auto_Only, remove_na and afprod are only relevant
when f2 stats are precomputed.

- **`max_mem`**:
Argument of extract_f2. Specifies maximum amount of memory to be used for precomputation of f2 stats. Defaults to 2000 (2GB).

- **`max_miss`**:
Argument of extract_f2. SNPs that are missing in a fraction of populations higher than max_miss are discarded during f2 precomputation. Has to be set between 0 and 1.
Defaults to 1, meaning all SNPs are kept.

- **`auto_Only`**:
Argument of extract_f2. Set to F to keep SNPs on autosomes as well as gonosomes during precomputation of f2 stats. Defaults to T, meaning only SNPs on chromosomes 1 - 22 are kept.

- **`remove_na`**:
Argument of f2_from_precomp. Set to F to keep blocks containing missing values reading the precomputed f2 stats. Defaults to T, meaning blocks with missing values are removed.

- **`afprod`**:
Argument of f2_from_precomp. Set to T to return negative allele frequency products instead of f2 estimates. Results in more precise f4-stats if data had large amounts of missingness. Defaults to F.

---

## Steps

The pipeline consists of 17 sequential steps implemented as Bash scripts, with additional R scripts called for specific functions.  
The steps are detailed below, including the script names and their operations. Each script is located in the `clusSearch/.scripts/` directory.

### Step 1: Initial Individual File Manipulation
**Script**: `1_initial_indfile_manipulation.sh`  
This step adjusts the `.ind` file population IDs to allow individual-based models. If custom right groups are provided, labels are updated accordingly.

...

(*Note: I’ll avoid repeating the full step descriptions here unless otherwise requested, as they are unchanged from your original README.*)

---

## Advanced Editing

In this pipeline, not all possible arguments for the called functions are implemented as parameters. To change arguments not covered by the parameter file (e.g., arguments for computing p-values directly on a fileset), you need to edit the source code itself.

**Best Practice**:  
Before editing, create a copy of the original script:
```bash
cp clusSearch/.scripts/scriptName.sh clusSearch/.scripts/scriptName_old.sh
```
To restore the original script:
```bash
mv clusSearch/.scripts/scriptName_old.sh clusSearch/.scripts/scriptName.sh
```

---

## Example Directory Structure
After installation, your directory structure should look like this:
```
clusSearch/
├── install_depencies.sh
├── exec.sh
├── .scripts/
│   ├── 1_initial_indfile_manipulation.sh
│   ├── 2_f2_extraction_individuals.sh
│   ├── ...
├── example_param
├── example_data/
│   ├── ids_directory/
│   └── ...
└── README.md
```

---
