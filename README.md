# Reproducing *Time-Contrastive Networks: Self-Supervised Learning from Video*
**Authors:** Max Waterhout (5384907), Amos Yususf (4361504), Tingyu Zhang (5478413)
***
In this blog post, we present the results of our attempted replication and study of the 2018 paper by Pierre Sermanet et al. 
*Time-Contrastive Networks: Self-Supervised Learning from Video* [[1]](#1). This work is part of the CS4245 Seminar Computer Vision
by Deep Learning course 2021/2022 at TU Delft. This whole reproduction is done from scratch and can be found in our github: https://github.com/maxiew123/TCN_self_supervised_learning/tree/main

***
***

## 1. Introduction
In the computer vision domain, deep neural networks have been successful on a big range of tasks where labels can easily be specified by humans, like object detection and segmentation. A bigger challenge lies in applications that are difficult to label, like in the robotics domain. An example would be labeling a pouring task. How can a robot understand what important properties are while neglecting setting changes? Ideally, a robot in the real world can learn a pouring task purely from observation and understanding how to imitate this behavior directly. In this reproduction, we train a network on a pouring task that tries to learn the important features like the pose and the amount of liquid in the cup while being viewpoint and setting invariant. This pouring task is learned through the use of self-supervised learning and representation learning. In the following, we will provide a motivation for this paper, our implementation of the model, the results that we achieved against the benchmarks and lastly we discuss the limitations of our implementation. 

<p align="center">
<img src="images/pouring_002.gif" width="500" height="300"/> </br>
<em>fig 1. An example sequence of a pouring task</em>
</p>


## 2. Motivation
Imitation learning has already been used for learning robotic skills from demonstrations [[2]](#2) and can be split into two areas: behavioral cloning and inverse reinforcement learning. The main disadvantage of these methods is the need for a demonstration in the same context as the learner. This does not scale well with different contexts, like a changing viewpoint or an agent with a different model. In this paper, a Time-Contrastive Network (TCN) is trained on demonstrations that are diverse in embodiments, objects and backgrounds. This allows the TCN to learn the best pouring representation without labels. Eventually with this representation a robot can use this as a reward function. The robot can learn to link its images to the corresponding motor commands using reinforcement learning or another method. In our blog, we do not cover the reinforcement learning part.


## 3. Implementation
For our implementation of the TCN we only use the data of the single-view data. The orignal single-view data set contains total of 21 water pouring videos where four of them are fake pouring behaviours. No liquids are poured  from these four videos and we did not include the fake pouring videos for either training or testing. The preprocessing stage includes video frame resizing and frame normalization.We first resize the original frame from 1080 x 1920 to 360 x 640. The mean and standard deviation values for normalization are found from  the pytorch Inception net V3. We than concatenate all training videos together. Hence, the input of the TCN is a sequence of preprocessed 360x640 frames. In total 11 sequences of around 5 seconds (~40 frames for each video) are used for training. The framework contains a deep network that outputs a 32-dimensional embedding vector, see fig [1].  


<p align="center">
<img src="images/single view TCN.png" width="360" height="261" alt="single view TCN"> </br>
<em>Fig. 1: The single-view TCN</em>
</p>


### 3.1 Training
The loss is calculated with a triplet loss [[3]](#3). The formula and an illustration can be seen in fig [2]. This loss is calculated with an anchor, positive and negative frame. For every frame in a sequence, The TCN encourages the anchor and positive to be close in embedding space while distancing itself from the negative frame. This way the 
learns what is common between the anchor and positive frame and different from the negative frame. In our case the negative margin range is 0.2 seconds (one frame) and negatives always come from the same sequence as the positive. \


<p align="center">
<img src="images/triplet loss formula.png" width="700" height="105" > </br>
</p>

<p align="center">
<img src="images/triplet_loss.png" width="600" height="161" alt="Training loss"> </br>
<em>Fig. 2: The triplet loss</em>
</p>

The main purpose of the triplet loss is to learn representations without labels and simultaneously learn meaningful features like pose while being invariant to viewpoint,scale occlusion, background etc.. \ 

## 3.2 Deep network
The deep network is used for feature extraction. This framework is derived from an Inception architecture initialized with ImageNet pre-trained weights. The architecture is up until the "Mixed-5D" layer followed by two 2 convolutional layers, a spatial softmax layer and a fully connected layer. Note that the spatial solftmax layer outputs the x and y cordinates of the maximum element multiplies that element from each layer.
<p align="center">
<img src="images/SS.png" width="360" height="160" alt="Spatial Softmanx"> </br>
<em>Fig. 2: Spatial Softmax</em>
</p>
Sine our reference paper did not give the convolution kernel size, we followed [https://arxiv.org/pdf/1504.00702.pdf], and used 5x5 Conv + ReLu.

## 4. Results
For the results we used accuracy measured by video allignment. The allignment captures how well a model can allign a video. The allignment metrics that are used are the L2 norm and the cosine simularity. The metric matches the nearest neighbors, in embedding space, with eachother. In this way, for each frame the most semantically similar frame is returned. We state that a true positive is when a frame lies in the positive range from eachother. This way frame sequence: [1,2] gives the same accuracy as [2,1]. The tolerence value in the evaluation function will further increase the positive range.
We compare our results against the pre-trained Inception-ImageNet model [[4]](#4). We use the 2048D output vector of the last layer before the classifier net as a baseline. Same baseline was used in our reference paper. 

### 4.1 Final result overview
Model is trained on the Google Cloud with one P100 GPU. SGD, SGD with momentum, and Adam were used during different training epochs. Between 1 to 800 epochs, the optimizer was the SGD and between 800 to 4200 epochs, we switched the optimizer to SGD with momentum because the improvement on the loss was slow. After 4200 epochs, we used Adam as the optimizer for the same reason. During the training, single view dataset was used and there were total of 17 videos (fake pouring videos were not used). Each video lasts 7 seconds and contains scenes of pouring taking from the front view. 11 videos were used as training dataset and the rest were for testing. Because there was no validation set to select the best training model, we only saved models for every 200 epochs and for models that had the new minimum losses. In the end, we trained the model for 13k iterations and the training loss is shown in Figure 1. The zigzaging behaviour is due to the 200 epoch gap as well as the missing data betweening 2000 to 6000 epochs after one virtual machine crash.   

<p align="center">
<img src="images/tain loss.png" width="360" height="261" alt="Training loss"> </br>
<em>Fig. 3: The training loss</em>
</p>



<p align="center">
<img src="./images/accuracy.png" width="360" height="261" alt="Figure 1 paper"> </br>
<em>Fig. 4: The testing accuracy</em>
</p>
The alignment accuracy from each saved network model for the testing set is ploted in figure 2. Various criterion were used to measure the similarity between two embedded frames, such as consine similarity and euclidean distance (l2). We paid more focus on the l2 distance with one frame tolerence because this setup is closely related to the training procedure.  
The best accuracy measured by that criteria is from the model at the 7200th iteration. The average alignment accuracy is 80.11 percent whereas the Baseline method has an average accuracy of 71.04 percent. 

<p align="center">
<img src="./images/everything.gif" width="360" height="261"> </br>
<em>Fig. 5: Overview</em>
</p>



https://user-images.githubusercontent.com/99979529/171060214-c9998001-4c61-43a1-82c8-dca6ab182bcd.mp4













### 4.2 Reproduced figure/ table
| Method  | Alignment Accuracy (kNN)  | Alignment Accuracy (l2, tor = 1) | Training iteration |
| :------------                  |:---------------:|:---------------:| -----:|
| Baseline                       | 70.2%*           | 71.0%           | -     |
| Random                         |      71.9% *     | -               |      -|
| Single-view TCN (max acc)      | -               |    80.1%        | 7.2k  |
| Single-view TCN (min loss )    | -               |    75.0%        | 11781  |
| Single-view TCN (max itr)      | -               |    76.1%        | 13k   |
| Single-view TCN (literature) [1]| 74.2% *         |    -            |266k   |

In the Table above, data with * are from the reference paper and the k neareast neighbour scheme was used for accuracy measurement. Our alighnment accuracy was measured from l2 distance since we did not train any classifier. The two accuracy measurements give the similar score for Baseline model, 70.2% and 71% percent. Although our training iteration was limited by the hardware, the increment on the accuracy matches with historical data which means that the model did learn the water pouring representation from the triplet loss. We contribute our higher accuracy results to the small sample size because the reference model was trained on multi-view data set where as we only trained the model for single view dataset. 


## 5. Discussion and Limitations

### 5.1 Discussion
1. Overal performance on our results
2. reEmphasis on what we did: evaluation scheme l2. 
3. future: add multiview dataset. Evaluate network on imitation tasks 
### 5.2 Limitations
1. preprocessing unknow. (normalization and reshape size). sol: Inception net for normalization. readme for reshape
2. computing power limits and cost (google cloud)
3. code from author was expired



## References
<a id="1">[1]</a> Sermanet, P., Corey, L., Chebotar Y., Hsu J., Jang E., Schaal S., Levine S., Google Brain (2018). Time-Contrastive Networks: Self-Supervised Learning from Video. <i>University of South California</i>. [https://arxiv.org/abs/1704.06888]() \
<a id="2">[2] </a> J.A. Ijspeert, J. Nakanishi, and S. Schaal. Movement imitation
with nonlinear dynamical systems in humanoid robots. In
ICRA, 2002.\
<a id="3">[3] </a> X. Wang and A. Gupta. Unsupervised learning of visual
representations using videos. CoRR, abs/1505.00687, 2015.\
<a id="4">[4] </a> J. Deng, W. Dong, R. Socher, L.-J. Li, K. Li, and L. Fei-Fei. ImageNet: A Large-Scale Hierarchical Image Database. In
CVPR, 2009.











