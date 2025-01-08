## Learning Multi-Scene Absolute Pose Regression with Transformers
The simple and (I hope) short review for this [**paper**](https://arxiv.org/abs/2103.11468)
![MS-APR](images/ms-aprs.jpg)

## Problem/Objective
- Traditional APR requires separate models for different scenes
- Training separate models requires more time and memory
- Make a model that can handle multiple scenes and train them in a parallel manner

## Contribution/Key Idea
- Leveraging Transformers for multi-scene absolute pose regression
- Demonstrated that self-attention allows aggregation of positional and rotational image cues

## What's the difference between APR and MS-APR?
Absolute Pose Regression (APR):
- Designed to only handle one scene at a time. For example, if you have a house, the APR can only handle a living room, a dining room, or a bedroom at one time.
  We can't have a whole house part at one time.
- It commonly employs a CNN backbone that processes the input image to output a global feature vector. We pass this global feature vector to the MLP head to regress the camera's pose 

Multi-scene Absolute Pose Regression (MS-APR):
- Designed to use one model to understand and predict the camera pose across various scenes.
- Like the APR, it uses CNN to extract global feature vectors. The head slightly differs from APR, there are various techniques to estimate the camera pose from the various scenes.
  It can use the MLP or transformer-based.

## Framework
The proposed framework is shown in the image above. For simplicity, we will divide the explanation into stages. 
![MS-APR Framework](images/framework.jpg)

**Feature Backbone**
- Pass the image into the CNN backbone. This step extracts the features of the images.
- There will be 2 results of the CNN. The activation map for position and orientation, A<sub>x</sub> and A<sub>q</sub>.
  The A<sub>x</sub> and A<sub>q</sub> have different resolutions (see Figure 2a).
- Before passing to the transformer, we need to prepare the activation map to match the transformer's input requirements. How?
  - Applied a 1&times;1 convolution to reduce dimensionality, C<sub>a</sub> -> C<sub>d</sub>
  - Flatten the 3D tensor to 2D: H<sub>a</sub> &middot; W<sub>a</sub> &times; C<sub>a</sub> -> H<sub>a</sub> &times; W<sub>a</sub> &times; C<sub>a</sub>
  - Here is the problem: when we flatten the tensor, there is a probability we can lose the spatial relationship. Thus, **positional encoding** is added to the sequence.
  - Make the positional embedding (X and Y):
    - Consider W<sub>a</sub> as the X position with j-th grid; H<sub>a</sub> as the Y position with i-th grid
    - The set of positions in X will be called E<sub>u</sub>, with the dimensionality C<sub>a</sub>/2
    - The set of positions in Y will be called E<sub>v</sub>, with the dimensionality C<sub>a</sub>/2
    - If we concatenate E<sub>u</sub> and E<sub>v</sub>, we can obtain the position data for every grid <i>E</i><sub>i,j</sub><sup>pos</sup> = [<i>E</i><sub>j</sub><sup>u</sup>; <i>E</i><sub>i</sub><sup>v</sup>]. Even if we flatten it, we will know the position of each grid.

**Transformers: Encoder**
- The input to the transformer is the flattened activation map plus positional encoding -> Z = A_flatten + E.
- There are two encoders: one for position and one for orientation.
- The key components of the encoder are the Multi-Head Attention (MHA) and the MLP (Multi-Layer Perceptron).
- First, normalize the input and pass it into the MHA. The result from the MHA is then added back to the initial input to produce the final result. (Refer to Eq 3 in the paper).

  Z<sup>l'</sup> = MHA(LN(Z<sup>l-1</sup>)) + Z<sup>l-1</sup> 
- The final result of the MHA is then normalized and passed into the MLP. The result from the MLP is then added back to the initial input of this stage. (Refer to Eq 4 in the paper).

  Z<sup>l</sup> = MHA(LN(Z<sup>l'-1</sup>)) + Z<sup>l'-1</sup> 
- Finally, perform the last linear transformation. (Refer to Eq 5 in the paper).

  Z<sup>L</sup> = LN(Z<sup>l-1</sup>) 

**Transformers: Decoder**
- Please remember that **we already know** how many scenes are in the dataset. We will use this as the index information in the decoder process.
- We take the result from the encoder (both position and orientation) to another attention network (similar to Eq 3 and 4).
- The result of the decoder, S<sup>L</sup>, for both position and orientation, will be concatenated later and passed into an FC (Fully Connected) layer to select the detected scenes.
- The detected scene (index) is then chosen.

**Position and Orientation Estimation**
- The selected index is communicated to the result of the transformer for both position and orientation.
- The result from the transformer decoder, corresponding to the selected index, is then regressed using an MLP to obtain the predicted pose.

## Multi-Scene Camera Pose Loss
- Position Loss, Euclidean distance -> <i>L<sub>x</sub> = ||x<sub>0</sub> - x||<sup>2</sup></i>
- Orientation Loss -> <i>L<sub>q</sub> = ||(q<sub>0</sub> - q) / ||q|| ||<sup>2</sup></i>
- Combined Camera Pose Loss (Lp) -> <i>L<sub>p</sub> = L<sub>x</sub> exp(-s<sub>x</sub>) + s<sub>x</sub> + L<sub>q</sub> exp(-s<sub>q</sub>) + s<sub>q</sub></i> 
- Overall Multi-Scene Loss -> <i>L<sub>multi-scene</sub> = L<sub>p</sub> + NLL(s, s<sub>0</sub>)</i>

## Conclusion
This multi-scene pose estimation using transformers appears to be a legitimate approach. It would be highly effective if we could apply it to a very large dataset.
