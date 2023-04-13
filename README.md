# tbng_veg
Vegetation analysis (primarily biomass mapping) for Thunder Basin National Grasslands.

## Objectives
This project has several sequential objectives. 
1. Predict herbaceous biomass from visual obstruction readings (VOR) - Erika
    * The main purpose of this is to increase the amount of ground-based estimates of biomass available to train satellite imagery (HLS). In Thunder Basin, there are many more VOR ground observations available compared to clipping, and the VOR estimates tend to represent larger areas, which is preferred for training 30 m satellite imagery. 
2. Predict herbaceous biomass from satellite imagery (HLS, and possibly other spatial co-variates, if necessary) - Erika
    * We want to be able to produce daily, near-real time maps of herbaceous biomass for Thunder Basin to enable adaptive managment and for use in other research projects. We will use the VOR readings and/or clipped biomass to train a model using satellite-derived vegetation indices and bands as predictors.
3. Predict clipped herbaceous biomass by functional group class from pin-frame measurements - Sean
    * This will enable plot-based monitoring and research related to plant communities and biomass over time. Pin-frame measurements are fairly rapid and widely used at Thunder Basin, but clipping by functional group is time-consuming and data are limited. We hope to use the limited clipping data to train a model to predict biomass from pin-frame measurements at the functional group level.

## Workflow description for Objectives 1 and 2
Objectives 1 and 2 are linked, with Objective 1 being a pre-cursor to 2. Therefore, those two objectives are combined into a single workflow here. The workflow for Objective 3 will be added below.
### Data preparation
Scripts used to prepare data for analysis are listed below, in the general order in which they must be run (loc_prep -> veg_prep). Also see ./TBNG veg QAQC and prep description.md (in progress...).
#### Location (./prep/loc_prep/)
Scripts in the directory ./prep/loc_prep clean and prepare the location data associated with a project.
##### *Inputs*
Currently, all inputs to the loc_prep scripts come from the T: drive.
##### *Outputs*
All outputs from the loc_prep scripts are saved to the .data/shps directory.
#### Vegetation (./prep/veg_prep/)
Scripts in the directory ./prep/veg_prep are run to clean and organize raw vegetation data files for additional analysis. There are three types of vegetation input data:
1. clipping
2. visual obstruction readings (VOR)
3. pin-frame measurements

The veg_prep steps also use shape files to add location data to the vegetation data when it is not already there. Therefore, if any preparation of location data is required, you must first have run the loc_prep scripts.

##### *Inputs*
Most inputs to the veg_prep scripts come from either the T: drive or from the ./data directory. Inputs are at the unit of sub-sample observation, so generally consist of multiple readings per experimental unit (e.g., plot).
##### *Outputs*
All outputs from the veg_prep scripts are aggregated as plot-scale means after cleaning of individual observations. They are written to sub-directories as ./data/\* (however, note that there are other sub-directories here that are outputs of other scripts). The sub-directories related to vegetation data are organized as follows:
* ./data/clip: all cleaned plot-scale clipping data (available for training satellite model - see ./tb_bm_modeling.ipynb and ./tb_bm_model_sel.ipynb)
* ./data/vor: all cleaned plot-scale vor data (available for training satellite model - see ./tb_bm_modeling.ipynb and ./tb_bm_model_sel.ipynb)
* ./data/vor_train: cleaned plot-scale vor and clipping data for plots and dates where both were taken (available for training the VOR model - see tb_vor_linreg.ipynb)
* ./data/pf_train: cleaned plot-scale pin-frame and clipping data for plots and dates where both were taken at the functional group level (available for training the pin-frame model - see tb_pf_linreg.ipynb; saved in both wide and long form)

### Main scripts
These are the main scripts, to be run after preparing all ground data. They generally need to be run in the order presented.
#### Fit VOR regression (tb_vor_linreg.ipynb)
Fits and saves the linear regression model used to predict herbaceous biomass from ground-based VOR measurements. The model is saved to the ./models directory using *pickle* for easy application later.
##### *Notes:*
* There are currently some added steps to pull in PRISM precip data, which we have been exploring as a co-variate. This portion of the script should probably be moved to its own separate script, perhaps in the ./prep directory.
#### Combine all ground data (tb_ground_combine.ipynb)
Combines into a single dataset all the ground-based VOR and clipping biomass data available for training satellite imagery. 
##### *Output*
The output is saved to ./data/bm_extract/TB_all_bm.csv.
##### *Notes:*
* The script will apply the saved VOR *pickle* model to the VOR data, thus creating an estimate of herbaceous biomass derived from the VOR data
* A subset of the clipping and VOR data are from the same plots on the same dates - they were used to produce the VOR regression! We include all of them in this single dataset in case we want to train/validate the satellite data using only VOR or only clipped data. If we choose to train/validate the satellite using all available data, we must decide what to do with these overlapping data (e.g., keep only VOR, keep only clipping, or average the two).
#### Extract HLS indices/bands for each plot (tb_extract_hls.ipynb)
Pulls Harmonized Landsat-Sentinel (HLS) data from LPDAAC and extracts mean vegetation indices and individual bands for each plot on each date. HLS data are pulled for a 150 m buffer around the plot center (all TB plots are treated as point data) and then averaged for the four closest cells to the plot center. All HLS data are despiked, interpolated and smoothed during this process. The script is currently quite slow as written as it loops through each plot-date combo individually, which requires reading from LPDAAC many times. The script is written in this way to deal with the NEX project, for which we can't simply take the nearest four cells to the plot center, since plots are adjacent to fenclines that separate differnet exclosure treatments. Instead the script takes the four closest cells within the same treatment. With some work, this script could be sped up dramatically.  
##### *Output*
The output is saved to ./data/bm_extract/TB_all_bm_veg_idxs.csv. The output is a CSV file identical to TB_all_bm.csv but with added columns representing the mean value for each HLS index/band.
##### *Notes:*
* Relies on the hls-nrt-utils package.
* Re-writes the output after processing each plot-date combo. This way, if there is an unresolvable error during processing, the script can be re-run and pick up where it left off by loading the saved dataset.
* Includes additional code after processing to perform manual checks.
#### Predict biomass from HLS (tb_bm_modeling.ipynb)
[In progress!]  
Uses the output from the HLS extraction (./data/bm_extract/TB_all_bm_veg_idxs.csv) to predict biomass using the model developed at CPER (see Kearney et al., 2022). First, the CPER model is applied directly to the TB data. Then, a new model is fit to the TB data, but using the same HLS index/band inputs that perfored best at CPER. Finally, a model selection procedure is carried out trying all (appropriate) combinations of HLS inputs and cross-validated using various different approaches.  
##### *Notes:*
This script is incomplete and messy! It includes many steps to visualize the data (at the beginning) and look for possible errors or outliers. The model selection procedure has not been finalized. We need to come up with a good way to perform cross-validation given the complex data coverage that results from combining many different projects. Some of the exploration at the end of the script is trying think about variability in year vs. project vs. location.
* This step currently incorporates the next step, bringing in soil texture as a co-variate. It simply reads in a .csv file that has extracted soil texture for each plot, and joins this to the full dataset with the HLS extraction. It can then be explored as a covariate during modeling.
* Also note that tb_bm_model_sel.ipynb was the beginning of rewriting the model-selection procedure, pulling from the CPER script. This script will likely not run as written as it is even less complete than tb_bm_modeling.ipynb.
#### Bring in additional co-variates
[In progress!]
This step has not been fully developed. But in general, here we would extract additional covariates, and then refit biomass models using the HLS inputs along with one or more co-variates. Currently, a rough workflow has been developed to do this for soil texture class (from SSURGO data) as follows:
* Prepare SSURGO data using ./prep/covar_prep/tb_process_SSURGO.ipynb. This script will create a shapefile of soil texture classes for the TB area.
* Extract SSURGO soil texture classes for each plot using ./tb_extract_SSURGO.ipynb. This is a very simple script that only relies on geopandas. It should work (with adaptation) to extract any shapefile data for the plots. Output is saved to ./data/bm_extract/TB_all_bm_soiltexture.csv.
* Refit models with soil texture class as a covariate. Simply join the data with the newly extracted covariate to the full dataset that includes the HLS extraction. Then the covariate can be added as input during modeling, for example using tb_bm_modeling.ipynb (or a modified copy).


    