# VSLAM之maplab篇

## maplab介绍

[maplab](https://github.com/ethz-asl/maplab)是ETH的ASL团队在2018年开源的一个视觉+惯性导航的建图框架，可以实现多场景建图的创建、处理和操作。它包含了两部分：在线的前端ROVIOLI用于创建地图，可以实现建图和定位功能；离线的地图处理包maplab_console，可以为生成的地图进行二次处理（生成地图优化、组合和建立稠密地图等）。其工作流程如下图所示

![tAMUG4.png](https://s1.ax1x.com/2020/05/27/tAMUG4.png)

- (a)  在VIO模式下使用ROVIOLI进行建图。

- (b)  使用maplab console对地图进行优化，实现不同地图的融合。

- (c)  在定位模式下运行ROVIOLI，定位的功能提高了视觉+惯性导航的位姿估计精度。

## maplab的编译

按照[官方教程](https://github.com/ethz-asl/maplab/wiki/Installation-Ubuntu)进行编译，

- 首先安装依赖

需要注意，clang-format-3.8已经无法在18.04的ubuntu上使用了，需要安装3.9版本.

```
# Install ROS 
export UBUNTU_VERSION=bionic #(Ubuntu 16.04: xenial, Ubuntu 14.04: trusty, Ubuntu 18.04: bionic)
export ROS_VERSION=melodic #(Ubuntu 16.04: kinetic, Ubuntu 14.04: indigo, Ubuntu 18.04: melodic)

# NOTE: Follow the official ROS installation instructions for melodic.
sudo apt install software-properties-common
sudo add-apt-repository "deb http://packages.ros.org/ros/ubuntu $UBUNTU_VERSION main"
wget https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -O - | sudo apt-key add -
sudo apt update
sudo apt install ros-$ROS_VERSION-desktop-full "ros-$ROS_VERSION-tf2-*" "ros-$ROS_VERSION-camera-info-manager*" --yes


# Install framework dependencies.
# NOTE: clang-format-3.8 is not available anymore on bionic, install a newer version.
sudo apt install autotools-dev ccache doxygen dh-autoreconf git liblapack-dev libblas-dev libgtest-dev libreadline-dev libssh2-1-dev pylint clang-format-3.9 python-autopep8 python-catkin-tools python-pip python-git python-setuptools python-termcolor python-wstool libatlas3-base --yes

sudo pip install requests
```

- 然后更新ROS环境

```
sudo rosdep init
rosdep update
echo ". /opt/ros/$ROS_VERSION/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

- 创建catkin工作空间

```
export ROS_VERSION=kinetic #(Ubuntu 16.04: kinetic, Ubuntu 14.04: indigo)
export CATKIN_WS=~/maplab_ws
mkdir -p $CATKIN_WS/src
cd $CATKIN_WS
catkin init
catkin config --merge-devel # Necessary for catkin_tools >= 0.4.
catkin config --extend /opt/ros/$ROS_VERSION
catkin config --cmake-args -DCMAKE_BUILD_TYPE=Release
cd src
```

- 下载

```
git clone https://github.com/ethz-asl/maplab.git --recursive
git clone https://github.com/ethz-asl/maplab_dependencies --recursive
```

- 编译：

```
cd $CATKIN_WS
catkin build maplab
```

- 编译maplab遇到问题及相应解决方法：

1. 下载依赖遇到问题：

   当编译依赖时，会首先下载源码，有时网络不好的话会导致下载失败，进而导致编译失败，解决方法可以参照官方[FAQ](https://github.com/ethz-asl/maplab/wiki/FAQ#q-why-do-i-get-missing-dependencies-when-building-the-maplab-workspace)，首先离线下载源码，然后copy到相应位置，比如opencv_catkin，可以从

   ```
   https://github.com/Itseez/opencv/archive/3.2.0.zip
   ```

   中下载，然后将源码解压至：

   ```
   ~/maplab_ws/build/opencv3_catkin/opencv3_src
   ```

   

2. 编译opencv3_catkin遇到问题`~/maplab_ws/build/opencv3_catkin/opencv3_src/cmake/OpenCVCompilerOptions.cmake`报以下错误：

   ```
   A duplicate ELSE command was found inside an IF block.
   ```

   在OpenCVCompilerOptions.cmake文件中注释掉报错的对应行即可。

3. 遇到brisk-detector源码问题：解决方法参照https://github.com/ethz-asl/ethzasl_brisk/pull/109，在brisk-feature-detector.cc中加入包含#include <functional>

## maplab运行

本部分介绍两种方法运行ROVIOLI进行建图，分别是使用rosbag和rostopic；然后运行了定位模式下的ROVIOLI，即加载建好的地图进行定位的方式。

- 需要的配置文件：

  相机标定文文件，[文件样式](https://github.com/ethz-asl/maplab/blob/master/applications/rovioli/share/ncamera-euroc.yaml)

  IMU参数-maplab，[文件样式](https://github.com/ethz-asl/maplab/blob/master/applications/rovioli/share/imu-adis16488.yaml)

  IMU参数-rovio，[文件样式](https://github.com/ethz-asl/maplab/blob/master/applications/rovioli/share/imu-sigmas-rovio.yaml)

  Rovio标定文件，[文件样式](https://github.com/ethz-asl/maplab/blob/master/applications/rovioli/share/rovio_default_config.info)

  默认的配置文件路径为`maplab_ws/src/maplab/applications/rovioli/share`

- 从rosbag中建图：

  首先从Euros[数据集](http://projects.asl.ethz.ch/datasets/doku.php?id=kmavvisualinertialdatasets)中下载rosbag，然后运行命令

  ```
  # Make sure that your maplab workspace is sourced!
  source ~/maplab_ws/devel/setup.bash
  roscore&
  rosrun rovioli tutorial_euroc save_folder MH_01_easy.bag
  ```

  其中，运行脚本路径为：`maplab_ws/src/maplab/applications/rovioli/scripts/tutorials`

  `/tutorial_euroc`，包含配置文件的路径和运行模式的选择等。 `save_folder` 为生成地图的保存路径，`MH_01_easy.bag`为rosbag的保存路径。

  运行效果如下图：[![tAy7Je.md.png](https://s1.ax1x.com/2020/05/27/tAy7Je.md.png)](https://imgchr.com/i/tAy7Je)

  [![tAyveP.md.png](https://s1.ax1x.com/2020/05/27/tAyveP.md.png)](https://imgchr.com/i/tAyveP)

  地图保存在相应的路径下：

  [![tA6Jw6.png](https://s1.ax1x.com/2020/05/27/tA6Jw6.png)](https://imgchr.com/i/tA6Jw6)

- 从rostopic中建图：

  使用小觅相机(S1030)作为传感器，首先需要将相机kalibr标定结果转换为maplab的配置文件的格式。从相机标定[文件](https://github.com/ethz-asl/maplab/blob/master/applications/rovioli/share/ncamera-euroc.yaml)中可知，除了相机的内参和畸变系数以外，还有相机坐标系到机体坐标系的外参转换矩阵T_B_C。由kalibr的标定结果可以获得`T_ic:  (cam0 to imu0)`即相机到IMU坐标系的转换，此处机体系和IMU系认为重合，故可在maplab的外参配置文件中使用该矩阵。

  使用以下脚本启动：

  ```
  LOCALIZATION_MAP_OUTPUT=$1
  NCAMERA_CALIBRATION="$ROVIO_CONFIG_DIR../mynteye/ncameras_pin_equ.yaml"
  IMU_PARAMETERS_MAPLAB="$ROVIO_CONFIG_DIR../mynteye/imu-icm20602.yaml"
  IMU_PARAMETERS_ROVIO="$ROVIO_CONFIG_DIR../mynteye/imu-sigmas-rovio.yaml"
  REST=$@
  
  rosrun rovioli rovioli \
    --alsologtostderr=1 \
    --v=2 \
    --ncamera_calibration=$NCAMERA_CALIBRATION  \
    --imu_parameters_maplab=$IMU_PARAMETERS_MAPLAB \
    --imu_parameters_rovio=$IMU_PARAMETERS_ROVIO \
    --datasource_type="rostopic" \
    --save_map_folder="$LOCALIZATION_MAP_OUTPUT" \
    --map_builder_save_image_as_resources=true \
    --rovio_enable_frame_visualization \
    --optimize_map_to_localization_map=false $REST \
  ```

  该脚本保存位置为`~/maplab_ws/src/maplab/applications/rovioli/scripts/mynt/tutorial_`

  `mynt_live_stereo_pinhole_equ`.

  首先启动小觅相机，然后启动rovioli：

  ```
  roslaunch mynt_eye_ros_wrapper display.launch
  source ~/maplab_ws/devel/setup.bash
  rosrun rovioli tutorial_mynt_live_stereo_pinhole_equ mynt_stereo_live
  ```

  运行效果如下所示：

  [![tZoD4H.png](https://s1.ax1x.com/2020/05/28/tZoD4H.png)](https://imgchr.com/i/tZoD4H)

  结束运行后同样会对地图进行保存

- 定位模式下运行ROVIOLI：

  定位模式会大大提高定位的精度，降低轨迹估计的漂移。和VIO模式一样，定位模式同样需要三样标定文件(camera, IMU maplab and IMU rovio)，另外，还需要一个经过优化过的之前生成的VI map。

  产生地图的方式有两种，一种是使用maplab console的方式[手动优化地图](https://github.com/ethz-asl/maplab/wiki/Preparing-a-single-session-map)，另一种是在VIO模式运行rovioli时便将`--optimize_map_to_localization_map`设置为true，这样rovioli运行结束后便会生成一张优化后的地图。

  可以选择rosbag运行定位模式，参照[官方教程](https://github.com/ethz-asl/maplab/wiki/Running-ROVIOLI-in-Localization-mode)，这里不作介绍了，下面主要介绍运行小觅相机使用rostopic的方式。

  运行以下脚本：

  ```
  LOCALIZATION_MAP_INPUT=$1
  LOCALIZATION_MAP_OUTPUT=$2
  NCAMERA_CALIBRATION="$ROVIO_CONFIG_DIR../mynteye/ncameras_pin_equ.yaml"
  IMU_PARAMETERS_MAPLAB="$ROVIO_CONFIG_DIR../mynteye/imu-icm20602.yaml"
  IMU_PARAMETERS_ROVIO="$ROVIO_CONFIG_DIR../mynteye/imu-sigmas-rovio.yaml"
  REST=$@
  
  rosrun rovioli rovioli \
    --alsologtostderr=1 \
    --v=2 \
    --ncamera_calibration=$NCAMERA_CALIBRATION  \
    --imu_parameters_maplab=$IMU_PARAMETERS_MAPLAB \
    --imu_parameters_rovio=$IMU_PARAMETERS_ROVIO \
    --publish_debug_markers  \
    --datasource_type="rostopic" \
    --optimize_map_to_localization_map=false \
    --vio_localization_map_folder=$LOCALIZATION_MAP_INPUT \
    --save_map_folder=$LOCALIZATION_MAP_OUTPUT \
    --map_builder_save_image_as_resources=false \
    --datasource_rosbag=$ROSBAG $REST
    --rovio_enable_frame_visualization \
  ```

  运行完成后会保存定位地图，启动小觅相机，然后启动rovioli：

  ```
  source ~/maplab_ws/devel/setup.bash
  roscore&
  rosrun rovioli tutorial_mynt_live_stereo_pinhole_equ_loc save_folder_loc_localization save_map_with_localization
  ```

  其中`save_folder_loc_localization` 为保存的定位地图路径，运行时选择加载。运行的效果如下所示，相比于VIO模式下的话题，在定位模式下的话题包含更多的关于定位信息的话题：

  - `/loopclosure_database`: Point cloud of all landmarks in the loop closure database.

  - `/loop_closures`: Lines connecting the current active vertex with all landmarks that have been used for the localization.
  - `/loopclosure_inliers`: Point cloud with all landmarks that have been used for the last successful localization.
  - `/debug_T_G_I_raw_localization`: Red points indicating the vertex positions where a successful localization occurred.

  [![teV8gA.png](https://s1.ax1x.com/2020/05/28/teV8gA.png)](https://imgchr.com/i/teV8gA)

