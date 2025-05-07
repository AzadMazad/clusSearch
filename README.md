# ClusSearch

ClusSearch is a pipeline that implements a workflow described in the paper "Stable population structure in Europe since the Iron Age, despite high mobility" by Antonio *et al*. (2024). For given regions, it groups genetically similar individuals into clusters, identifies outliers within regions and traces their potential sources across regions.

The central tool of this workflow is **Admixtools2 qpadm**, which is applied as one-component models. Given a target `T`, a left component `L`, and a group of reference populations `R`, such a model tests against the null hypothesis that `T` and `L` form a clade in respect to `R`. A resulting p-value above the significance threshold therefore means that `L` and `T` do form a clade. This concept is used for initial clustering of each regions individuals, as well as for confirming outlier status of potential outlier clusters, source detection for confirmed outliers and model competition of multiple potential sources. For a full breakdown of the workflow, see [Steps](#steps).

---

## Requirements & Installation

### Platform Requirements

ClusSearch is a **bash-based pipeline**, so you need to have a bash compatible environment to execute it. 
On **Linux Systems**, bash is the default shell of the terminal application.
**Mac Users** also have acces to a bash compatible terminal by default. You can see how to open it [here](https://support.apple.com/de-de/guide/terminal/apd5265185d-f365-44cb-8b09-71a064a42125/mac).
**Windows Users** can install the Windows Subsystem  for Linux (WSL). For info about WSL and its installation, see [microsofts WSL article](https://learn.microsoft.com/en-us/windows/wsl/install).

### Setup
#### 1. Download and Extract the Pipeline
1. Go to the [Releases page](https://github.com/YourGitHubUsername/clusSearch/releases) and download the latest ZIP file.
2. Extract the ZIP file into a directory of your choice.

The extracted directory have this structure:
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
**CAUTION**: Do not move `exec.sh` individually to another location. It calls scripts from `clusSearch/.scripts/`, and therefore needs to be located in the `clusSearch/` directory when executed.

### 2. Make Scripts Executable
In your bash envitonment, navigate into the extracted clusSearch directory and excute the following command to make sure all scripts are executable:
```bash
chmod +x *.sh
chmod +x .scripts/*.sh
```

### 3. Install Dependencies
For easy installation of dependencies, you will find an installer script in the clusSearch directory. If you are on a **Mac System**, be aware that [Homebrewer](https://brew.sh/) is necessary to succesfully run the installer. To execute the installation script, run the following command:
```bash
./install_dependencies.sh
```

The script will install the following dependencies:
- **`R`**
- **`Python 3`**
- **`R libraries`**: dplyr, tidyverse, reticulate, filelock, devtools, admixtools
- **`Python packages`**: scipy

To see which Versions where used during the project, see [here](#program-versions-used-during-project).

## Execution
If all dependencies are sucessfully installed you are ready to run ClusSearch. To run ClusSearch, execute the `exec.sh` script inside the ClusSearch directory and pass a parameter file as command line argument:
```bash
~/path/to/clusSearch/exec.sh parameter_file
```
An example parameter file and all necessary data for the corresponding example run are provided with the distribution. For a detailed description of the parameter file and all parameters, see [Parameters](#Parameters) section.

During its execution, clusSearch will create a tab_separated dataframe for each region passed, adding columns with new information as it proceeds through the workflow. For a detailed description of every pipeline step, see [Steps](#steps) section f.

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
- **`eigenstrat_fileset`**:
Prefix of the eigenstrat fileset (.geno, .snp, .ind files) containing all samples to be clustered and reference set individuals. **CAUTION**: Do not have the Plink format (.bed, .bim, .fam files) of the same fileset in the same directory. Otherwise, Admixtools2's `extract_f2` may attempt to use the `.fam` file, which does not have updated population labels, and fail.

- **`ids_directory`**:
A directory holding text files, each specifying the individuals of a region, one ID per line. The text file names **must end with** "_IDs" (e.g. `Sardinia_IDs`). The string before this suffix will be considered the region's name (e.g. "Sardinia"). **Hint**: Due to the lists being fetched by the suffix, you can simply change the suffix of a list to have it ignored by the pipeline (e.g. `Sardinia_ids`).

- **`rightlist`**:
A text file specifying the populations to be used as the "rightlist". One population ID per line.

- **`f2_directory`**:
A directory where precomputed f2 stats are saved. If precomputation is turned off, this parameter is not necessary.

- **`output_folder`**: A directory where the results are saved.

### Optional Parameters:
- **`dates_file`** (Optional): A tab-separated text file that has individual IDs in the first column and date in years BP in the second column. The dates are necessary to split clusters into periodic subclusters.

  Example:  
  ```
  I16326   2625
  I2446    4215
  I13778   3234
  ```

  If not provided, date-specific annotations will be set as **NA**. For the standard use-case, it is recommended to provide a dates file holding date information on every individual from the regional ID lists. Providing no dates file at all will result in all subclusters having the same period label ("NA"), effectively not splitting the initial clusters at all. This might be an option if you are interested in running the initial clusters through the whole pipeline without subclustering them.


- **`periods_file`**:
A tab separated text file where custom periods can be set upon which the clusters are split into periodic subclusters. First column is periods name, second column is older period boundary and third column
is younger period boundary; in years BP. Period boundaries are inclusive, so avoid overlap! If not provided, defaults to clusSearch/.scripts/standard_periods, which divides 10000 to 0 BP into 500 year bins.

Example:
```
Iron_Age        2750    1950
Roman_Imperial  1949    1550
```

- **`custom_rightgroups`**:
A tab separated text file to set custom population labels. First column shows individual ID, second column a custom population label. Example:

Example:
```
Bichon.SG       WHG
Canes.SG        WHG
Chan.SG         WHG
```

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

- **`python_path`**:
If no python3 path is provided, it is automatically assigned by the pipeline by running the bash command `which python3´. If you want to set a specific path, e.g. in case you have several python3 installations, you can specifiy the path to your preferred installation here. 

---

## Steps

ClusSearch consists of 17 Bash scripts that are executed sequentially, as well as additional R scripts which are called to run Admixtools2 functions or to allow effective data-frame manipulation.
The Bash scripts start with a number corresponding to execution order. The R scripts end with numbers corresponding to the steps by which they are called. All scripts as well as some default parameters are stored in clusSearch/.scripts. Below, every step is detailed, stating the exact name of the shell script and any subscripts called and describing the step's process. Parameters set by the user are specified by
the parameters name in single quotes, e.g. 'output_folder'.


#### STEP 1: Manipulating .ind file to have unique population IDs
**Script**: 1_initial_indfile_manipulation.sh


This script creates a new .ind file, where all individuals from the regional ID lists in the 'ids_directory' have their population ID overwritten by their individual ID. This is done because qpadm expect its `L` and `T` arguments to be population IDs, so unique population IDs are needed for all indivudals that are to be clustered to allow setting them as `L` and `T`.

Also, if a 'custom_rightgroups' file is provided, the individuals from the first column of that file that are present in the .ind file will have the custom label specified in the second column of that file set as their population ID, effectively creating custom populations. This is useful, because oftentimes the populations specified in the .ind file are not exactly those that one wants to use as right groups.
The .ind file that was passed to the pipeline is moved to .original_ind, with the modified .ind file takings its place so it gets recognized as part of the fileset downstream. If a .original_ind already exists for the used fileset, the move of .ind to .original_ind is omitted to avoid overwriting during repeated pipeline executions.


#### STEP 2: Calculation of f2 stats between each regions individuals
**Scripts**: 2_f2_extraction_indivduals.sh, f2_extraction_regional_individuals_2.R

For each region, this script computes f2 stats between all its individuals and then reads them into one cummulative .rds file. The individual f2 stats will be saved under
'f2_directory'/regional_pairwise/regionName, the .rds file under 'f2_directory'/f2_blocks/regional_pairwise/regionName.rds. This step is omitted when 'precompute_f2' is set to F.

#### STEP 3: Computing p-values for clustering
**Scripts**: 3_pairwise_pvalues_onF2data.sh, qpadm_oneComp_parallelized_onF2data_3_11_13.R

This script runs a one-component model of qpadm for every unique pair of individuals in each region in parallelized manner, using precomputed f2 stats as data input. The number of cores specified in
'n_cores' are used during computation. For every region, a tab separated file will be saved to 'output_folder'/metadata/regionName_pvalues_pairwise, holding L and T in the first two columns
and the resulting p-value of the model in the third.

**Scripts**: 3b_pairwise_pvalues_onFileset.sh, qpadm_oneComp_parallelized_onFileset_3b_11b_13b.R

If 'precomptue_f2' is set to F, the one-component qpadm models are run with the fileset itself as input data. Qpadm will be run with default arguments when this option is chosen. As above,
the results will be saved to a file for each region in 'output_folder'/metadata.

Warnings and Errors cast by the qpadm executions will collected saved to text files in 'output_folder'/metadata, to avoid cluttering of the console. 

#### STEP 4: Transformation of p-value tables into dissimilarity matrices
**Scripts**: 4_transform_p_to_d.sh, transform_pvalues_to_dvalues_4.R

The tab separated p-value table of the former step is transformed into a tab separated dissimilarity matrix. A p-value is transformed into a dissimilarity value by calculating $d = -\log_{10}(\text{p-value})$,
turning low p-values - which favor rejection of the null-hypothesis that L and T form a clade - into high dissimilarity scores. The resulting d-matrix is saved to 'output_folder'/metadata/regionName_dmatrix.

#### STEP 5: Clustering each regions individuals
**Script**: 5_cluster_upon_dMatrix.sh

The UPGMA algorithm implemented in pythons scipy.cluster.hierarchy (v 1.6.1) is used to perform hierarchical clustering on each regional d-matrix. The hierarchical clusters are then split along a
dissimilarity cut-off determined by calculating the dissimilarity value corresponding to the specified 'pval_threshold_clustering' , $c = -\log_{10}(\text{pval-cluster-threshold})$.
The resulting flat clusters of each region are saved to a tab separated text file called 'output_folder'/metadata/regionName_clusters, which shows the regions IDs in the first column and the assigned cluster
number in the second column. Now the initial dataframe is created for each region under 'output_folder'/regionName_df which has "ID" and "Cluster" as header, where cluster assignment is annotated as
regionName_clusterNumber to allow differentiation of clusters between regions.

#### STEP 6: Determining majority and potential outlier status for each cluster
**Script**: 6_annotate_clusterType.sh

For each region, cluster type is annotated as majority or potential_outlier, by determining the absolute size of the cluster and the percentage of the regions individuals the cluster contains and comparing
those sizes against 'outlier_threshold_absolute' and 'outlier_threshold_relative'. If a cluster contains no more samples then the former value or less percentage of the regions IDs than the latter, it
is considered a potential outlier cluster. The columns "Frac. Cluster/Region" and "Cluster type" are added to each regions dataframe.

#### STEP 7: Splitting clusters into periodic subclusters
**Script**: 7_split_upon_custom_periods.sh

The clusters of each region are split into periodic subclusters. To do so, each samples date in years BP is found from 'dates_file' and compared to the periods boundaries specified in
'custom_periods'. The columns "Date in years BP" and "Periodic subcluster" are added to each regions dataframe, the latter being annotated in the format regionName_periodName_clusterNumber.
If the date of a sample is not found in the provided 'dates_file', is not of Integer type or is outside of the periods boundaries, a respective tag will be put into the "Date in years BP" column and
the subcluster name will be regionName_NA_clusterNumber. If no 'dates_file' is provided, "Date in years BP" will be set NA and the subcluster name will be set as regionName_NA_clusterNumberAdditionaly,
effectively resulting in no subclassification.

#### STEP 8: Calculating relative sizes of periodic subclusters
**Script**: 8_calculate_subclusters_fractions.sh

The fraction of a periodic subcluster to the region as well as to the aperiodic main cluster is determined and the columns "Frac. subcluster/region" and "Frac. sucluster/cluster" are added to the data frame.

#### STEP 9: Manipulating .ind file to have subclusters as population IDs
**Scripts**: 9_update_indfile_to_periodicgroups.sh, manipulate_ind_1_9.R

To allow the comparison of found subclusters in qpadm models, a new .ind file is created that assigns each clustered individuals subcluster as its population ID. The current .ind with individual
population names is moved to 'output_folder'/metadata/FilesetPrefix.individuals_ind, with the new .ind file taking its place so it gets recognized as part of the fileset downstream. This new and final .ind
file, which annotates found suclusters as population labels, also gets copied to 'output_folder'/metadata/FilesetPrefix.clusters_ind.

#### STEP 10: Calculating f2 stats between all found subclusters
**Script**: 10_f2_extraction_periodic_groups.sh

F2 stats are computed between all subclusters of all regions and read into a cumulative .rds file. The subclusters f2 stats are saved to 'f2_directory'/crossregional_groupwise/subclusterName and the .rds file
to 'f2_directory'/f2blocks/crossregional_groupwise/all_regions.rds. If 'precompute_f2' is set to F, this step is omitted.

#### STEP 11: Computing p-values for outlier validation
**Scripts**: 11_compute_outlierValidation_pvalues_onF2data.sh, qpadm_oneComp_parallelized_onF2data_3_11_13.R

For every region, every one-component qpadm model consisting of a potential outlier and an in-region majority subcluster is run using the precomputed f2 data as input. The resulting pvalues are saved to a tab separated
file under 'output_folder'/metdata/regionName_pvalues_outlierValidaton, with the first column denoting the potetnial outlier, the second column a in-region majority subcluster and the third the resulting p-value.

**Scripts**: 11b_compute_outlierValidation_pvalues_onFileset.sh, qpadm_oneComp_parallelized_onFileset_3b_11b_13b.R

If 'precompute_f2' is set to F, the p-values for potential outlier - in-region majority models are computed using the fileset as input data for qpadm. Qpadm will be run with default arguments when this option is
chosen. As above, the results will be saved to a file for each region in 'output_folder'/metadata.

Warnings and Errors cast by the qpadm executions will collected saved to text files in 'output_folder'/metadata, to avoid cluttering of the console.

#### STEP 12: Validating outliers
**Script**: 12_validate_outliers_from_pvals.sh

To test whether potential outlier subcluster are truly distinct from regional majority ancestries, it is checked if they have any one-component qpadm model with an in-region majority cluster that had a
p-value above 'pvalue_threshold_validate'. If so, the potential outlier is cladal to the majority and is therefore no longer considered a potential outlier. The column "subcluster type" is added to the each dataframe,
denoting "majority", "outlier" or "connected to majority" for the respective cases. For each region, a file is created in 'output_folder'/metadata/regionName_connected subclusters shows every valid model
that was found in a region between a potential outlier and a in-region majority.


#### STEP 13: Computing p-values for ancestry search
**Scripts**: 13_compute_ancestrySearch_pvalues_onF2data.sh, qpadm_oneComp_parallelized_onF2data_3_11_13.R

For every outlier subcluster, one-component qpadm models between the outlier and every majority subcluster from a different region are calculated using the precomputed f2 data as input. The resulting p-values
are saved to a  tab separated file under 'output_folder'/metadata/regionName_pvalues_ancestrySearch, Storing outlier in the first, majority in the second and the resulting p-value in the third column.

**Scripts**: 13b_compute_ancestrySearch_pvalues_onFileset.sh, qpadm_oneComp_parallelized_onFileset_3b_11b_13b.R

If precompute f2 is set to F, the p-values for outlier - cross-region majority models are computed using the fileset as input data for qpadm. Qpadm will be run with default arguments when this option is
chosen. As above, the results will be saved to a file for each region in 'output_folder'/metadata.

Warnings and Errors cast by the qpadm executions will collected saved to text files in 'output_folder'/metadata, to avoid cluttering of the console.

#### STEP 14: Finding ancestries
**Script**: 14_find_potAncs_from_pvals.sh

If a one-conponent model of a outlier subcluster and a trans-region majority had a p-value above 'pvalue_threshold_ancestry', the majority is considered to be cladal to the outlier and is therefore considered
a potential source to the outlier. For each region, a tab separated file is created under 'output_folder'/metadata/regionName_potential_outlierSources, where each such instance is saved with the outlier in
the first and the potential source in the second column. If a has no potential sources found, its written into 'output_folder'/metadata/regions_wo_ancestries, to allow a user to quickly discern such cases.

#### STEP 15: Computing p-values for model-competition
**Scripts**: 15_compute_modelComp_pvals.sh, qpadm_modelComp_parallelized_onF2data_15.R

In this step, p-values for model-competition are computed. Model-competition is a process in which several potential sources that were found for one outlier are compteted against each other. Consider an
outlier T and its potential sources x and y. Model competition means that the fit of every potential source is retested while adding another potential source to the reference populations R. If the formerly
valid model of T and x is rejected when y is part of R, it suggests that there is some siginificant allele sharing between y and T that cannot be explained by x. Therefore x is considered suboptimal to y as
a source. To allow that process, every one-component qpadm model is run that consists of an outlier with more than one sources, one of its potential sources and the set of reference populations with one of its
other potential sources added, using precomputed f2 stats as input data. The resulting p-values are saved to 'output_folder'/metadata/regionName_pvalues_modelCompetition, where the first column denotes the
outlier, the second column the source used as left component, the third column the competitor rotated into the reference set and the fourth column the resulting p-value.

**Scripts**: 15b_compute_modelComp_pvals_onFileset.sh, qpadm_modelComp_parallelized_onFileset_15b.R

If 'precompute_f2' is set to F, the p-values for model competition are computed using the eigenstrat-fileset as input data for qpadm. Qpadm will be run with default arguments when this option is
chosen. As above, the results will be saved to a file for each region in 'output_folder'/metadata.

Warnings and Errors cast by the qpadm executions will collected saved to text files in 'output_folder'/metadata, to avoid cluttering of the console.

#### STEP 16: Model-competing multiple ancestry sources
**Script**: 16_find_suboptimalAncs_from_pvals.sh

Model-competition is achieved by checking for every source x of an outlier if it has a model_competition model where the model is rejected, in which case source x is written to a temporary list of suboptimal
sources. In an ideal scenario, some or all but one sources are suboptimal, and only the sources that are not suboptimal to any other are kept. However, due to the nature of qpadm, it can happen that all sources
are suboptimal to another one, i.e. if source x has some allele sharing with outlier T that cannot be explained by source y, but also y has some allele sharing (with other parts of the genome) with T that
cannot be explained by x, resulting in both being suboptimal to each other. In such a case, where every source of an outlier is suboptimal to another of its sources, all sources of the outlier are kept
to avoid information loss, and the problematic outlier written to 'output_folder'/metadata/problematic_modelComp. The remaining sources of every outlier are written to
'output_folder'/metadata/regionName_potential_outlierSources_modelComped, a tab separated text file which i every line denotes an outlier and one of its sources post model-competition.

#### STEP 17: Filtering multiple sources based on period
**Script**: 17_keep_contemp_or_older_sources.sh

In this final step, for every outlier only the sources that are from the same or an older period are kept. If an outlier only has sources from younger periods, they are kept as well to avoid information loss.
The final colum "Potential sources" is created, where for every sample belonging to an outlier cluster the remaining sources to its subcluster are listed, separated by a space.

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

## Program Versions used during Project
- **`GNU bash`** version 5.1.16
- **`R`** version 4.1.2
- **`Python 3`** version 3.12.7
- **`R libraries`**: dplyr v1.1.4, tidyverse v2.0.0, reticulate v1.38.0, filelock v1.0.3, devtools v2.4.5, admixtools v2.0.5
- **`Python packages`**: scipy v1.14.1


## Contact Info

If you have any remarks or questions, feel free to contact me via azhos@students.uni-mainz.de
