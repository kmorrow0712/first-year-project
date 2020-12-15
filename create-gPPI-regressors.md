# Creating gPPI regressors
---
## Relevant details

### Dataset
MAX (N = 5)

### Seed region(s)
#### left Crus II (cerebellum)
`SUIT_l-CrusII_YeoNetwork6_intersect_gm_2mm.nii.gz` <br>
**transformations**
1. Crus II multipled with [Buckner et al's](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3214121/) cognitive control network.
2.    Intersection then multiplied with a [SUIT](http://www.diedrichsenlab.org/imaging/suit.htm) gray matter mask.


### Programs utilized
AFNI

### Goal
Practice generating general psychophysiological interaction (gPPI) terms to explore functional connectivity between seed regions and all other voxels of the brain and how this relationship relates to task. </br>

### Info
 Scripts and data files are described below. Each step is documented with output images or examples when applicable. Very small group of subjects used with a simple analysis in mind (for now...). <br>
 _All scripts shown are shell script and show typical shell shortcuts e.g., `"$subj"`_

---

### 1. Extract baseline from first level subject data

First, need to extract the baseline which includes drifts, motion, and motion derivatives as output from `3dDeconvolve` using `3dSynthesize`. Baseline data is extracted from a basic first level model in the sense that it just consists of blocks, but fully accounts for the task. <br>

**Purpose:** Extact variance associated with anything but the task and then subtract this baseline from functional data to have task-based variance left.

    3dSynthesize -cbucket ./MAX"$subj"_Main_block_MR_REML_beta_shockcensored_I.nii.gz \
    -matrix ./"MAX"$subj"_Main_block_MR_uncensored_I.x1D" \
    -select baseline -overwrite \
    -prefix "$output"/MAX/"$subj"_EP_Main_TR_MNI_2mm_I_denoised_baselineModel.nii.gz

### 2. Subtract baseline from preprocssed functional data
With the baseline model extracted, we now need to subtract it from preprocessed functional data using `3dcalc`

    3dcalc -a $proj_path"/dataset/preproc/MAX"$subj"/func2/ICA_AROMA/MAX"$subj"_EP_Main_TR_MNI_2mm_SI_denoised.nii.gz" \
    -b ./"MAX"$subj_EP_Main_TR_MNI_2mm_I_denoised_baselineModel.nii.gz -prefix "$output"/MAX"$subj"_EP_Main_TR_MNI_2mm_I_denoised_NoBaseline.nii.gz \
    -expr 'a-b' \
    -overwrite \

### 3. Extract time series from seed region
    3dROIstats -quiet -overwrite
    -mask /data/bswift-1/kmorrow/02-ROI_masks/SUIT_cerebellum_2mm/SUIT_l-CrusII_YeoNetwork6_intersect_gm_2mm.nii.gz \ $output"MAX"$subj"_EP_Main_TR_MNI_2mm_I_denoised_NoBaseline.nii.gz" > \
    $output"MAX"$subj"_l-CrusII_seed_NoBaseline_avg.1D"

<fig>
  <img src="/assets/images/gPPI-seedTimeseries-example.png" width="400" height="300" />
  <figcaption>Example of extracted time series for left Crus II ROI from MAX101</figcaption>
</fig>



### 4. Create boxcar functions and upsample each regressor

For the sake of this simple analysis, I've used a simple but *full* model. This means that variance is accounted for from all events in the paradigm, but are simple blocks.

Codes are as follows:<br>
`RNT` - neutral threat block shocks <br>
`RNS` - neutral block with touch<br>
`RPT` - positive threat blocks with shocks <br>
`RPS` - positive safe blocks with touch <br>
`FNT` - neutral threat blocks without touch <br>
`FNS` - neutral safe without shock <br>
`FPT` - positive threat without shock <br>
`FPS` - positive safe without touch <br>

Anxiety ratings after each block: <br>
`RNT_rate`
`RNS_rate`
`RPT_rate`
`RPS_rate`
`FNT_rate`
`FNS_rate`
`FPT_rate`
`RPS_rate`


We will downsample TRs from 1.25s to 0.05s (upsample rate of 25)

`stim_dur` is `16.25` which is the length of a block (response ratings are 2s and computed in a second script)

`run_len` is `425`, which is the number of volumes per run times the original TR `340 * 1.25`

_Reminder that these scripts are generally in a loop which can be seen in full in `/scripts` directory_

    timing_tool.py
    -timing $proj_path/stim_times/"$regressors[$regressor_index]".txt \
    -timing_to_1D /data/bswift-1/kmorrow/03-gPPI_testing/dataset/regressors/MAX"$i_subj"/gPPI/"$regressors[$regressor_index]_upsample.txt" \
     -tr 0.05 \
     -stim_dur 16.25 \
     -min_frac 0.3 -run_len 425 425 425 425 425 425

<fig>
  <img src="/assets/images/gPPI-upsampledReg-example.png" width="400" height="300" />
  <figcaption>Example of regressor upsampled to 0.05 TR. False neutral threat (FNT)</figcaption>
</fig>


### 5. Get seed region data per run
Shell script loops through each run based on start and ending index.

`start_index1 = run index * 340`
`end_index1 = (run index * 340) + (340 - 1) `

    1dcat /data/bswift-1/kmorrow/03-gPPI_testing/output/"MAX"$i_subj"_l-CrusII_seed_NoBaseline_avg.1D'($start_index1..$end_index1}[0]'" > "$seed_names[$seed_index]"_Seed_perRun.1D

### 6. Upsample seed data
Upsample to the same rate as the regressors in step #4 (0.05)

    1dUpsample "$UpSampRate" "$seed_names[$seed_index]"_Seed_perRun.1D > "$seed_names[$seed_index]"_Seed_perRun_upsample.1D

<fig>
  <img src="/assets/images/gPPI-seedPerRunUpsample-example.png" width="400" height="300" />
<figcaption>Seed data upsampled to 0.05s TR</figcaption>
</fig>

### 7a. Create gamma function
Resembles the hemodynamic response function at the new upsampled TR (0.05s)

    Waver -dt 0.05 -peak 1 -GAM -inline 1@1 > GammaHR_TR05.1D
<fig>
  <img src="/assets/images/gPPI-gammaFunction.png" width="400" height="300" />
  <figcaption>Gamma function to deconvolve seed timeseries</figcaption>
</fig>

### 7b. Deconvolve seed time seed timeseries with gamma function


    3dTfitter
    -RHS "$seed_names[$seed_index]"_Seed_perRun_upsample.1D \
    -FALTUNG /data/bswift-1/kmorrow/03-gPPI_testing/GammaHR_TR05.1D \ "$seed_names[$seed_index]"_Seed_Neur_perRun_upsample.1D 012 -2

    1dtranspose "$seed_names[$seed_index]"_Seed_Neur_perRun_upsample.1D > "$seed_names[$seed_index]"_Seed_Neur_perRun_upsampleT.1D

### An aside: Does mean-centering simple effects before creating interaction term impact collinearity?
There are mixed opinions on whether mean centering helps reduce collinearity between the interaction term and simple effects. To test this, I've created two sets of interaction terms, one set is mean center and the other not. I will calculate variance inflation factors between A, B, and A*B for each set and compare.

VIFs - mean-centered


VIFs - not mean-centered



### 8. De-mean neuronal timeseries
    1d_tool.py -infile "$seed_names[$seed_index]"_Seed_Neur_perRun_upsampleT.1D
    -demean -write \ "$seed_names[$seed_index]"_Seed_Neur_perRun_upsampleT_demean.1D

  <img src="/assets/images/gPPI-seedTimeseriesUpsampled_demeaned.png" width="400" height="300" />


### 9. Create interaction terms for each condition

    1dcat $out_path/dataset/regressors/MAX"$i_subj"/gPPI/""$regressors[$regressor_index]"_upsample.txt'{$start_index2..$end_index2}'" > tmp.1D

    1d_tool.py -infile tmp.1D -demean -write tmp_demean.1D

    1deval -a "$seed_names[$seed_index]"_Seed_Neur_perRun_upsampleT_demean.1D \
    -b tmp_demean.1D \
    -expr 'a*b’ > "$seed_names[$seed_index]"_Seed_"$regressors[$regressor_index]"_Neur_perRun_upsampleT.1D

### 10. Convolve newly created terms with canonical HRF

    waver -GAM \
    -peak 1 \
    -TR $sub_TR \
    -input "$seed_names[$seed_index]"_Seed_"$regressors[$regressor_index]"_Neur_perRun_upsampleT.1D -numout $noOfVol_perRun_upsample >  "$seed_names[$seed_index]"_Seed_"$regressors[$regressor_index]"_Conv_perRun_upsampleT.1D

### 11. Downsample to original TR (1.25s)

    1dcat "$seed_names[$seed_index]"_Seed_"$regressors[$regressor_index]"_Conv_perRun_upsampleT.1D'{0..$(25)}' >> "$seed_names[$seed_index]"_Seed_"$regressors[$regressor_index]"_Conv_allRuns.1D``

<fig>
  <img src="/assets/images/gPPI-interactionTerm_FNT-example.png" width="400" height="300" />
<figcaption>Example of interaction term for False neutral threat (FNT) condition</figcaption>
</fig>


### These should obviously be de-spiked or censored in some way.
