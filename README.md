# 6d-pose-estimation-rob26
A repo linking together the various methods explored for a Robotics project having as subject 6d pose estimation.
Use the following command to init all submodules:
```
git clone --recursive *github link*
```

## Observations
Following are some notes on working with the corresponding methods

### HappyPose
The initial repo did not work for my setup since I needed a way to abstract from my machine cuda version, the best solution seemed to dockerize the software.
During the dockerization process some issues popped up, in particular:
- Needed to fix some dependency specifications in pyproject.yml (a753b69)
- Fix bug where rotation matrix used in megapose was not loaded in the same device as the input rendered images (75abbbc)
- Need to specify weights_only=False in new versions of torch when using torch.load() for megapose to work as expected (9311f52)
- Some parts of the mirrors for the BOP dataset used in the download script were no longer available, in these cases the instructions were updated to use the huggingface cmd utility to download the dataset (7cdebad)
- There was no way to run the methods on linemod without adding configs for which detectors to use (85e568a)
- Some issues in the config of folders containing the lm dataset, fixed by adding explicit parameters for it (ea68991)

### PVNet
The main issues with this method are related to the fact that it's old and no longer maintained. 
Although there already was a docker configuration setup to allow for easy usage, the image on which it relied (old version of nvidia/cuda) was no longer available due to deprecation.
The base image has therefore been changed to a relatively similar pytorch one.
After many trials a mixture of pip and apt has been used to replicate the original environment with success.

## Inference

### HappyPose
Here are the instructions to reproduce the inference steps using both sub-packages of HappyPose.
To run the methods on single images, some example folders have been setup in the examples/ dir.

Note:
- the results are saved in the target example's folder
- the resulting output is formatted as a list of couples consisting of a quaternion and a translation vector
To convert from quaternions to rotation matrix for metric calculations, the function `compute_rotation_matrix_from_quaternions` in `happypose/toolbox/lib3d/rotations.py` can be used.
There doesn't seem to be any direct way to compute ADD/ADD-S metrics from examples currently, for that keep only the desidered test images in the dataset folder and the corresponding entries inside test_targets_bop19.json

#### CosyPose
```
# download pretrained linemod detector weights
python -m happypose.toolbox.utils.download --cosypose_models\
    detector-bop-lmo-pbr--517542 \
    coarse-bop-lmo-pbr--707448 \
    refiner-bop-lmo-pbr--325214

# download ycbv model weights
python -m happypose.toolbox.utils.download --cosypose_models \
          detector-bop-ycbv-pbr--970850 \
          coarse-bop-ycbv-pbr--724183 \
          refiner-bop-ycbv-pbr--604090

# move linemod example into dataset folder
# bop toolkit info used to build example: https://github.com/thodan/bop_toolkit/blob/main/docs/bop_datasets_format.md
cp -r examples/eggs dataset/examples/

# run example
python -m happypose.pose_estimators.cosypose.cosypose.scripts.run_inference_on_example eggs --dataset lm --run-inference --run-detections --vis-detections --vis-poses
# the results will be in dataset/examples/eggs/visualizations

# run method on lm/lmo dataset, results are in dataset/results/lm-debug
python -m happypose.pose_estimators.cosypose.cosypose.scripts.run_full_cosypose_eval_new detector_run_id=bop_pbr coarse_run_id=coarse-bop-ycbv-pbr--724183 refiner_run_id=refiner-bop-ycbv-pbr--604090 ds_names=["lm.bop19"] result_id=lm-debug detection_coarse_types=["detector"] inference.renderer=panda3d inference.n_workers=4
```

#### MegaPose
```

# download all megapose model weights and move them to the correct folder
python -m happypose.toolbox.utils.download --megapose_models
cp ./dataset/megapose-models/* ./dataset/experiments/

# download pretrained ycbv detector weights
python -m happypose.toolbox.utils.download --cosypose_models detector-bop-ycbv-pbr--970850

# download pretrained linemod detector weights
python -m happypose.toolbox.utils.download --cosypose_models detector-bop-lmo-pbr--517542

# move linemod example into dataset folder
# bop toolkit info used to build example: https://github.com/thodan/bop_toolkit/blob/main/docs/bop_datasets_format.md
cp -r examples/eggs dataset/examples/

# run example
python -m happypose.pose_estimators.megapose.scripts.run_inference_on_example eggs --run-inference --run-detections --vis-detections --vis-poses --detector detector-bop-lmo-pbr--517542
# the results will be in dataset/examples/eggs/visualizations

# run method on lm/lmo dataset, results are in dataset/results/lm-debug
python -m happypose.pose_estimators.megapose.scripts.run_full_megapose_eval detector_run_id=bop_pbr coarse_run_id=coarse-rgb-906902141 refiner_run_id=refiner-rgb-653307694 ds_names=[lm.bop19] result_id=detector_1posehyp detection_coarse_types=[["detector","SO3_grid"]] inference.n_pose_hypotheses=1 skip_inference=false run_bop_eval=true

```

## PVNet
The use of docker is recommended for this method, follow the README inside docker/.
Follow the README instructions in the clean-pvnet repo, starting from the *Testing* section, to set it up and run the commands inside the docker/environment.

```bash
# Test if it's working 
python run.py --type evaluate --cfg_file configs/linemod.yaml model cat cls_type cat
```

## Datasets
Some commands to download the parts of the datasets needed for evaluation.

### Linemod
```
# download dataset
hf download bop-benchmark/lm \
            --local-dir ./dataset/bop_datasets/lm \
            --repo-type=dataset \
            lm_base.zip lm_models.zip lm_test_all.zip
mv ./dataset/bop_datasets/lm ./dataset/bop_datasets/lmo
cd ./dataset/bop_datasets/lm
7z e lm_base.zip
7z x lm_models.zip 
7z x lm_test_all.zip
```

## Evaluation (bop-toolkit)
After running the chosen 6d estimation method, use the utilities provided by bop-toolkit (command may vary depending on how it's been installed) to compute the ADD and ADD-S (adi) metrics.
Requires downloading the dataset which the results csv refers to.
```
python eval_calc_errors.py --error_type add --result_filenames relative/path/to/results/csv
python eval_calc_errors.py --error_type adi --result_filenames relative/path/to/results/csv
```

After computing the errors, use this to compute the scores:
```
python /happypose/.venv/bin/eval_calc_scores.py --error_dir_paths comma/separated/list/of/relative/paths
```
