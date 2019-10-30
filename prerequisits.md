# IMCYTO-dev

# Pipeline Documentation

-> HAVE ANOTHER DOCUMENT AS OUTPUTS.MD FOR THE OUTPUT FILES (CELLMASK, CSV) ETC WITH PIPELINE REPORTS AT THE END AS OTHER OUTPUTS (CAN COPY THE EXPLANATION FROM HARSHIL'S OTHER PROJECTS IN GITHUB

## Introduction:

This is an automated pipeline for pre-processing and single cell segmentation of imaging data, generated using Imaging Mass Cytometry data but is flexible enough to be applicable to various types of imaging data (eg. confocal). 

This pipeline allows input data in the format of mcd, ome.tiff or txt files from which stacks of tiff files are generated for subsequent analysis. The various stages of this pipeline allow the tiff images to be pre-processed, and segmented using multiple CellProfiler .cppipe project files and the pixel-classification software Ilastik. The concept of this stepwise image segmentation combining Ilastik with CellProfiler was based on the analysis pipeline as described by the Bodenmiller group [(Zanotelli & Bodenmiller, Jan 2019)](https://github.com/BodenmillerGroup/ImcSegmentationPipeline/blob/development/documentation/imcsegmentationpipeline_documentation.pdf). 

The plugins supplied with the pipeline constitute the minimal requirements to generate a single cell mask. A more refined and comprehensive pipeline will be uploaded in due course.

This pipeline was automated by Harshil Patel and Nourdine Bah at the Francis Crick Institute, using dockers/singularity/nextflow to package together various image analysis tools batch processing/parallelised/fast/flexible 
This pipeline is designed to run on a server/cluster without the need to pre-install any of the software packages. However, to modify/customise the pipeline, one needs to install the [CellProfiler(v3.1.8)](https://cellprofiler.org/ 'CellProfiler') and [Ilastik(v1.3.3)](https://www.ilastik.org/ 'Ilastik') softwares on a local machine (see **Pipeline Adaptations** section).  

## Pipeline Summary:

1. **(IMCTools)** Generate .tiff files from image acquisition output: Open .mcd, .ome.tiff or .txt files and contained ROIs and save individual .tiff files for channels with names matching those defined in a metadata.csv file into corresponding folders – full_stack for all channels being analysed in single cell expression analysis; ilastik_stack for channels being used to generate the cell mask.

2. **(CellProfiler – full_stack_preprocessing.cppipe)** Preprocess full_stack images: Apply preprocessing filters to all .tiff files in the full_stack folder and save.

3. **(CellProfiler – ilastik_stack_preprocessing.cppipe)** Generate a composite image representive of all cells plasma membranes: Merge select .tiff images from the ilastik_stack subfolder to create a composite RGB image of cell nuclei and plasma membranes and save as a composite.tiff.

4. **(Ilastik)** Apply pixel classification to the composite cell map: Use composite.tiff to classify pixels as membrane, nuclei or background, and save probability maps as .tiffs.
*Alternative option to skip ilastik pixel classification and use composite.tiff instead of probabilty .tiff in subequent steps. This approach might be preferred if CellProfiler modules alone are deemed sufficient to achieve a reliable segmentation mask.*

5. **(CellProfiler – segmentation.cppipe)** Generate a single cell mask: Use probability .tiffs and pre-processed full_stack .tiffs for single cell segmentation to generate a cell mask. 

6. **(CellProfiler – segmentation.cppipe)** Output single cell expression data: Overlay cell mask onto full_stack .tiff images to extract single cell information generating a csv file.

## Pipeline Workflow Schematic:
 
![Workflow schematic](images/schematic.pdf)

## Pipeline Prerequisites: 

- **.mcd**, **.ome.tiff** or **.txt** data file, generated without any spaces in the file name. Associated antibody panel should contain metal and antibody information in the form of ‘metal_antibody’ (eg. 89Y_CD45).

- **metadata.csv** file containing your antibody panel to identify which corresponding .tiff files are to be put into full_stack and ilastik_stack folders (example shown in **Pipeline Workflow Schematic**). File should contain only three columns titled ‘metal’, ‘full_stack’ and ‘ilastik_stack’. ‘metal’ column should contain all the metals used in your antibody panel. ‘full_stack’ and ‘ilastik_stack’ columns should be used to identify which metals to include in boolean labelling (1 = use or 0 = don’t use).

- **‘NamesAndTypes’** module in all CellProfiler .cppipe files (see **Pipeline Adaptations**) will need to be edited to match your antibody panel and desired markers to identify cell nuclei and membranes. Other recommended changes to the pipeline are outlined below - **Pipeline Details**.

## Pipeline Adaptations:

- To view and edit the .cppipe files, download [CellProfiler(v3.1.8)](https://cellprofiler.org/ 'CellProfiler') as well as the custom plugins created by Bodenmiller group - smoothmultichannel.py and measureobjectintensitymultichannel.py (found [here](https://github.com/BodenmillerGroup/ImcPluginsCP 'CellProfiler Bodenmiller custom plugins')). Your own custom plugins can also be used. Open CellProfiler>Preferences and change ‘CellProfiler plugins directory’ path to where you stored the custom plugins. When saving the edited .cppipe files, make to sure export the file as a .cppipe and to name the files as either ‘full_stack_preprocessing’, ‘ilastik_stack_preprocessing’ or ‘segmentation’ in line with the schematic above. 

- If you use any custom modules in any of the CellProfiler .cppipe files, make sure to add the python script into the ‘plugins/cp_plugins’ directory before running the pipeline (described below – **Running the Pipeline**).

- The pipeline is packaged with a pre-trained Ilastik pixel classifer but to train a classifier using your own dataset download [Ilastik(v1.3.3)](https://www.ilastik.org/ 'Ilastik'), you will need to input desired composite RGB images in .tiff format and use ‘membrane’, ‘nuclei’ and ‘background’ labels to train your classifer. The input image should be the output of ‘ilastik_stack_preprocessing’. When selecting export settings select source as ‘Probabilities’, transpose axis order to ‘cyx’ and export output file as ‘tiff sequence’ to generate probability maps. The Ilastik file should be titled ‘ilastik_training_params.ilp’.

- Adding to the flexiblity of this pipeline we have included a --skip_ilastik flag in the run.sh script as an optional argument that can be included to skip the Ilastik pixel classification step and proceed via a CellProfiler only pipeline. In this case, no --ilastik_training_ilp is provided in the run.sh script and images that have been preprocessed through ‘ilastik_stack_preprocessing.cppipe’ will be directly passed to the 'segmentation.cppipe' bypassing Ilastik. This approach might be preferred if CellProfiler alone is deemed sufficient to achieve a reliable segmentation mask.

- Additionally, --compensation_tiff is an optional argument to the pipeline, included if the user wishes to apply a predetermined spillover compensation function or illumination function to images during one or both of the ilastik_stack and full_stack preprocessing steps. If a compensation .tiff file is provided, it should be saved in the mcd folder along with the images to be processed and metadata (described below – **Running the Pipeline**). Examples of compensation might include: illumination correction to microscopy images, spillover compensation for imaging mass cytometry data. 

## Pipeline Details:

Each step of the pipeline as depicted in the schematic is broken down and explained below, with the inputs and outputs highlighted.

### 1. IMCTools
**Input:** .mcd, .ome.tiff or .txt file and metadata.csv file.
- Opens data files and any contained ROIs and converts each channel into individual .tiff files.
- Uses metadata.csv file to sort .tiffs into into corresponding folders (full_stack/ilastik_stack).

**Output:** full_stack and ilastik_stack folders containing .tiffs.

### 2. full_stack_preprocessing
**Input:** full_stack of .tiff images generated by IMCTools step.
- This step selects all images in full_stack folder and sequentially processes them through various filtering methods, including removal of hot pixels and median filter, and then saves all files.
- These image filtering parameters can be changed by opening and customising the cppipe file in CellProfiler.
- To keep your data raw, uncheck all the modules except ‘SaveImages’. This will simply re-save the unchanged input images into the correct place for the next step in the pipeline.
- Make sure to export a .cppipe file titled ‘full_stack_preprocessing’.

**Output:** full_stack folder containing processed .tiffs.

### 3. ilastik_stack_preprocessing
**Input:** ilastik_stack of .tiff images generated by IMCTools step.
- This step first selects specific files by finding matching names and labels them as either ‘membrane’ and ‘nuclei’. These images are then filtered and merged to create an RGB composite image, which is saved as composite.tiff.
- Open .cppipe file in CellProfiler to change names of input images in ‘NamesAndTypes’. Select markers that are representative of total cell plasma membranes and cell nuclei. If one marker does not cover all cell type membranes, then various membrane markers can be merged by adding in an ‘ImageMath’ module to create a total membrane image, this image will need to be titled ‘membrane’.
- Further parameters can be customised, such as the filtering methods and parameters.
- Make sure to export a .cppipe file titled ‘ilastik_stack_preprocessing’.

**Output:** ilastik_stack folder containing a .tiff image titled ‘composite’.

### 4. ilastik_training_params
**Input:** composite.tiff generated by ilastik_stack_preprocessing step.
- Pixels are classified into three sets: membrane, nuclei and background by Ilastik using a pretrained pixel classifer to generate and save a probability map of each.
- If required, the ilastik classifier can be retrained using a new dataset. These inputs should be the output composite image of ‘ilastik_stack_preprocessing’ step. When selecting export settings, select source as ‘Probabilties’, transpose axis order to ‘cyx’ and export output file as ‘tiff sequence’ to generate probability maps.
- Make sure to save ilastik classifier as ‘ilastik_training_params.ilp’.
- To skip the Ilastik pixel classification altogether, a --skip_ilastik flag in the run.sh script can be included. In this case, no --ilastik_training_ilp is provided in the run.sh script and images that have been preprocessed through ‘ilastik_stack_preprocessing.cppipe’ will be directly passed to the 'segmentation.cppipe' bypassing Ilastik.

**Output:** three .tiff images titled ‘composite_Probabilities_’ ending in either 0 (for membrane), 1 (for nuclei) or 2 (for background).

### 5. segmentation
**Input:** preprocessed full_stack of .tiff images generated by full_stack_preprocessing step and composite_Probabilities_ generated by Ilastik or composite.tiff if Ilastik is skipped.
- Open .cppipe file in CellProfiler to change names of input images in ‘NamesAndTypes’ to match your antibody panel, making sure to keep ‘_Probabilities_0’ as prob_membrane and ‘_Probabilities_2’ as prob_background.
- Currently, this process uses the nuclear image to identify nuclei as primary objects, subtracts background probability from the membrane probability image and then uses both identfied nuclei and membrane porbabily to identify whole cells as secondary objects. These cell objects are then converted into a unit16 image to generate the cell mask and saved. The cell mask is then used to measure size and shape information of all cell objects, and extract single cell expression data.
- You may need to adapt parameters present in ‘IdentifyPrimaryObjects’ and ‘IdentifySecondaryObjects’ modules to best suit your data. Common changes include typical diameter size in pixels of nuclear objects, thresholding strategy and method to distinguish clumped objects. Alternatively, one can opt for using the nuclei probability output (‘_Probabilities_1) from Ilastik to identify the nuclei as primary objects.
- Within ‘MeasureObjectIntensity Multichannel’ you will need to select all the images that you would like the intensity to be measured for.
- Further parameters can be customised, such as the desired measurements to export in ‘ExportToSpreadsheet’. Currently this pipeline exports data only for each cell object: area, mean intensity, object center location and object number.
- Make sure to export a .cppipe file titled ‘segmentation’.

**Output:** .tiff image titled ‘Cells_mask’ and ‘Cells.csv’ file containing single cell data.

## Running the pipeline:

To run the pipeline, you will need to login into cluster … 
The .mcd file and metadata.csv should be put together in a folder titled ‘mcd’. A ‘plugins’ folder should contain all three CellProfiler files (i.e. full_stack_preprocessing.cppipe, ilastik_stack_preprocessing.cppipe, segmentation.cppipe), ilastik (ilastik_training_params.ilp) parameters file as well as any extra custom CellProfiler modules (.py) in ‘cp_plugins’ folder. The run_imcyto script should be put along with these folders. Once you are ready to run the pipeline excute the sh run.imcyto command, from within the main experiment folder.

**Example folder structure:**

![Example folder structure](images/example_folder_structure.png)

## How the pipeline runs:

Pull latest version from Github, docker containers, online/offline, run pipeline... 

## Pipeline reporting:

Once your pipeline has run successfully it will generate a range of reports alongside the pipeline outputs …
Example image of what the output folder structure looks like?

## Common errors and solutions:

- When the pipeline fails to run, first try to rerun it with command -resume

- The most common source of error is in the naming of the channels. Check the metadata.csv and the CellProfiler 'NamesandTypes' module. Make sure all metal names are correct and unambiguous (eg. CD4 will find both CD4 and CD44 in 'NamesandTypes'). 

- For ROIs larger than ~2000-2000um the pipeline may struggle/fail to compete due to reaching memory capacity during ‘IdentifyObjects’ modules in ‘segmentation.cppipe’. As a workaround for this we recommend adding in a ‘Crop’ module before identifying objects to crop your images into two. Make sure to duplicate any subsequent modules and adjust input names as required.

## Other related software packages:

- [MCD Viewer](https://www.fluidigm.com/software 'MCD Viewer') software provided by the Hyperion imaging system allows users to visualise, review and export .mcd image files.

- [Fiji-ImageJ](https://imagej.net/Fiji 'Fiji') can also be used to view .mcd files but requires downloading IMCTools plugin. This plugin allows the .mcd file to be viewed as an image5D with your antibody markers as individual channels that can be toggled on/off.

- Additionally, [QuPath](https://qupath.github.io/ 'QuPath'), can also be used for visualisation with the added benefit of viewing all channels at once in a mini viewer panel.

## Hyperlinks:
- Zanotelli & Bodenmiller, Jan 2019: https://github.com/BodenmillerGroup/ImcSegmentationPipeline/blob/development/documentation/imcsegmentationpipeline_documentation.pdf 
- CellProfiler: https://cellprofiler.org/
- CellProfiler Bodenmiller custom plugins: https://github.com/BodenmillerGroup/ImcPluginsCP 
- Ilastik: https://www.ilastik.org/
- MCD Viewer: https://www.fluidigm.com/software 
- Fiji-ImageJ: https://imagej.net/Fiji 
- IMCTools: https://github.com/BodenmillerGroup/imctools
- QuPath: https://qupath.github.io/ 

