**Project 1: Comparing Sentinel-2 and PlanetScope Random Forest Models for Lake Ice-On Date Estimation Across Three Alaskan Study Sites (2019–2025)**

**1 Introduction**

Approximately 86% of the 117 million lakes worldwide are estimated to freeze on an annual basis. (Verpoorter et al. 2014, Korver et al. 2024\) However, warming of winter air temperatures across the Arctic is leading to widespread loss of ice cover days (Wang et al. 2022, Richardson et al. 2021\) Frozen lakes and ponds are now at the forefront of anthropogenic climate change. (Rantanen et al., 2022).

In conjunction with the advent of the high resolution satellite constellation Sentinel-2, in recent years, the development of commercial satellite constellations has opened the door for new potential understanding in pan-Arctic monitoring of environmental trends. Planet Labs operates a constellation of over 450 CubeSats, offering between 4 and 8 spectral bands at a 3m spatial resolution with a near-daily revisit rate worldwide. This unprecedented level of spatial and temporal detail has the potential to expand understanding of rapid changes in ice phenology of small lakes (Cooley et al. 2017, Cooley et al. 2019\) However, to date no studies have investigated the utility of PlanetScope for widespread monitoring of small lakes. Thus, there is potential to investigate whether Planet data can be used to expand our understanding of ice phenology dynamics in small lakes, especially as they relate to climate sensitivity.

<img width="776" height="530" alt="Screenshot 2026-01-09 150241" src="https://github.com/user-attachments/assets/ff18aeaf-ac13-4b9e-9684-1ddc71c7ab78" />

**Figure 1:**We outlined 50x50km bounding boxes for each of our three study sites

(a)   Site I. Yukon-Kuskokwim Delta

(b)   Site II. Yukon Flats

(c)   Site III. North Slope

 

We chose these sites to test satellite performance across a range of latitude and sun angle conditions. We limited our analysis to Alaska in order to use the high resolution ALPOD dataset as a static water mask.

 

**3 Methods**

*3.1 Data*

(a)   Lake Masks \- We used the preexisting Alaska Lakes and Ponds Occurence (ALPOD) dataset (Levenson et al. 2024\) as our lake boundary polygons for both satellite analyses. This dataset contains a high resolution static lake mask of all lakes and ponds across Alaska as small as 0.001km². Lakes were ordered by the “id” column, and were uploaded to the DCC for analysis.

(b)   PlanetScope Data \- For processing PlanetScope imagery, we downloaded 4-band and 8-band Surface Reflectance scenes through Planet’s Data API, with quotas fulfilled through a NASA CSDA grant. We filtered imagery using a 50x50km bounding box around each of our study sites across freeze-up seasons from the years 2019-2025. We approximated the start and end dates of each shoulder season with a quick visual inspection on Planet Explorer, giving at least 1 month of leeway on either side to account for potential interannual variation of ice-on dates. All imagery was limited to 0% total image cloud cover due to the inaccuracy of the UDM-2 cloud mask determined in Section 3.1. We uploaded all freezeup imagery to the DCC in the /work/ directory.

(c)   Sentinel-2 Data \- For processing Sentinel-2 imagery, we utilized an 85% image cloud cover threshold with data retrieved from AWS buckets using the STAC API.  Before analysis, each image was clipped to contain only open lake pixels using binary usable data masks created from the Scene Classification Layer (SCL), Copernicus cloud probability percent mask, and the ALPOD lake boundaries mask.We upload all freezeup imagery to the DCC in the /work/ directory.

*3.2 Freezeup Season*

For freeze-up imagery, traditional pixel-based red thresholding methodologies for ice detection are not feasible due to low sun angles and thin ice cover limiting the spectral information contained within image histograms. Last semester, we tested model accuracy for both satellites across both local and global scale models, and found that site-specific models better accounted for Arctic sun angle variations and light conditions, which improved accuracy of ice classification. Thus, for this project, we trained and tuned random forest models for each of our three study sites, using grid search cross validation and hyperparameter tuning to ensure the characteristics of each study site do not confuse classification at other latitudes. 

Once our models were trained using our pre-existing manual labeling workflow, we uploaded them to the DCC alongside our satellite data in order to track the progression of ice formation across our study sites. 

To efficiently process multiple terabytes of satellite imagery across our three study sites, we parallelized computation using the mpi4py library, which distributed the work of ice classification across multiple nodes on the compute cluster.  We requested 25 parallel ranks, where each rank had 16 worker CPUs, and 128 GB of RAM, allocated via SLURM using sbatch. To start, Rank 0 acted as a coordinator to distribute our list of satellite images evenly between each rank.  This built a queue of images for each rank to process into temporary output CSV files immediately upon completion to save progress, and then Rank 0 would combine all outputs at the end into a single CSV file for the run. Because results were written to disk after each image rather than at the end of the run, this allowed for interrupted jobs to resume without significant reprocessing.

To build our freezeup time series, each rank moved through its image quueue using the multiprocessing library to distribute its work across 16 worker CPUs. For each image, the rank’s main process first loads the full satellite image into POSIX shared memory, allowing for all workers to read from the same space without needing to copy the data thousands of times. For Sentinel-2 only, an additional step was taken on all spectral bands to resample to a common 10m grid. The main thread then clips the ALPOD lake polygons shapefile to only queue those lakes falling entirely within the current image’s spatial footprint, and then distributes intersecting lake polygons evenly across the worker pool. 

Each worker then classified the pixels of a single lake at a time.  Rather than operating on the full image array, which would be computationally intensive, each worker first computed the pixel-space bounding box of its assigned polygon and restricted all subsequent operations (mask burning, cloud filtering, and pixel extraction) to that small window. This reduced the per-lake masking cost by several orders of magnitude. For Sentinel-2, cloud and shadow pixels were excluded using the SCL band, filtering out no data, saturated, dark areas, cloud shadow, unclassified, medium or high confidence cloud, and thin cirrus.  The remaining valid SCL values, which carry land cover information such as snow, water, and bare soil, were retained as input features to the random forest classifier.  For Planet, since we only input images containing zero cloud cover, cloud masking was handled on the scene level. Once each worker received its array of pixels from shared memory, it classified each as either “ice” or “water” using the random forest model trained for that particular site.  We summed the total percent ice cover for each observation day by calculating the % ice pixels out of total observed pixels, and exported those values to a CSV, so that with all images processed, we could build time series to understand the error rates and uncertainty associated with each satellite during freeze up monitoring. 

<img width="585" height="488" alt="Screenshot 2026-03-31 131106" src="https://github.com/user-attachments/assets/66f9215a-3e96-4eef-bea3-ee6aab54bef0" />

**Figure 2:** Comparison of breakup & freezeup histograms highlighting the limitations of low sun angle conditions for red thresholding. Note the shorter tails towards high reflectance values in the freeze-up image, as thin ice under low light levels appears spectrally similar to water, especially since it lacks the winter accumulated snow cover which aids in breakup ice detection.

 

The DCC improved our runtime by approximately 200x, speeding up a process which would have taken nearly a month on a single core to around 4 hours. Using 400 logical cores allowed us to extend our analysis over a larger timespan (7 years) to better understand the benefits and drawbacks of using each satellite for ice monitoring.

*3.3 Time Series Filtering & Breakup Date Calculation*

Once time series of ice cover were calculated from icy pixel percentage, freeze-up dates were estimated from the output CSVs using a 75% ice cover threshold, linearly interpolated from the two closest surrounding observation dates. We also filtered for undetected cloud pixels which slipped through Planet’s UDM-2 mask using sanity filters applied across all time series points (e.g. removing rapid changes in ice cover caused by errant cloud or cloud shadow outside of our estimated breakup window.) Uncertainty was calculated based on mean distance between the interpolated breakup date and two closest actual observations.

**4 Results & Discussion**

*3.1 Freezeup Season Uncertainty*

We found that Planet suffered at high latitudes due to its inferior cloud masking algorithm reducing the number of available images from their purported “near-daily” observation rate by an order of magnitude. This on top of Sentinel’s orbital convergence towards the poles meant that Sentinel-2 exhibited a more precise uncertainty range across the majority of years. This effect was slightly reduced in the Yukon Delta, as this site is in the south of Alaska and did not have the added revisit benefit from the more polar latitudes, leading to Planet moderately outperforming Planet in 4 of 7 years by a lower error margin of 0.2 days.

<img width="729" height="533" alt="Screenshot 2026-03-25 143310" src="https://github.com/user-attachments/assets/e13aaa1d-5dd2-4473-84b6-025a571f4ced" />

**Figure 3:** Average error days for each of our three study sites across all 7 years of our data.  Sentinel-2 outperformed Planet’s observation frequency in the majority of years, but that advantage was lost in the lower latitude Yukon Kuskokwim Delta during freezeup season.

*3.2 Time Series Results*


From our time series themselves, we find that Sentinel-2 consistently showed more reliable revisit rates thanks to its superior cloud masking capabilities, making it unquestionably the more reliable option for monitoring ice phenology at high latitudes.  Planet’s large constellation of satellites could offer some temporal benefits for ice monitoring at lower latitudes, as evidenced by the slightly improved rates in the Yukon Delta, although across all sites its cloud mask suffers in all seasons, which produces noisy time series requiring extensive manual filtering.  Sentinel-2 was consistently able to produce accurate freezeup date estimates without excessive manual filtering, which we validated by hand using OpenCV-2 to visually inspect ice percentages and breakup dates.

<img width="1160" height="921" alt="4ee9a089-07f6-4563-8411-0225e5ca4b03" src="https://github.com/user-attachments/assets/cde69a9a-503a-4939-aa55-5c2902d9f187" />

**Figure 4:** Full year time series, including freezeup data results (Sep-Nov) calculated on the DCC, and breakup data from previous workflows developed in 2025\.  For brevity, this figure just shows three years sampled from the Yukon Flats, but time series were calculated across all 7 years for all 3 study sites. Data gaps during the summer are shown with crosshatching and dotted lines, as ice is assumed to be absent in warmer summer months.  The solid curves show the estimated breakup time series for all lakes across this 50x50km site in the Yukon Flats, where maximum Y values (100%) represents the date when all lakes within the study site are free of ice cover, and 0% represents the date when all lakes are frozen over.  The number of lakes observed each day is represented for each satellite in the center bar chart– note that Planet’s observation frequency over the site is much more ragged due to its smaller swath width and uncertainty with the cloud mask, so that only chunks of our full study site are observed at a time.  This irregularity over larger areas leads to higher uncertainty windows (shaded) particularly in 2022, where almost the entire freezeup season failed to produce observations. Sentinel (red) on the other hand shows very consistent, reliable revisit rates, with nearly the entire site observed on a near-daily basis thanks to its superior cloud masking allowing for cloudier scenes to still be partially usable.

**5 Conclusions**

We used the Duke Compute Cluster to accelerate the calculation of ice freezeup dates across three study sites in the North, South, and center of Alaska.  We found that Sentinel-2 significantly outperformed Planet in the majority of years, with reduced benefit at lower latitudes, despite Planet’s purportedly higher revisit rates in marketing materials.  Sentinel-2’s SCL cloud mask was found to be sgnificantly more reliable than Planet’s u-Net powered UDM-2, leading to consistently more usable imagery for the Sentinel mission across all sites, especially during freezeup.  Sentinel-2 remains the superior mission for ice phenology monitoring in the Arctic today, but if in the future Planet improves its UDM-2 cloud mask with the addition of SWIR bands and additional arctic training data, it could drastically improve its usable data revisit rate.

**6 Code Availability**

Codebase accessible on our Github.
