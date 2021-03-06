# DNNs For ASR
The work is part of a Google Summer of Code project, the goal of which was to integrate DNNs with CMUSphinx. This particular repository contains some convenient scripts that wrap Keras code and allow for easy training of DNNs.
## Getting Started
Start by cloning the repository.
### Prerequisites
The required python libraries available from pypi are in the requirements.txt file. Install them by running:
```
pip install -r requirements.txt
```
Additional libraries not available from pypi:
- tfrbm- for DBN-DNN pretraining.
	- available at https://github.com/meownoid/tensorfow-rbm
## Getting Started
Since the project is primarily intended to be used with PocketSphinx the file formats for feature files, state-segmentation output files and the prediction files are in sphinx format. 
### Feature File Format
```
N: number of frames
M: dimensions of the feature vector
N*M (4 bytes)
Frame 1: f_1...f_M (4*M bytes)
.
.
.
Frame N: f_1,...,f_M (4*M bytes)
```
Look at readMFC in utils.py
### state-segmentation files
format for each frame:
```
 2    2   2    1   4  bytes
st1 [st2 st3] pos scr
```
### Prediction output
format for each frame:
```
N: number of states
N (2 bytes)
scr_1...scr_N (2*N bytes)
```
### Wrapper Scripts 
```
runDatasetGen.py -train_fileids -val_fileids [-test_fileids] -n_filts -feat_dir -feat_ext -stseg_dir -stseg_ext -mdef [-outfile_prefix] [-keep_utts]
```
runDatasetGen takes feature files and state-segmentation files stored in sphinx format along with the definition file of the GMM-HMM model to generate a set of numpy arrays that form a python readable dataset.
runDatasetGen writes the following files in the directory it was called in:
- Data Files
	- <outfile_prefix>_train.npy
	- <outfile_prefix>_dev.npy
	- <outfile_prefix>_test.npy
- label files
	- <outfile_prefix>_train_label.npy
	- <outfile_prefix>_dev_label.npy
	- <outfile_prefix>_test_label.npy
- metadata file
	- <outfile_prefix>_meta.npz

The metadata file is a zipped collection of arrays with the follwing keys:
- File names for utterances
	- filenames_Train
	- filenames_Dev
	- filenames_Test
- Number of frames per utterance (useful if -keep_utts is not set)
	- framePos_Train
	- framePos_Dev
	- framePos_Test
- State Frequencies (useful for scaling in some cases)
	- state_freq_Train
	- state_freq_Dev
	- state_freq_Test
```
runNNTrain.py -train_data -train_labels -val_data -val_labels -nn_config [-context_win] [-cuda_device_id] [-pretrain] [-keras_model] -model_name
```
runNNTrain takes the training and validation data files (as generated by runDatasetGen) and trains a neural network on them. 
The architecture and parameters of the neural network is defined in a text file. Currently this script supports 4 network types:
- MLP (mlp)
- Convolutional Neural Network (conv)
- MLP with short cut connections (resnet)
- Convolutional Network with residual connections in the fully connected layers (conv + resnet)
See sample_nn.cfg for an example.
The format for the configuration file consists of ```param``` and ```value``` pairs
if value has multiple elemets (represented by ... below) they should be separated by spaces.
Params and possible values:
- **type** 			mlp, conv, resnet, conv+resnet
- **width** 		any integer value
- **depth**			any integer value
- **dropout**		float in (0,1)
- **batch_norm**	-
- **activation**	sigmoid, hard_sigmoid, elu, relu, selu, tanh, softplus, softsign, softmax, linear
- **optimizer**		sgd, adam, adagrad
- **lr** 			float in (0,1)
- **batch_size** 	any integer value
- **ctc_loss**		-
- for type = conv and type = conv+resnet
	- **conv** 			[n_filters, filter_window]...
	- **pooling**		None, [max/avg, window_size, stride_size]
- for type = resnet and type = conv+resnet
	- **block_depth**	any integer value
	- **n_blocks**		any integer value
```
runNNPredict -keras_model -ctldir -inext -outdir -outext -nfilts [-acoustic_weight] [-context_win] [-cuda_device_id]
```
runNNPredict takes a keras model and a list of feature files to generate predictions. The predictions are stored as binary files in sphinx readable format (defined above).
Please ensure that the dimensionality of the feature vectors matches nfilts and the context window is the same as that for which the model was trained.
The acoustic_weight is used to scale the output scores. This is required because if the scores are passed through a GMM-GMM decoder like PocketSphinx are too small or too large then the decoding performance suffers. One way of estimating this weight is to generate scores from the GMM-HMM decoder being used, fit a linear regression between the GMM-HMM scores and the NN-scores and use the coefficient as the weight.
```
readSen.py -gmm_score_dir -gmm_ctllist -nn_score_dir -nn_ctllist [-gmm_ext] [-nn_ext]
```
readSen takes scores (stored in sphinx readable binary files) obtained from a GMM-HMM decoder and a NN, and fit a regression to them.

## Example workflow with CMUSphinx
- Feature extraction using sphinx_fe:
	 ```
	 sphinx_fe -argfile ../../en_us.ci_cont/feat.params -c etc/wsj0_train.fileids -di wav/ -do feat_ci_mls -mswav yes -eo mls -ei wav -ofmt sphinx -logspec yes
	 ```
- State-segmentation using sphinx3_align
	```
	sphinx3_align -hmm ../../en_us.ci_cont/ -dict etc/cmudict.0.6d.wsj0 -ctl etc/wsj0_train.fileids -cepdir feat_ci_mls/ -cepext .mfc -insent etc/wsj0.transcription -outsent wsj0.out -stsegdir stateseg_ci_dir/ -cmn batch
	```
- Generate dataset using runDatasetGen.py
- Train NN using runNNtrain.py
- Generate predictions from the NN using runNNPredct.py
- Generate predictions from PocketSphinx
```
pocketsphinx_batch -hmm ../../en_us.ci_cont/ -lm ../../tcb20onp.Z.DMP -cepdir feat_ci_mfc/ -ctl ../../GSOC/SI_ET_20.NDX -dict etc/cmudict.0.6d.wsj0 -senlogdir sendump_ci/ -compallsen yes -bestpath no -fwdflat no -remove_noise no -remove_silence no -logbase 1.0001 -pl_window 0
```
- Compute the acoustic weight using readSen.py
- Decode the scaled NN predictions with PocketSphinx
```
pocketsphinx_batch -hmm ../../en_us.ci_cont/ -lm ../../tcb20onp.Z.DMP -cepdir senscores/ -cepext .sen -hyp NN2.hyp -ctl ../../GSOC/SI_ET_20.NDX -dict etc/cmudict.0.6d.wsj0 -compallsen yes -logbase 1.0001 -pl_window 0 -senin yes
```




