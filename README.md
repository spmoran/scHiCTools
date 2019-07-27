# scHiCTools

### Summary
A computational toolbox for analyzing single cell Hi-C (high-throughput sequencing for 3C) data which includes functions for:
1. Load single-cell HiC datasets
2. Smoothing the contact maps with linear convolution, random walk or network enhancing
3. Calculating embeddings for single cell HiC datasets efficiently with reproducibility measures include InnerProduct, fastHiCRep and Selfish

### Installation
  **Required Python Packages**
  - Python (version >= 3.6)
  - numpy (version >= 1.15.4)
  - scipy (version >= 1.0)
  
  **Install from GitHub**
  
  You can install the package with following command:
  ```console
  % install
  ```
  Or directly download "scHiCTools-master.zip" from GitHub.
  
### Usage
  **Import Package**
  ```console
  >>>import scHiCTools
  ```
  
  **Load scHiC data**
  
  The scHiC data is stored in a series of files, with each of the files corresponding to one cell.
  You need to specify the list of scHiC file paths.
  
  Only intra-chromosomal interactions are counted.
  ```console
  >>>from scHiCTools import scHiCs
  >>>files = ['./cell_1', './cell_2', './cell_3']
  >>>loaded_data = scHiCs(
  ... files, reference_genome='mm9',
  ... resolution=500000, max_distance=4000000, normalization=None,
  ... format='customized', adjust_resolution=True,
  ... line_format=12345, header=False, chromosomes='no Y',
  ... preprocessing=['reduce_sparsity', 'convolution'],
  ... kernel_shape=3
  ... )
  ```
  - reference genome: now supporting 'mm9', 'mm10', 'hg19', 'hg38'.
  If your reference genome is not in ['mm9', 'mm10', 'hg19', 'hg38'], you need to provide the lengths of chromosomes
  you are going to use with a Python dict. e.g. {'chr1': 150000000, 'chr2': 130000000, 'chr3': 200000000}
  - resolution: the resolution to separate genome into bins.
  If using .hic file format, the given resolution must match with the resolutions in .hic file.
  - max_distance: only consider contacts within this genomic distance. Default: None.
  If 'None', it will store full matrices in scipy sparse matrix format, which will use too much memory sometimes
  - normalization: whether to normalize or what method to use.
  Methods include: 'OE' (observed / expected), 'VC', 'VC_SQRT', 'KR'. Default: None
  - format: file format, '.hic' or 'customized', if 'customized', you need to
  provide the format for each line in argument 'line_format'.
  - adjust_resolution: whether to adjust resolution for the input file.
  Sometimes the input file is already in the proper resolution (e.g. position 3000000 has already been changed to 6 with 500kb resolution).
  For this situation you can set adjust_resolution=False. Default: True.
  - header: whether the files have a header line. Default: False.
  - line_format: the column indices in the order of "chromosome 1 - position 1 - chromosome 2 - position 2 - contact reads".
  e.g. if the line is "chr1 5000000 chr2 3500000 2", the format should be '12345' or [1, 2, 3, 4, 5]; if there is no column
  indicating number of reads, you can just provide 4 numbers like '1234', and contact read will be set as 1.
  Default: '12345'.
  - chromosomes: chromosomes to use, eg. ['chr1', 'chr2'], or just 'except Y', 'except XY', 'all'.
  Default: 'all', which means chr 1-19 + XY for mouse and chr 1-22 + XY for human.
  - preprocessing: the methods use for pre-processing or smoothing the maps given in a list.
  The operations will happen in the given order.
  Operations: 'reduce_sparsity', 'convolution', 'random_walk', 'network_enhancing'.
  Default: None.
  - For preprocessing and smoothing operations, sometimes you need additional arguments (introduced in next sub-section).
  
  You can also skip pre-processing and smoothing in loading step (preprocessing=None).
  But if you didn't store full map (i.e. max_distance=None), longer-range contacts will not be stored
  thus there might be some bias if you do smoothing in next steps.
  
  Thus, preprocessing and smoothing during loading data is recommended.

  **Pre-processing and Smoothing**
  ```console
  >>>processed_data = loaded_data.preprocessing(
  ... ['reduce_sparsity', 'convolution', 'network_enhancing'],
  ... kernel_shape=3,
  ... kNN = 20
  ... )
  ```
  - reduce_sparsity: Hi-C reads in a single contact map usually varies between several magnitude
  (often depends on genomic distance), taking logarithm or powers might make it easy for later calculation.
  Arguments include:
    - sparsity_method: 'log' [new_W_ij = log(W_ij + 1)], thus 0 in original matrix is stll 0 in processed matrix)
    or 'power' [new_W_ij = (W_ij)^pow]. Default: 'log'
    - power: (if you choose sparsity_method='power') a number usually between 0 and 1.
    e.g. power=0.5 means all values W_ij in contact map will be changed to (W_ij)^0.5
    (i.e. sqrt(W_ij)). Default: 0.5.
  - convolution: smoothing with a N by N convolution kernel, with each value equal to 1/N^2
  Argument:
    - kernel_shape: an integer. e.g. kernel_shape=3 means a 3*3 matrix with each value = 1/9. Default: 3.
  - Random walk: multiply by a transition matrix (also calculated from contact map itself).
  Argument:
    - random_walk_ratio: a value between 0 and 1, e.g. if ratio=0.9, the result will be
    0.9 * matrix_after_random_walk + 0.1 * original_matrix. Default: 1.0.
  - Network enhancing: transition matrix only comes from k-nearest neighbors of each line.
  Arguments:
    - kNN: value 'k' in kNN. Default: 20.
    - iterations: number of iterations for network enhancing. Default: 1
    - alpha: similar with random_walk_ratio. Default: 0.9
  - normalization: matrix normalization.
   Argument:
    - normalization_method: 'OE' (observed / expected), 'VC', 'VC_SQRT', 'KR'
  
  **Learn Embeddings**
  ```console
  >>>embs = loaded_data.learn embedding(
  ... dim=2, method='inner_product',
  ... max_distance=None, aggregation='median',
  ... return_distance=False
  ... )
  ```
  This function will return the embeddings in the format of a numpy array with shape (#_of_cells, #_of_dimensions).
  - dim: the dimension for the embedding
  - method: reproducibility measure, 'InnerProduct', 'fastHiCRep' or 'Selfish'. Default: 'InnerProduct'
  - max_distance: only consider contacts within this genomic distance. Default: None.
  If None, it will use the 'max_distance' in the previous loading data process, thus if you
  set 'max_distance=None' in previous data loading, you must specify a max distance for this step or it will raise an error.
  - aggregation: method to aggregate different chromosomes, 'mean' or 'median'. Default: 'median'.
  - return_distance: (bool) if True, return (embeddings, distance_matrix); if False, only return embeddings. Default: False.
  - Some additional argument for Selfish:
    - n_windows: split contact map into n windows, default: 10
    - sigma: sigma in the Gaussian-like kernel: default: 1.6

### Citation


### References
A. R. Ardakany, F. Ay, and S. Lonardi.  Selfish: Discovery of differential chromatininteractions via a self-similarity measure.bioRxiv, 2019.

N. C. Durand, J. T. Robinson, M. S. Shamim, I. Machol, J. P. Mesirov, E. S. Lander, and E. Lieberman Aiden. "Juicebox provides a visualization system for Hi-C contact maps with unlimited zoom." Cell Systems 3(1), 2016.

J. Liu, D. Lin, G. Yardimci, and W. S. Noble. Unsupervised embedding of single-cellHi-C data.Bioinformatics, 34:96–104, 2018.

T. Nagano, Y. Lubling, C. Várnai, C. Dudley, W. Leung, Y. Baran, N. M. Cohen,S.  Wingett,  P.  Fraser,  and  A.  Tanay.    Cell-cycle  dynamics  of  chromosomal organization at single-cell resolution.Nature, 547:61–67, 2017.

B. Wang, A. Pourshafeie, M. Zitnik, J. Zhu, C. D. Bustamante, S. Batzoglou, andJ.  Leskovec.   Network  enhancement  as  a  general  method  to  denoise  weighted biological networks.Nature Communications, 9(1):3108, 2018.

T. Yang, F. Zhang, G. G. Y. mcı, F. Song, R. C. Hardison, W. S. Noble, F. Yue, andQ. Li. HiCRep: assessing the reproducibility of Hi-C data using a stratum-adjusted correlation coefficient.Genome Research, 27(11):1939–1949, 2017.

G.  G.  Yardımcı,   H.  Ozadam,   M.  E.  Sauria,   O.  Ursu,   K. K.  Yan,   T.  Yang,A. Chakraborty, A. Kaul, B. R. Lajoie, F. Song, et al. Measuring the reproducibilityand quality of hi-c data.Genome Biology, 20(1):57, 2019.

