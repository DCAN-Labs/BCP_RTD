# Proccessed Derivatives from the BCP imaging data

- BIDS format
- s3 bucket structure (Release date?)

## DCAN-infant

## BIBSNet

## Nibabies
# BCP Imaging Processing Tools

BCP imaging data were processed using a two-stage workflow. First, MRI data were preprocessed using **NiBabies**, a neuroimaging preprocessing pipeline for infant structural and functional MRI data. After NiBabies preprocessing, functional MRI derivatives were post-processed using **XCP-D** to prepare the data for connectivity analysis.

This page summarizes the BCP-specific imaging workflow, including script locations, Singularity container images, NiBabies surface reconstruction settings, input and output locations, major command-line settings, expected outputs, and QC-related processing decisions.

## Processing Workflow

The BCP processing workflow includes two main stages:

1. **NiBabies preprocessing**
   - Runs anatomical and functional MRI preprocessing.
   - Uses the appropriate NiBabies surface reconstruction method based on session age: `mcribs`, `infantfs`, or `freesurfer`.
   - Provides BIBSNet segmentation derivatives as additional input for `mcribs` and `infantfs` runs.
   - Produces preprocessed imaging derivatives for downstream analyses.

2. **XCP-D post-processing**
   - Uses NiBabies derivatives as input.
   - Runs post-processing for functional MRI data.
   - Runs XCP-D in HBCD mode.
   - Produces outputs for downstream connectivity analysis.

Current derivative output locations:

```text
NiBabies outputs:
s3://bcp-nibabies/

XCP-D outputs:
s3://bcp-xcpd/
```

---

## NiBabies

### Overview

BCP imaging data were preprocessed using **NiBabies**. The BCP NiBabies workflow used BIDS-formatted subject/session inputs and generated preprocessed anatomical and functional MRI derivatives.

The NiBabies processing scripts are located here:

```text
/projects/standard/faird/shared/projects/bcp_processing_work/nibabies_work/
```

The current NiBabies derivatives are stored here:

```text
s3://bcp-nibabies/
```

### Container Image

NiBabies processing was run using the following Singularity image:

```text
/home/faird/shared/code/external/pipelines/nibabies/nibabies_25.0.1.sif
```

### BCP Surface Reconstruction Configuration

The BCP NiBabies workflow selected the surface reconstruction method based on session age.

| Session age | NiBabies surface reconstruction method | BIBSNet derivatives used |
|---|---|---|
| 3 months or younger | `mcribs` | Yes |
| Older than 3 months and up to 24 months | `infantfs` | Yes |
| Older than 24 months | `freesurfer` | No |

The selected surface reconstruction method was assigned in the script using the `recon` variable and passed to NiBabies with:

```bash
--surface-recon-method ${recon}
```

The three possible values were:

```bash
recon=mcribs
recon=infantfs
recon=freesurfer
```

For NiBabies runs using `mcribs` or `infantfs`, existing BIBSNet segmentation outputs were provided as additional input:

```bash
--derivatives /derivatives
```

In the BCP script, `/derivatives` refers to the mounted BIBSNet output directory inside the container.

### Input and Output Bindings

The NiBabies command used Singularity bind mounts to map project directories into the container.

Common bindings:

```bash
-B ${data_dir}/bids_dir/sub-${subj_id}_ses-${ses_id}:/bids_dir
-B ${data_dir}/processed/nibabies/sub-${subj_id}_ses-${ses_id}:/output_dir
-B ${data_dir}/work_dir/nibabies/sub-${subj_id}_ses-${ses_id}:/wd
-B ${run_dir}/license.txt:/license.txt
```

For NiBabies runs using `mcribs` or `infantfs`, BIBSNet derivatives were also mounted:

```bash
-B ${data_dir}/processed/bibsnet/sub-${subj_id}_ses-${ses_id}:/derivatives
```

This means the BCP NiBabies workflow used the following container paths:

```text
Input BIDS directory:
/bids_dir

NiBabies output directory:
/output_dir

NiBabies work directory:
/wd

FreeSurfer license:
/license.txt

BIBSNet derivatives, when used:
/derivatives
```

### Command-Line Template

The general NiBabies command template was:

```bash
singularity run --cleanenv \
  -B ${data_dir}/bids_dir/sub-${subj_id}_ses-${ses_id}:/bids_dir \
  -B ${data_dir}/processed/nibabies/sub-${subj_id}_ses-${ses_id}:/output_dir \
  -B ${data_dir}/work_dir/nibabies/sub-${subj_id}_ses-${ses_id}:/wd \
  -B ${run_dir}/license.txt:/license.txt \
  /home/faird/shared/code/external/pipelines/nibabies/nibabies_25.0.1.sif \
  /bids_dir /output_dir participant \
  --output-spaces MNI152NLin6Asym:res-2 \
  --fs-license-file /license.txt \
  --omp-nthreads <OMP_THREADS> \
  --nprocs 8 \
  --cifti-output 91k \
  -vv \
  --project-goodvoxels \
  --fd-radius 45 \
  --surface-recon-method ${recon} \
  --multi-step-reg \
  --norm-csf \
  -w /wd
```

For NiBabies runs using `mcribs` or `infantfs`, the command also included:

```bash
--derivatives /derivatives
```

### NiBabies Processing Settings

Major NiBabies settings used in the BCP workflow included:

| Setting | Value | Description |
|---|---|---|
| `--output-spaces` | `MNI152NLin6Asym:res-2` | Requests anatomical and functional outputs in MNI152NLin6Asym space with `res-2` output sampling |
| `--fs-license-file` | `/license.txt` | Points NiBabies to the mounted FreeSurfer license file |
| `--omp-nthreads` | `8` or `3` | Sets the maximum number of OpenMP threads per process; in the BCP script, `8` was used for `mcribs` and `freesurfer`, and `3` was used for `infantfs` |
| `--nprocs` | `8` | Sets the maximum number of threads across all processes |
| `--cifti-output` | `91k` | Outputs preprocessed BOLD data as CIFTI dense time series at 91k grayordinates |
| `-vv` | enabled | Runs NiBabies with increased logging verbosity |
| `--project-goodvoxels` | enabled | Excludes locally high-variation voxels during surface resampling |
| `--fd-radius` | `45` | Sets the head radius, in millimeters, used for framewise displacement calculation |
| `--derivatives` | `/derivatives` | Provides precomputed BIBSNet segmentation derivatives for runs using `mcribs` or `infantfs`; omitted for `freesurfer` runs in the BCP script |
| `--surface-recon-method` | `${recon}` | Selects the NiBabies surface reconstruction method: `mcribs`, `infantfs`, or `freesurfer` |
| `--multi-step-reg` | enabled | Uses multi-step registration for MNI152NLin6Asym output |
| `--norm-csf` | enabled | Replaces low-intensity voxels in the CSF mask with the average value |
| `-w` | `/wd` | Sets the NiBabies working directory for intermediate files |

### BCP Anatomical File Selection

For BCP sessions with multiple anatomical runs, the best available anatomical image was selected before final curated outputs were generated. When a subject/session had multiple T1w or T2w anatomical runs, the best anatomical run was selected based on manual quality-control measures.

### Expected NiBabies Outputs

Expected NiBabies outputs include:

```text
- Visual quality-assessment HTML reports
- Preprocessed anatomical derivatives
- Preprocessed functional MRI derivatives
- CIFTI dense time series outputs, when --cifti-output 91k is enabled
- Surface reconstruction outputs, when surface reconstruction is run
- Output metadata and log-related files
```

---


## XCP-D

### Overview

After NiBabies preprocessing, BCP functional MRI data were post-processed using **XCP-D**. XCP-D was used to prepare the preprocessed functional MRI derivatives for connectivity analysis.

The XCP-D processing scripts are located here:

```text
/projects/standard/faird/shared/projects/bcp_processing_work/xcpd_work/
```

The current XCP-D derivatives are stored here:

```text
s3://bcp-xcpd/
```

### Container Image

XCP-D processing was run using the following Singularity image:

```text
/home/faird/shared/code/external/pipelines/ABCD-XCP/xcp_d_0.10.7.sif
```

### Inputs and Outputs

XCP-D used NiBabies derivatives as input. In HBCD mode, XCP-D expects NiBabies derivatives as the preprocessing input.

The NiBabies output directory was mounted as read-only input:

```bash
-B ${data_dir}/processed/nibabies/sub-${subj_id}_ses-${ses_id}:/data:ro
```

The XCP-D work directory was mounted here:

```bash
-B ${data_dir}/work/xcpd/sub-${subj_id}_ses-${ses_id}:/wd
```

The XCP-D output directory was mounted here:

```bash
-B ${data_dir}/processed/xcpd/sub-${subj_id}_ses-${ses_id}:/out
```

The FreeSurfer license was mounted here:

```bash
-B /home/faird/shared/code/external/utilities/freesurfer_license/license.txt:/opt/freesurfer/license.txt
```

### Motion Filter Settings

The BCP XCP-D workflow selected motion filter band-stop values based on session age.

| Session age | `--band-stop-min` | `--band-stop-max` |
|---|---:|---:|
| Younger than 12 months | `30` | `60` |
| 12 to 24 months | `25` | `50` |
| Older than 24 months | `20` | `35` |

These values were used with the notch motion filter:

```bash
--motion-filter-type notch
--band-stop-min ${bs_min}
--band-stop-max ${bs_max}
```

### Command-Line Template

The BCP XCP-D command template was:

```bash
env -i ${singularity} run --cleanenv \
  -B /home/faird/shared/code/external/utilities/freesurfer_license/license.txt:/opt/freesurfer/license.txt \
  -B ${data_dir}/processed/nibabies/sub-${subj_id}_ses-${ses_id}:/data:ro \
  -B ${data_dir}/work/xcpd/sub-${subj_id}_ses-${ses_id}:/wd \
  -B ${data_dir}/processed/xcpd/sub-${subj_id}_ses-${ses_id}:/out \
  /home/faird/shared/code/external/pipelines/ABCD-XCP/xcp_d_0.10.7.sif \
  /data /out participant \
  -r auto \
  -f 0.3 \
  --participant-label ${subj_id} \
  --mode hbcd \
  --motion-filter-type notch \
  --band-stop-min ${bs_min} \
  --band-stop-max ${bs_max} \
  --min-time 104 \
  -w /wd
```

### XCP-D Processing Settings

Major XCP-D settings used in the BCP workflow included:

| Setting | Value | Description |
|---|---|---|
| `-r` / `--head-radius` | `auto` | Estimates the head radius from the preprocessed brain mask for framewise displacement calculation |
| `-f` / `--fd-thresh` | `0.3` | Sets the framewise displacement threshold for censoring high-motion volumes |
| `--participant-label` | `${subj_id}` | Runs XCP-D for the selected subject |
| `--mode` | `hbcd` | Runs XCP-D in HBCD mode, which applies mode-specific processing defaults |
| `--motion-filter-type` | `notch` | Applies a notch filter to motion regressors |
| `--band-stop-min` | `${bs_min}` | Sets the lower frequency for the notch motion filter; value depends on session age |
| `--band-stop-max` | `${bs_max}` | Sets the upper frequency for the notch motion filter; value depends on session age |
| `--min-time` | `104` | Requires at least 104 seconds of usable data after high-motion volumes are removed |
| `-w` / `--work-dir` | `/wd` | Sets the XCP-D working directory for intermediate files |

### Expected XCP-D Outputs

Expected XCP-D outputs include:

```text
- Denoised BOLD derivatives
- Parcellated time series
- Functional connectivity matrices
- Motion and censoring outputs
- Quality-control files and summary reports
- Anatomical derivatives passed through or transformed from the preprocessing outputs
- Processing metadata and log-related files
```

