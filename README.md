
# Deep Learning Blog

This repository is a clone of Frontier Development Labs' PyRain, and is the codebase for the CS4240 reproducibility project.

Authors:

**Keval Sakhida** (4646916) k.sakhida@student.tudelft.nl

**Datta Vemulapalli** (4447441) d.vemulapalli@student.tudelft.nl

**Danning Zhao** (5459583) D.ZHAO-3@student.tudelft.nl

## Introduction

This blog documents an attempt to reproduce the work carried out in “RainBench: Enabling Data-Driven Precipitation Forecasting on a Global Scale” by Catherine Tong et al. In this paper, the authors propose a global precipitation forecasting model, PyRain, using convolutional LSTMs (convLSTMs). This blog will discuss the architecture of PyRain, and RainBench – a multi modal dataset that could advance global precipitation forecasting capabilities. Thes goal of this project are to reproduce the benchmark results obtained by the authors, and further tweak the model to generate additional insights. The approach taken to meet these goals will be discussed.

## Dataset

RainBench, a dataset presented by the authors of the reference paper, is an amalgamation of three different sources of data. These are:

1.  **SimSat** – a model simulated databank generated by the European Centre for Medium Range Weather Forecasts (ECWMF)’s model that is based on Radiative transfer model. This data essentially provides information about various precipitation features such as cloud cover and moisture across the globe, at a spatial resolution of $0.1\degree$ (equivalent to 1000km) and a temporal resolution of 3 hours.

2.  **IMERG** – another source of data was based on the precipitation estimates provided by NASA’s polar and geo stationary satellites which were refined further through ERA5 and rain-gauge data. Once again, the data set had a spatial resolution of $0.1\degree$ and a temporal resolution of 30 minutes from June 2000.

3.  **ERA 5** – The data from this source originates from ERA5 Reanalysis Product which provides global hourly estimates. Unlike the two previous sources of data, this source provides additional atmospheric variables such as specific humidity, temperature and geopotential at different pressure levels. The spatial resolution remains $0.1\degree$ whilst the data can be traced back to 1979 with a lag time of 5 days.

In order to make the experiment more efficient, the spatial resolution of the aforementioned data was reduced from their original resolution to $5.625\degree$ by means of bilinear interpolation. The dataset to train the model was a subset of RainBench that started from April 2016. The data from 2018 to 2019 was used for validation by the authors. The variables geopotential, temperature, humidity, cloud liquid water content and cloud ice water content, sampled at different geopotential heights were used as the inputs of the model based on ERA5 dataset. On the other, inputs for the SimSat dataset were cloud brightness temperatures at three different wavelengths, normalized about its global mean.

## Model Architecture
### The ConvLSTM Model
The ConvLSTM [1] network is an extension of a full connected LSTM [2] (FC-LSTM) which has convolutional structures in both the input-to-state and state-to-state transitions. By using ConvLSTM, the authors are able to build a network model for the precipitation nowcasting problem.

The inputs X_1, ... X_t, cell outputs C_1, ... C_t, hidden states H_1, ... H_t, input gate i_t, forget gate f_t and output gate σ_t of ConvLSTM are 3D tensors whose dimensions are channels, and spatial dimensions (rows and columns). In order to facilitate the understanding of the input dimension, the spatial dimension can be regarded as the surface of the earth corresponding to a certain geographical range, that is, the space grid. The channels are understood as various climate data corresponding to the same geographical location. The climate data in the same space grid species are shared.

![Input Dimension](https://github.com/BadABrownie/OurRain/blob/master/images/img1.png)

The equations of ConvLSTM [1] are shown below, where * denotes the convolution operator and ∘ denotes the Hadamard product:

$i_t = \sigma(W_{xi}*X_t+W_{hi}*H_{t-1}+W_{ci}\circ C_{t-1}+b_i)$
$f_t = \sigma(W_{xf}*X_t+W_{hf}*H_{t-1}+W_{cf}\circ C_{t-1}+b_f)$
$C_t = f_t \circ C_{t-1}+i_t \circ \tanh(W_{xc}*X_t + W_{hc}*H_{t-1}+b_c)$
$o_t = \sigma(W_{xo}*X_t+W_{ho}*H_{t-1}+W_{co}\circ C_t+b_o)$

The drawback of FC-LSTM is the lack of no spatial information encoded with the usage of full connection in input-to-state and state-to-state transitions. The ConvLSTM overcomes this problem by predicating the future state of a certain cell in the grid with the data of its local neighbours, by taking advantage of convolution operator.

However, the convolution operation will shrink the width and height of the inputs and the states. Padding is needed in order to ensure that the states have the same spatial dimensions as the inputs. Padding of hidden padding of the hidden states on the boundary can be viewed as using *the state of the outside world* for calculation [1]. In more detail, initialization of the states H_0, C_0 of the LSTM to zero means ignoring the past before the earliest time sequence of inputs. Similarly, zero padding assumes no prior knowledge about the climate data outside the grid.

### Implementation of the ConvLSTMForecaster Model

The benchmark results of the paper are obtained by using the architecture of the MetNet model, ConvLSTMForecaster [4] and is composed as follows: a ConvLSTM model is used, consisting of two Conv2d layers and a ReLU layer, and structure the forecasting task based on MetNet’s [4] configurations. The inputs are a time series of standardized climate features, and the output is a precipitation forecast of one certain lead time.

**IMG2**
**IMG3**

### Model Training

The ConvLSTM receives as input a four-dimensional tensor of size [t, c, w, h]. The time dimension comprises t slices sampled every 3 hours over a 12 hour interval prior to $T_x$, where $T_x$ is the time at which the model makes a prediction into the future. “c” represents the channels which varies according to different input dataset, “w” and “h” are the index of space grid which represents a large geographic surface.

We use the choosen features of dataset according to Appendix C of the RainBench paper. The chosen input features for benchmark tasks are as follow: For ERA5, the time-varying features are geopotential (z), temperature (t), humidity (q), cloud liquid water content (clwc), cloud ice water content (ciwc), each sampled at 300 hPa, 500 hPa and 850 hPa geopotential heights and static features are latitude, longitude, land-sea mask, orography and soil type. For the SimSat dataset, the inputs features  are: cloud-brightness temperature (clbt) taken at three wavelengths. We normalize each variable with its global mean and standard deviation.

Moreover, the day, month and year corresponding to the input data and the lead time also needs to be concatenated into the input sequence. The date is embedded into three channels with the same width and height of input data. A forward pass through ConvLSTM makes a prediction for a single lead time. The model is informed about the desired lead time by concatenating this information with the descriptive input features. The lead times are sampled every 24 hours over a period of 120 hours. We use five layers of shape [w, h] to express a specific lead time. The layer corresponding to the leadtime is all filled with 1, and the other layers are all filled with 0 [4].

**IMG4**

According to the amount of input features above, the number of input channels of the model when using different input data datasets can be obtained

SimSat: 3(clbt) + 3(date) + 5(leadtime) = 11
ERA5: 5X3(time-varying) + 3(date) + 5(leadtime) + 5(constant) = 28
SimSat + ERA5: 3(clbt) + 5X3(time-varying) + 3(date) + 5(leadtime) + 5(constant) = 31

### Loss Function and Optimizer

We use the mean latitude-weighted RMSE [5] as a loss function

$$RMSE = \frac{1}{N_{forecasts}}\sum_i^{N_{forecasts}}\sqrt{\frac{1}{N_{lat}N_{lon}}\sum_j^{N_{lat}}\sum_k^{N_{lon}}L(j)(f_{i, j, k} - t_{i, j, k})^2}$$

where f is the model forecast and is the truth. L(j) is the latitude weighting factor for the latitude at the $j_{th}$ latitude index:

$$L(j) = \frac{\cos(lat(j))}{\frac{1}{N_{lat}}\sum_j^{N_{lat}}\cos(lat(j))}$$

We use an Adam optimizer.

### Climatology and Persistence

Climatology is defined as a past multiple year mean of precipitation of the same date for predicting. It is often used as the ‘control forecast’ in verification [6], meaning that any forecast method claiming to be useful should beat climatology in terms of an accuracy attribute which is the RMSE.

Persistence method considered the climate condition of next period is the same as current period, i.e.  is the forecast for. This is essentially a lazy forecast with no great expense involved, but because the atmosphere (and especially the ocean) varies slowly, except in special and somewhat rare circumstances.

## Reproduction Methodology
### Local machines
The PyRain github by Frontier Development Lab was cloned onto a Windows machine, and the required datasets were downloaded. The files amounted to almost 51GB, which meant that not all members participating in this project could apply this approach. The datasets consisted of memory map (.mmap) files, which were a novel format to the group. Furthermore, running the code on a local machine proved to be difficult due to the compatibility of required python packages, as well as the large amount of memory and computational power required. Additionally, orienting ourselves and attempting to understand the complex structure of the code added to the difficulties that were faced during the initial stages of this project. As such, this approach quickly turned into a dead end* due to a multitude of errors, the most prominent of which at this stage, were memory errors.

  

### Google Cloud Platform (Google Collab)

  
Google Cloud Platform (Google Collab): With support from our TA, we received computational credits for GCP. With no prior experience using this platform, fetching the datasets and getting code to run proved to be a challenge. While the data was retrieved off the Google cloud buckets with the use of Gsutil, the lack of access to Collab Pro meant that the disk space was rather limited when the GPU was enabled in runtime and meant that the program was not executable. This notebook can be found here: 

https://colab.research.google.com/drive/1FySqzpVjNm1Cwgm_L2c5TY2A_GTd-Oju?usp=sharing

At this point, the scope of our project was reduced to getting the code running to produce the baseline results, in collaboration with the other groups involved with reproducing the same paper.

### University Cluster

The final approach that was attempted, was to run the entire program on the TU Delft cluster. This was yet again a challenging task as neither of our group members attempting this approach had any experience with bash scripting or using a linux command line interface. However once this was overcome and the program was able to run on the cluster, we ran into errors that may be attributed to the structure of the code and the datasets involved. Upon deliberating with other groups and our TA, we concluded that it was currently infeasible to run the code as intended by the original authors, and neither group managed to successfully obtain any relevant outputs.

### *Bonus approach
The large datasets were resampled by one of our project members and attempts were made to build our own trainer, similar to the ones used in the assignments of CS4240. However, this approach proved to be rather time consuming and required a greater level of understanding of the author’s code. Both which were beyond the scope of this assignment. The repository for this approach can be found here:

https://github.com/Timelrod4021/MyRain

## Conclusion

We unfortunately failed to reproduce the baseline results of the paper and were far from performing additional studies. This is a direct result of not being able to get the code running as intended by the original authors, despite receiving additional support from our TA, and collaborating with other groups. As such, we cannot comment on the conclusions drawn by the authors of the paper.

We would like to stress the importance of reproductions , whether successful or not, as they can serve to inform other deep learning researchers and further our own knowledge. The hurdles encountered by our group may not be representative for everyone in this space and we wish for better success to anyone attempting this.

  

## Reflection and Task Division

### Keval Sakhida
*Data loading and execution of PyRain on a local machine and the TU Delft cluster*


Being a part of the reproducibility project was quite a learning experience. As an aerospace engineering student, the prospect of working with satellite data for a global precipitation model was exciting. However, despite having an existing codebase, it was difficult to carry out a reproduction without a thorough understanding of the original authors' code. Despite acquiring a high-level understanding of the code, we were unsuccessful at being able to run the program to obtain the baseline results and unfortunately never got to experiment with model tuning and further studies.

Tinkering with the code to attempt and reproduce the benchmark results was a challenge that required multiple approaches. In doing so, I did experience a few new python libraries which may be useful in later projects. By debugging and attempting to run the code on a local Windows machine, I realized that I was being limited by the machine's memory.   

Additionally, I learnt how to run tasks on a computer cluster as part of the second approach. As we tried to understand the linux command line interface, I developed an appreciation for GUI based systems, but I became comfortable with leveraging the computer cluster to work with a program requiring a large dataset.

### Datta Vemulapalli


*Data loading and execution of PyRain on Google Cloud and the TU Delft cluster*

This Project gave me a rather exciting opportunity to use actual satellite data! The objective of this study resonates quite strongly with me as this would help developing countries plan in advance of extreme weather events. I have learnt that reproducing the results of a published paper can be rather tricky despite having access to the author’s code. As we couldn’t reproduce the baseline results, the opportunity to fine tune the model and see the implications of it was unavailable. However, I made a positive stride towards being able to execute a code despite the lack of intricate understanding of the code.

I discovered a few handy python libraries, such a pytorch-lightning, and I believe that this assignment will serve me a foundation to build on my understanding of this library. Thanks to the Cloud credits and the access to the TUDelft Clusters, I have developed an understanding of using cloud platforms and Clusters to work with large datasets for the first time. I have made the appropriate changes to the author’s original run_benchmark file to keep it in line with the latest the pytorch-lightning version. This version of the codebase can be found here: 

https://github.com/BadABrownie/MyRain

I have added the appropriate lines of code on both collab and cluster’s linux terminal to retrieve the mmap files from GCS buckets and through limited understanding of author’s was able to execute the code, though only managing to complete one training epoch before running into other issues.

### Danning Zhao
*Model building, tuning and understanding the dataset*

This is my first time participating in a paper reproduction project. At the beginning, we are divided to understand the model and dataloader first. I was responsible for reading model-related papers and building the model. I read the multiple references papers and finally figured out how the model is built and how the input data features are added together. By referring to the original author's code, I learned to use the standard model provided by pytorch.nn to build a custom model, and learned some tricks to cope with data dimensions. By building my custom model, I have a deeper understanding of LSTM and the flow of data.

The code provided by the original author is very complicated, I cannot fully understand it. What’s more, we want to get at least some results, even if we can't use all data (~50G), so I prefer to use part of the author's code to build our own code for reproduction. I read the author's collect_data.py file, figured out how to build the dataloader and convert the data dictionary into the standard input format [t, c, w, h]. But due to time constraints, I haven't finished building the training framework.

## References

[1] X. Shi, Z. Chen, H. Wang and D. Yeung, *“Convolutional lstm network: A machine learning approach for precipitation nowcasting”*, Advances in neural information processing systems, pp. 802-810, 2015.

[2] J. Zhao, F. Deng, Y. Cai and J. Chen, *“Long short-term memory - Fully connected (LSTM-FC) neural network”* Chemosphere, pp. 486-492, 2019.

[3] C. Tong, C. Schroeder de Witt, V. Zantedeschi, D. De Martini, A. Kalaitzis, M. Chantry, D. Watson-Parris and P. Bilinski, *"RainBench: Enabling Data-Driven Precipitation Forecasting on a Global Scale"*

[4] C.K. Sønderby, L. Espeholt, J. Heek, M. Dehghani, A. Oliver, T. Salimans, J. Hickey, S. Agrawal and N. Kalchbrenner,  *“Metnet: A neural weather model for precipitation forecasting”*, 2020

[5] S. Rasp, P. D. Dueben, S. Scher, J. A. Weyn, S. Mouatadid and N. Thuerey, *“WeatherBench: A Benchmark Data Set for Data-Driven Weather Forecasting”*, Journal or advances in modeling earth systems, 2020.

[6] H. v. d. Dool, “National Weather Service Climate Prediction Center,”  Available: cpc.ncep.noaa.gov/products/people/wd51hd/book/review/chapter8.pdf
