# 6d-pose-estimation-rob26
A repo linking together the various methods explored for a Robotics project having as subject 6d pose estimation.
Use the following command to init all submodules:
```
git clone --recursive
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

## Inference
Here are the instructions to reproduce the inference steps using both sub-packages of HappyPose

### CosyPose (HappyPose)
```
# download ycbv model weights
python -m happypose.toolbox.utils.download --cosypose_models \
          detector-bop-ycbv-pbr--970850 \
          coarse-bop-ycbv-pbr--724183 \
          refiner-bop-ycbv-pbr--604090

# move linemod example into dataset folder
# bop toolkit info used to build example: https://github.com/thodan/bop_toolkit/blob/main/docs/bop_datasets_format.md
cp -r examples/eggs dataset/examples/

# run example
python -m happypose.pose_estimators.cosypose.cosypose.scripts.run_inference_on_example eggs --dataset ycbv --run-inference --run-detections --vis-detections --vis-poses
# the results will be in dataset/examples/eggs/visualizations
```

### MegaPose (HappyPose)
```
# download all megapose model weights
python -m happypose.toolbox.utils.download --megapose_models

# download pretrained ycbv detection model weights
python -m happypose.toolbox.utils.download --cosypose_models detector-bop-ycbv-pbr--970850

# move linemod example into dataset folder
# bop toolkit info used to build example: https://github.com/thodan/bop_toolkit/blob/main/docs/bop_datasets_format.md
cp -r examples/eggs dataset/examples/

# run example
python -m happypose.pose_estimators.megapose.scripts.run_inference_on_example eggs --run-inference --run-detections --vis-detections --vis-poses
# the results will be in dataset/examples/eggs/visualizations
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
