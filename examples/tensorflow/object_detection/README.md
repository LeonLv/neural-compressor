Step-by-Step
============

This document is used to list steps of reproducing TensorFlow ssd_resnet50_v1 iLiT tuning zoo result.


## Prerequisite

### 1. Installation
```Shell
# Install iLiT
pip instal ilit
```
### 2. Install Intel Tensorflow 1.15/2.0/2.1
```shell
pip intel-tensorflow==1.15.2 [2.0,2.1]
```

### 3. Install Additional Dependency packages
```shell
cd examples/tensorflow/object_detection && pip install -r requirements.txt
```

### 4. Install Protocol Buffer Compiler

`Protocol Buffer Compiler` in version higher than 3.0.0 is necessary ingredient for automatic COCO dataset preparation. To install please follow
[Protobuf installation instructions](https://grpc.io/docs/protoc-installation/#install-using-a-package-manager).

### 5. Prepare Dataset

#### Automatic dataset download
Run the `prepare_dataset.sh` script located in `examples/tensorflow/object_detection`.

Usage:
```shell
cd examples/tensorflow/object_detection
. prepare_dataset.sh
```

This script will download the *train*, *validation* and *test* COCO datasets. Furthermore it will convert them to
tensorflow records using the `https://github.com/tensorflow/models.git` dedicated script.

#### Manual dataset download
Download CoCo Dataset from [Official Website](https://cocodataset.org/#download).

### 6. Download Frozen PB

#### Automated approach
Run the `prepare_model.py` script located in `LowPrecisionInferenceTool/examples/tensorflow/object_detection`.

```
usage: prepare_model.py [-h] [--model_name {ssd_resnet50_v1,ssd_mobilenet_v1}]
                        [--model_path MODEL_PATH]

Prepare pre-trained model for COCO object detection

optional arguments:
  -h, --help            show this help message and exit
  --model_name {ssd_resnet50_v1,ssd_mobilenet_v1}
                        model to download, default is ssd_resnet50_v1
  --model_path MODEL_PATH
                        directory to put models, default is ./model
```

#### Manual approach

##### Ssd_resnet50_v1
```shell
wget http://download.tensorflow.org/models/object_detection/ssd_resnet50_v1_fpn_shared_box_predictor_640x640_coco14_sync_2018_07_03.tar.gz
tar -xvzf ssd_resnet50_v1_fpn_shared_box_predictor_640x640_coco14_sync_2018_07_03.tar.gz -C /tmp
```

##### Ssd_mobilenet_V1

```shell
wget http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_2018_01_28.tar.gz
tar -xvzf ssd_mobilenet_v1_coco_2018_01_28.tar.gz
```

## Run Command
  ```Shell
  # The cmd of running ssd_resnet50_v1
  python infer_detections.py --batch-size 1 --input-graph /tmp/ssd_resnet50_v1_fpn_shared_box_predictor_640x640_coco14_sync_2018_07_03/frozen_inference_graph.pb --data-location /path/to/dataset/coco_val.record --accuracy-only --config ssd_resnet50_v1.yaml
  ```

Details of enabling iLiT on ssd_resnet50_v1 for Tensorflow.
=========================

This is a tutorial of how to enable ssd_resnet50_v1 model with iLiT.
## User Code Analysis
1. User specifies fp32 *model*, calibration dataset *q_dataloader*, evaluation dataset *eval_dataloader* and metric in tuning.metric field of model-specific yaml config file.

2. User specifies fp32 *model*, calibration dataset *q_dataloader* and a custom *eval_func* which encapsulates the evaluation dataset and metric by itself.

For ssd_resnet50_v1, we applied the latter one because our philosophy is to enable the model with minimal changes. Hence we need to make two changes on the original code. The first one is to implement the q_dataloader and make necessary changes to *eval_func*.


### q_dataloader Part Adaption
Specifically, we need to add one generator to iterate the dataset per iLiT requirements. The easiest way is to implement *__iter__* interface. Below function will yield the images to feed the model as input.

```python
def __iter__(self):
    """Enable the generator for q_dataloader

    Yields:
        [Tensor]: images
    """
    data_graph = tf.Graph()
    with data_graph.as_default():
        self.input_images, self.bbox, self.label, self.image_id = self.get_input(
        )

    self.data_sess = tf.compat.v1.Session(graph=data_graph,
                                          config=self.config)
    for i in range(COCO_NUM_VAL_IMAGES):
        input_images = self.data_sess.run([self.input_images])
        yield input_images
```

### Evaluation Part Adaption
The Class model_infer has the run_accuracy function which actually could be re-used as the eval_func.

Compare with the original version, we added the additional parameter **input_graph** as the iLiT would call this interface with the graph to be evaluated. The following code snippet also need to be added into the run_accuracy function to update the class members like self.input_tensor and self.output_tensors.
```python
if input_graph:
    self.infer_graph = input_graph
    # Need to reset the input_tensor/output_tensor
    self.input_tensor = self.infer_graph.get_tensor_by_name(
        self.input_layer + ":0")
    self.output_tensors = [
        self.infer_graph.get_tensor_by_name(x + ":0")
        for x in self.output_layers
    ]
```

### Write Yaml config file
In examples directory, there is a ssd_resnet50_v1.yaml. We could remove most of items and only keep mandatory item for tuning.

```yaml
framework:
  - name: tensorflow
    inputs: image_tensor
    outputs: num_detections,detection_boxes,detection_scores,detection_classes

calibration:
  - iterations: 1, 5, 10, 20
    algorithm:
      - weight: minmax
        activation: minmax

tuning:
    accuracy_criterion:
      - relative: 0.01
    timeout: 0
    random_seed: 9527
```
Here we set the input tensor and output tensors name into *inputs* and *outputs* field. Meanwhile, we set mAp target as tolerating 0.01 relative mAp of baseline. The default tuning strategy is basic strategy. The timeout 0 means early stop as well as a tuning config meet accuracy target.

### Code update

After prepare step is done, we just need update infer_detections.py like below.
```python
import ilit

at = ilit.Tuner(args.config)
q_model = at.tune(infer.get_graph(),
                        q_dataloader=infer,
                        eval_func=infer.accuracy_check)
```

The iLiT tune() function will return a best quantized model during timeout constrain.