## 使用Paddle-TRT进行ResNet50图像分类样例

该文档为使用Paddle-TRT预测在ResNet50分类模型上的实践demo。如果您刚接触Paddle-TRT，推荐先访问[这里](https://paddle-inference.readthedocs.io/en/latest/optimize/paddle_trt.html)对Paddle-TRT有个初步认识。

### 获取paddle_inference预测库

下载paddle_inference预测库并解压存储到`Paddle-Inference-Demo/c++/lib`目录，lib目录结构如下所示

```
Paddle-Inference-Demo/c++/lib/
├── CMakeLists.txt
└── paddle_inference
    ├── CMakeCache.txt
    ├── paddle
    │   ├── include                                    C++ 预测库头文件目录
    │   │   ├── crypto
    │   │   ├── internal
    │   │   ├── paddle_analysis_config.h
    │   │   ├── paddle_api.h
    │   │   ├── paddle_infer_declare.h
    │   │   ├── paddle_inference_api.h                 C++ 预测库头文件
    │   │   ├── paddle_mkldnn_quantizer_config.h
    │   │   └── paddle_pass_builder.h
    │   └── lib
    │       ├── libpaddle_inference.a                  C++ 静态预测库文件
    │       └── libpaddle_inference.so                 C++ 动态态预测库文件
    ├── third_party
    │   ├── install                                    第三方链接库和头文件
    │   │   ├── cryptopp
    │   │   ├── gflags
    │   │   ├── glog
    │   │   ├── mkldnn
    │   │   ├── mklml
    │   │   ├── protobuf
    │   │   └── xxhash
    │   └── threadpool
    │       └── ThreadPool.h
    └── version.txt
```

本目录下，

- `trt_fp32_test.cc` 为使用Paddle-TRT进行FP32精度预测的样例程序源文件（程序中的输入为固定值，如果您有opencv或其他方式进行数据读取的需求，需要对程序进行一定的修改）。
- `trt_gen_calib_table_test.cc` 为离线量化校准中，产出量化校准表的样例程序源文件。
- `trt_int8_test.cc` 为使用Paddle-TRT进行Int8精度预测的样例程序源文件，根据传入布尔类型参数`use_calib`为`true`或`false`，可以进行加载离线量化校准表进行Int8预测，或加载PaddleSlim量化产出的Int8模型进行预测。
脚本`compile.sh` 包含了第三方库、预编译库的信息配置。
脚本`run.sh` 一键运行脚本。

### 获取模型
首先，我们从下列链接下载所需模型：

[ResNet50 FP32模型](https://paddle-inference-dist.bj.bcebos.com/Paddle-Inference-Demo/resnet50.tgz)

[ResNet50 PaddleSlim量化模型](https://paddle-inference-dist.bj.bcebos.com/inference_demo/python/resnet50/ResNet50_quant.tar.gz)

其中，FP32模型用于FP32精度预测，以及Int8离线校准预测；量化模型由模型压缩工具库PaddleSlim产出，PaddleSlim模型量化相关信息可以参考[这里](https://paddlepaddle.github.io/PaddleSlim/quick_start/quant_aware_tutorial.html)。使用Paddle-TRT进行Int8量化预测的介绍可以参考[这里](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/docs/optimize/paddle_trt.rst#int8%E9%87%8F%E5%8C%96%E9%A2%84%E6%B5%8B)。

### 一、使用TRT FP32精度预测

1）**修改`run_impl.sh`**

打开`compile.sh`，我们对以下的几处信息进行修改：

```shell
DEMO_NAME=trt_fp32_test

# 根据预编译库中的version.txt信息判断是否将以下三个标记打开
WITH_MKL=ON
WITH_GPU=ON
USE_TENSORRT=ON

# 配置预测库的根目录
LIB_DIR=${work_path}/../lib/paddle_inference

# 如果上述的WITH_GPU 或 USE_TENSORRT设为ON，请设置对应的CUDA， CUDNN， TENSORRT的路径。
CUDNN_LIB=/usr/lib/x86_64-linux-gnu/
CUDA_LIB=/usr/local/cuda/lib64
TENSORRT_ROOT=/usr/local/TensorRT-6.0.1.5
```

运行 `bash compile.sh`， 会在目录下产生build目录。


2） **运行样例**

```shell
# 运行样例
./build/trt_fp32_test --model_file resnet50/inference.pdmodel --params_file resnet50/inference.pdiparams
```

运行结束后，程序会将模型预测输出的前20个结果打印到屏幕，说明运行成功。

### 二、使用TRT Int8离线量化预测

使用TRT Int8离线量化预测分为两步：生成量化校准表，以及加载校准表执行Int8预测。需要注意的是TRT Int8离线量化预测使用的仍然是ResNet50 FP32 模型，是通过校准表中包含的量化scale在运行时将FP32转为Int8从而加速预测的。

#### 生成量化校准表

1）**修改`compile.sh`**

打开`compile.sh`，我们对以下的几处信息进行修改：

```shell
# 选择生成量化校准表的demo
DEMO_NAME=trt_gen_calib_table_test

# 本节中，我们使用了TensorRT，需要将USE_TENSORRT打开
WITH_MKL=ON       
WITH_GPU=ON         
USE_TENSORRT=ON

# 配置预测库的根目录
LIB_DIR=${work_path}/../lib/paddle_inference

# 如果上述的WITH_GPU 或 USE_TENSORRT设为ON，请设置对应的CUDA， CUDNN， TENSORRT的路径。请注意CUDA和CUDNN需要设置到lib64一层，而TensorRT是设置到根目录一层
CUDNN_LIB=/usr/lib/x86_64-linux-gnu/
CUDA_LIB=/usr/local/cuda/lib64
TENSORRT_ROOT=/usr/local/TensorRT-6.0.1.5
```

运行 `bash compile.sh`， 会在目录下产生build目录。

2） **运行样例**

```shell
# 运行样例
./build/trt_gen_calib_table_test --model_file resnet50/inference.pdmodel --params_file resnet50/inference.pdiparams
```

运行结束后，模型文件夹`ResNet50`下的`_opt_cache`文件夹下会多出一个名字为`trt_calib_*`的文件，即校准表。

#### 加载校准表执行Int8预测

1） 再次修改`compile.sh`，换成执行Int8预测的demo：

```shell
# 选择执行Int8预测的demo
DEMO_NAME=trt_int8_test

# 本节中，我们使用了TensorRT，需要将USE_TENSORRT打开
WITH_MKL=ON       
WITH_GPU=ON         
USE_TENSORRT=ON

# 配置预测库的根目录
LIB_DIR=${work_path}/../lib/paddle_inference

# 如果上述的WITH_GPU 或 USE_TENSORRT设为ON，请设置对应的CUDA， CUDNN， TENSORRT的路径。请注意CUDA和CUDNN需要设置到lib64一层，而TensorRT是设置到根目录一层
CUDNN_LIB=/usr/lib/x86_64-linux-gnu/
CUDA_LIB=/usr/local/cuda/lib64
TENSORRT_ROOT=/usr/local/TensorRT-6.0.1.5
```

运行 `bash compile.sh`， 会在目录下产生build目录。

2） **运行样例**

```shell
# 运行样例，注意此处要将use_calib配置为true
./build/trt_int8_test --model_file resnet50/inference.pdmodel --params_file resnet50/inference.pdiparams --use_calib=true
```

运行结束后，程序会将模型预测输出的前20个结果打印到屏幕，说明运行成功。

**Note**

观察`trt_gen_calib_table_test`和`trt_int8_test`的代码可以发现，生成校准表和加载校准表进行Int8预测的TensorRT配置是相同的，都是

```c++
config.EnableTensorRtEngine(1 << 30, FLAGS_batch_size, 5, AnalysisConfig::Precision::kInt8, false, true /*use_calib*/);
```

Paddle-TRT判断是生成还是加载校准表的条件是模型目录下`_opt_cache`文件夹里是否有一个名字为`trt_calib_*`的与当前模型对应的校准表文件。在运行时为了防止混淆生成与加载过程，可以通过观察运行log来区分。

生成校准表的log：

```
I0623 08:40:49.386909 107053 tensorrt_engine_op.h:159] This process is generating calibration table for Paddle TRT int8...
I0623 08:40:49.387279 107057 tensorrt_engine_op.h:352] Prepare TRT engine (Optimize model structure, Select OP kernel etc). This process may cost a lot of time.
I0623 08:41:13.784473 107053 analysis_predictor.cc:791] Wait for calib threads done.
I0623 08:41:14.419198 107053 analysis_predictor.cc:793] Generating TRT Calibration table data, this may cost a lot of time...
```

加载校准表预测的log：

```
I0623 08:40:27.217701 107040 tensorrt_subgraph_pass.cc:258] RUN Paddle TRT int8 calibration mode...
I0623 08:40:27.217834 107040 tensorrt_subgraph_pass.cc:321] Prepare TRT engine (Optimize model structure, Select OP kernel etc). This process may cost a lot of time.
```

### 三、使用TRT 加载PaddleSlim Int8量化模型预测

这里，我们使用前面下载的ResNet50 PaddleSlim量化模型。与加载离线量化校准表执行Int8预测的区别是，PaddleSlim量化模型已经将scale保存在模型op的属性中，这里我们就不再需要校准表了，所以在运行样例时将`use_calib`配置为false。

1）**修改`compile.sh`**

打开`compile.sh`，我们对以下的几处信息进行修改：

```shell
# 选择使用Int8预测的demo
DEMO_NAME=trt_int8_test

# 本节中，我们使用了TensorRT，需要将USE_TENSORRT打开
WITH_MKL=ON       
WITH_GPU=ON         
USE_TENSORRT=ON

# 配置预测库的根目录
LIB_DIR=${work_path}/../lib/paddle_inference

# 如果上述的WITH_GPU 或 USE_TENSORRT设为ON，请设置对应的CUDA， CUDNN， TENSORRT的路径。请注意CUDA和CUDNN需要设置到lib64一层，而TensorRT是设置到根目录一层
CUDNN_LIB=/usr/lib/x86_64-linux-gnu/
CUDA_LIB=/usr/local/cuda/lib64
TENSORRT_ROOT=/usr/local/TensorRT-6.0.1.5
```

运行 `bash compile.sh`， 会在目录下产生build目录。


2） **运行样例**

```shell
# 运行样例，注意此处要将use_calib配置为false
./build/trt_int8_test --model_dir=./ResNet50_quant/ --use_calib=false
```

运行结束后，程序会将模型预测输出的前20个结果打印到屏幕，说明运行成功。

### 更多链接
- [Paddle Inference使用Quick Start！](https://paddle-inference.readthedocs.io/en/latest/)
- [Paddle Inference C++ Api使用](https://paddle-inference.readthedocs.io/en/latest/quick_start/cpp_demo.html)
- [Paddle Inference Python Api使用](https://paddle-inference.readthedocs.io/en/latest/quick_start/python_demo.html)

### 四、使用TRT dynamic shape变长输入功能

TRT的默认模式限制输入是定长的，即所有输入的shape大小必须相同，并与模型输入大小一致。如果需要输入不同尺寸的图片，需要设置变长输入模型。相关接口如下：

```c++
std::map<std::string, std::vector<int>> min_input_shape = {
    {"image", {FLAGS_batch_size, 3, 112, 112}}};
std::map<std::string, std::vector<int>> max_input_shape = {
    {"image", {FLAGS_batch_size, 3, 448, 448}}};
std::map<std::string, std::vector<int>> opt_input_shape = {
    {"image", {FLAGS_batch_size, 3, 224, 224}}};
config.SetTRTDynamicShapeInfo(min_input_shape, max_input_shape,
                            opt_input_shape);
```

其中 `min_input_shape` 、`max_input_shape` 、`opt_input_shape` 都是string到int类型的map，表示输入变量的最小、最大和最优shape，最优的含义是指，TRT会以最优shape来进行优化，理论上，接近最优shape的输入图片性能最佳。

上面几个map的key是指TRT子图的输入变量名，注意不一定与模型的输入名一致。若存在多个TRT子图，需要对每一个子图的所有输入变量配置最小、最大和最优shape。

运行样例的方式与使用TRT FP32精度预测相同，运行脚本换为`trt_dynamic_shape_test`即可。

