# Inference in the wild

**Update:** we have added support for Detectron2.

In this short tutorial, we show how to run our model on arbitrary videos and visualize the predictions. Note that this feature is only provided for experimentation/research purposes and presents some limitations, as this repository is meant to provide a reference implementation of the approach described in the paper (not production-ready code for inference in the wild).

Our script assumes that a video depicts *exactly* one person. In case of multiple people visible at once, the script will select the person corresponding to the bounding box with the highest confidence, which may cause glitches.

The instructions below show how to use Detectron to infer 2D keypoints from videos, convert them to a custom dataset for our code, and infer 3D poses. For now, we do not have instructions for CPN. In the last section of this tutorial, we also provide some tips.

## Step 1: setup
The inference script requires `ffmpeg`, which you can easily install via conda, pip, or manually.

Download the [pretrained model](https://dl.fbaipublicfiles.com/video-pose-3d/pretrained_h36m_detectron_coco.bin) for generating 3D predictions. This model is different than the pretrained ones listed in the main README, as it expects input keypoints in COCO format (generated by the pretrained Detectron model) and outputs 3D joint positions in Human3.6M format. Put this model in the `checkpoint` directory of this repo.

**Note:** if you had downloaded `d-pt-243.bin`, you should download the new pretrained model using the link above. `d-pt-243.bin` takes the keypoint probabilities as input (in addition to the x, y coordinates), which causes problems on videos with a different resolution than that of Human3.6M. The new model is only trained on 2D coordinates and works with any resolution/aspect ratio.

## Step 2 (optional): video preprocessing
Since the script expects a single-person scenario, you may want to extract a portion of your video. This is very easy to do with ffmpeg, e.g.
```
ffmpeg -i input.mp4 -ss 1:00 -to 1:30 -c copy output.mp4
```
extracts a clip from minute 1:00 to minute 1:30 of `input.mp4`, and exports it to `output.mp4`.

Optionally, you can also adapt the frame rate of the video. Most videos have a frame rate of about 25 FPS, but our Human3.6M model was trained on 50-FPS videos. Since our model is robust to alterations in speed, this step is not very important and can be skipped, but if you want the best possible results you can use ffmpeg again for this task:
```
ffmpeg -i input.mp4 -filter "minterpolate='fps=50'" -crf 0 output.mp4
```

## Step 3: inferring 2D keypoints with Detectron

### Using Detectron2 (new)
Set up [Detectron2](https://github.com/facebookresearch/detectron2) and use the script  `inference/infer_video_d2.py` (no need to copy this, as it directly uses the Detectron2 API). This script provides a convenient interface to generate 2D keypoint predictions from videos without manually extracting individual frames.

To infer keypoints from all the mp4 videos in `input_directory`, run
```
cd inference
python infer_video_d2.py \
    --cfg COCO-Keypoints/keypoint_rcnn_R_101_FPN_3x.yaml \
    --output-dir output_directory \
    --image-ext mp4 \
    input_directory
```
The results will be exported to `output_directory` as custom NumPy archives (`.npz` files). You can change the video extension in `--image-ext` (ffmpeg supports a wide range of formats).

**Note:** although the architecture is the same (ResNet-101), the weights used by the Detectron2 model are not the same as those used by Detectron1. Since our pretrained model was trained on Detectron1 poses, the result might be slightly different (but it should still be pretty close).

### Using Detectron1 (old instructions)
Set up [Detectron](https://github.com/facebookresearch/Detectron) and copy the script `inference/infer_video.py` from this repo to the `tools` directory of the Detectron repo. This script provides a convenient interface to generate 2D keypoint predictions from videos without manually extracting individual frames.

Our Detectron script `infer_video.py` is a simple adaptation of `infer_simple.py` (which works on images) and has a similar command-line syntax.

To infer keypoints from all the mp4 videos in `input_directory`, run
```
python tools/infer_video.py \
    --cfg configs/12_2017_baselines/e2e_keypoint_rcnn_R-101-FPN_s1x.yaml \
    --output-dir output_directory \
    --image-ext mp4 \
	--wts https://dl.fbaipublicfiles.com/detectron/37698009/12_2017_baselines/e2e_keypoint_rcnn_R-101-FPN_s1x.yaml.08_45_57.YkrJgP6O/output/train/keypoints_coco_2014_train:keypoints_coco_2014_valminusminival/generalized_rcnn/model_final.pkl \
    input_directory
```
The results will be exported to `output_directory` as custom NumPy archives (`.npz` files). You can change the video extension in `--image-ext` (ffmpeg supports a wide range of formats).

## Step 4: creating a custom dataset
Run our dataset preprocessing script from the `data` directory:
```
python prepare_data_2d_custom.py -i /path/to/detections/output_directory -o myvideos
```
This creates a custom dataset named `myvideos` (which contains all the videos in `output_directory`, each of which is mapped to a different subject) and saved to `data_2d_custom_myvideos.npz`. You are free to specify any name for the dataset.

**Note:** as mentioned, the script will take the bounding box with the highest probability for each frame. If a particular frame has no bounding boxes, it is assumed to be a missed detection and the keypoints will be interpolated from neighboring frames.

## Step 5: rendering a custom video and exporting coordinates
You can finally use the visualization feature to render a video of the 3D joint predictions. You must specify the `custom` dataset (`-d custom`), the input keypoints as exported in the previous step (`-k myvideos`), the correct architecture/checkpoint, and the action `custom` (`--viz-action custom`). The subject is the file name of the input video, and the camera is always 0.
```
python run.py -d custom -k myvideos -arc 3,3,3,3,3 -c checkpoint --evaluate pretrained_h36m_detectron_coco.bin --render --viz-subject input_video.mp4 --viz-action custom --viz-camera 0 --viz-video /path/to/input_video.mp4 --viz-output output.mp4 --viz-size 6
```

You can also export the 3D joint positions (in camera space) to a NumPy archive. To this end, replace `--viz-output` with `--viz-export` and specify the file name.

## Limitations and tips
- The model was trained on Human3.6M cameras (which are relatively undistorted), and the results may be bad if the intrinsic parameters of the cameras of your videos differ much from those of Human3.6M. This may be particularly noticeable with fisheye cameras, which present a high degree of non-linear lens distortion. If the camera parameters are known, consider preprocessing your videos to match those of Human3.6M as closely as possible.
- If you want multi-person tracking, you should implement a bounding box matching strategy. An example would be to use bipartite matching on the bounding box overlap (IoU) between subsequent frames, but there are many other approaches.
- Predictions are relative to the root joint, i.e. the global trajectory is not regressed. If you need it, you may want to use another model to regress it, such as the one we use for semi-supervision.
- Predictions are always in *camera space* (regardless of whether the trajectory is available). For our visualization script, we simply take a random camera from Human3.6M, which fits decently most videos where the camera viewport is parallel to the ground. 