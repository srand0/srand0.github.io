---
layout: article
title:  'Cartographer(¡¡)!'
tags: Cartographer Strcture
aside:
  toc: true
mathjax: true
mathjax_autoNumber: true
cover: /image/posts/2020-03-08/1.cartographer-code.png
---

![cartographer-code](/image/posts/2020-03-08/1.cartographer-code.png)

分析cartographer前后端调用的逻辑，本文只针对2D情况进行分析。

<!--more-->

## 1. 接口

cartographer的接口类是MapBuilder。首先需要调用AddTrajectoryBuilder函数生成GlobalTrajectoryBuilder指针，并且作为一个元素插入到trajectory_builders_数据成员中（该成员为vector，内部可以存储TrajectoryBuilderInterface的子类）。

GlobalTrajectoryBuilder包含两个数据成员——local_trajectory_builder_和pose_graph_：local_trajectory_builder_是LocalTrajectoryBuilder2D的对象指针，也是前端操作的主类；pose_graph_是PoseGraph2D的对象指针，主要进行后端优化计算，所有的位姿和子图都会在该类中进行存储，最终的输出也由该指针生成。

## 2. 前端

前端的主要工作是读取各个传感器数据，短时间内的传感器累积误差还是比较小的，将传感器数据进行匹配后就可以创建一个局部地图，也称为子图(submap)，当子图中的传感器数据累积到一定的量之后，误差就比较大了，这时候就需要重新创建一个子图，再在新的子图上累加传感器数据。

如图所示，前端接收传感器数据的类为LocalTrajectoryBuilder2D，可以接收IMU、里程计(Odometry)、激光(Range)数据，分别对应AddImuData、AddOdometryData、AddRangeData。由于各个传感器的数据不可能总是同一个时刻到达，所以，需要PoseExtrapolator类将IMU和里程计的数据进行融合后推算出激光数据到达时刻机器人的位姿。

下一步就是将激光数据和推算出的位姿通过AddAccumulatedRangeData添加到子图中，该过程会调用前端最重要的算法ScanMatch，根据激光匹配计算出激光收集到的时刻机器人的最佳位姿。得到最佳位姿后会反馈给PoseExtrapolator用于以后的位姿推算，然后将激光数据添加到子图ActiveSubmaps2D中，随后GlobalTrajectoryBuilder2D会将添加过新激光数据的InsertionResult通过PoseGraph2D的AddNode函数添加到pose_graph_中。

到此，前端的工作基本完成了，下面就是后端的工作，主要在PoseGraph2D中完成。

## 3. 后端

从前面的叙述，我们已经知道，后端PoseGraph2D这个类中已经存储了所有的子图的信息。后端的主要工作是根据这些信息通过图优化算法，计算出机器人的最佳轨迹。

首先AddNode函数接收到新的InsertionResult后，会调用ComputeConstraintForNode函数进行图的约束的更新，当Node的数量（得到合法的激光数据的次数）满足条件后，会调用DispatchOptimization进行图优化算法的计算，即数据成员optimization_problem_调用Solve函数。

当图优化计算完成后，会得到从一开始到现在的最佳轨迹，就可以更新trajectory_nodes_和global_submap_poses_中的位姿了。

当所有的数据都录入完成后，直接从pose_graph_中导出连接后的子图，就可以得到一张完整的地图。



---


