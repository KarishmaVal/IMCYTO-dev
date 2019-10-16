# IMCYTO-dev

# Pipeline Documentation

-> HAVE ANOTHER DOCUMENT AS OUTPUTS.MD FOR THE OUTPUT FILES (CELLMASK, CSV) ETC WITH PIPELINE REPORTS AT THE END AS OTHER OUTPUTS (CAN COPY THE EXPLANATION FROM HARSHIL'S OTHER PROJECTS IN GITHUB

## Introduction:

This is an automated pipeline for preprocessing and segmenting imaging data from Imaging Mass Cytometry to obtain single cell information. 

This pipeline was written by Harshil Patel and Nourdine Bah at the Francis Crick Institute, using dockers/singularity/nextflow É to package together various image analysis tools É batch processing/parallelised/fast/flexible 

This pipeline is designed to run on a server/cluster without the need to pre-install any of the software packages. However, to modify/customise the pipeline, one needs to install the [CellProfiler(v3.1.8)](https://cellprofiler.org/ 'CellProfiler') and [Ilastik(v1.3.3)](https://www.ilastik.org/ 'Ilastik') softwares on a local machine (see **Pipeline adaptations** section).  

The concept of the stepwise image segmentation using a pixel classifier (Ilastik) to generate a membrane probability map combined with CellProfiler modules for subsequent single cell mask generation, is based on the analysis pipeline as described by the Bodenmiller group [(Zanotelli & Bodenmiller, Jan 2019)](https://github.com/BodenmillerGroup/ImcSegmentationPipeline/blob/development/documentation/imcsegmentationpipeline_documentation.pdf). 
The plugins supplied with the pipeline constitute the minimal requirements to generate a single cell mask. A more refined and comprehensive pipeline will be uploaded in due course.

## Pipeline summary:

1. Generate .tiff files from image acquisition output: Open .mcd or .txt files and contained ROIs and save individual .tiff files for channels with names matching those defined in a metadata.csv file into corresponding folders – full_stack for all channels being analysed in single cell expression analysis; ilastik_stack for channels being used to segment cells (IMCTools)
2. Preprocess full_stack images: Apply preprocessing filters to all .tiff files in the full_stack subfolder and save (CellProfiler – full_stack_preprocessing.cppipe)
3. Generate a cell map from ilastik_stack images: Merge .tiff images from the ilastik_stack subfolder to obtain an RGB image of cell nuclei and membranes and save as a composite.tiff (CellProfiler – ilastik_stack_preprocessing.cppipe)
4. Apply pixel classification to the composite cell map: Use composite.tiff to classify pixels as membrane, nuclei or background, and save probability maps as .tiffs (Ilastik)
5. Generate a single cell mask: Use probability .tiffs and preprocessed full_stack .tiffs for single cell segmentation to generate a cell mask (CellProfiler – segmentation.cppipe)
6. Output single cell expression data: Overlay cell mask onto full_stack tiff images to extract single cell information generating a csv file (CellProfiler – segmentation.cppipe)

## Pipeline workflow schematic:
 
[Workflow schematic](images/schematic.pdf)

[Example folder structure](images/example_folder_structure.png)

