Can we identify the flights and perches of thought (James, 1890) directly from the brain? We showed that the network transition timepoints identified by this pipeline are aligned to high-level cognitive state changes and suggest that characterizing these transitions is a method to characterize mentation (i.e., the act of thinking) directly from neural signal [(Tseng & Poppenk, 2019)](https://www.biorxiv.org/content/10.1101/576298v4). The following pipeline will take multidimensional timeseries data, use a non-linear dimensionality reduction technique (t-SNE) to project the high dimensional data onto a two-dimensional state space, and identify potential jumps of interest from this reduced space. 

- [System Requirements](#system-requirements)
- [Installation Guide](#installation-guide)
- [How to Use](#how-to-use)
- [Demo](#demo)
- [Network Representation](#network-representation)

## System Requirements

This pipeline was originally developed on MATLAB (r2017a) and run on a Linux server. If the Parallel Computing Toolbox is available to you, then `parfor` is used to process multiple participants' data at once in the first reducing dimensionality step. No custom/non-standard hardware was used.

## Installation Guide

Download this function library and ensure that the path to its folder is visible to MATLAB. 

## How to Use

The following section provides pseudocode for the steps carried out in each function. If you are looking for how to obtain the 15-network representation of brain activity from your 4D functional data, see the [Network Representation](#network-representation) section at the end of this README. The code enclosed here assumes that you have already generated network representation .txt files from FSL's dual regression function.

`reduceDim.m`

1. generate cell array of subject IDs based on the data folder
2. for each participant
   + for each run
     + load the .txt file containing their network representation data (time x 15 networks)
     + apply temporal smoothing with a span of 5 TRs
     + for the requested number of repetitions
       + execute t-SNE to reduce dimensionality to 2
       + save this 2D output
3. return: 
   + reducedData: a (num_participants x num_runs) cell array
     + each cell contains a matrix of shape (num_iterations x num_timepoints x 2)
   + subjID: a cell array containing the subject IDs, useful to carry forward
   
`jumpCalculator.m`

0. Input = output of reduceDim. 
1. for each run
   + for each participant
      + for each iteration
         + calculate the Mahalanobis distance between consecutive timepoints
      + save the mean step distance vector across repeated iterations
2. return:
   + distance: a cell array with shape (1 x num_runs), where each cell contains a matrix with compiled mean step distance vectors across participants of shape (num_participants x num_timepoints)
   
`findManyPks.m`

0. Input = output of jumpAnalyzer
1. if no minimum peak prominence (MPP) value has been specified, use the 80th percentile step distance value
2. if no stability width threshold has been specified, use 10 TRs
3. for each run
   + identify transitions 
   + identify meta-stable timepoints
4. return:
   + pks: a struct with a separate field for each run
      + pks.runX.idx: cell array of length (num_participants), where each cell contains an array of values associated to a transition timepoint
      + pks.runX.bin: binary matrix of size (num_participants x num_timepoints) where 1s indicate a transition timepoint, 0 otherwise
      + pks.runX.base_idx and pks.runX.base_bin: analogs of the above, but with the meta-stable timepoints
   
`analyzeTrajectory.m`

0. Input = pks output from findManyPks
1. for each run
   + count up the number of transitions
   + divide by the time elapsed in minutes
2. return:
   + array of shape (num_participants x num_runs) containing transition rate per minute

`groupAlign.m`

If your data is task-based and you wish to calculate the similarity between participants' step distance vectors, use this code. 
0. Input = mean step distance vectors from jumpCalculator
1. for each run
   + for each participant
     + isolate their mean step distance vector
     + calculate the median step distance vector across all _other_ participants
     + take the correlation between the individual and group step distance vectors
2. return:
   + array of shape (num_participants x num_runs) containing conformity for each run

## Demo

A demo of the pipeline can be executed with the `demo_main.m` run code and the five simulated participant data found in the `demo/data` folder. The data was generated by phase randomizing (same shift across voxels) HCP 7T participants' movie run 1 functional data, and then running dual regression to transform functional data to network timeseries.

`demo_main.m`

This master run code defines the workflow of the analysis. Modify the variables at the top with information relevant to your dataset (e.g., path to `.txt` files, number of runs). With the Parallel Computing Toolbox, 5 participants, 1 run with 900 timepoints, and 100 requested t-SNE repetitions, tic/toc returned an elapsed time of 6.74 minutes to run the full pipeline described above and return transition rate. 

## Network Representation

Should you wish to transform fMRI data into the same network space as in our paper, you will require:
+ Spatial maps from the Human Connectome Project (HCP) that are the output of group-ICA and group-PCA on the 3T resting state dataset
+ FSL's [dual regression function](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/DualRegression)
+ Your 4D fMRI data

Then, steps are as follows:
1. Match the resolution of the spatial maps from HCP to that of your functional data. 
2. Use FSL's command-line dual regression function to regress the spatial maps into your functional data. The key output here is the stage 1 output, which is a `dr_stage1_*.txt` file containing a row for each TR and a column for each network. 
