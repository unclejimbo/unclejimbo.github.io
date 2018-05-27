---
title: 'Implementing MVCNN with Caffe, Part I'
date: 2017-1-5
categories:
- Computer Vision
tags:
- Computer Graphics
- Computer Vision
- Deep Learning
---

# Introduction

This post is about a paper published in ICCV2015, called ["Multi-view Convolutional Neural Networks for 3D Shape Recognition"](http://www.cv-foundation.org/openaccess/content_iccv_2015/papers/Su_Multi-View_Convolutional_Neural_ICCV_2015_paper.pdf). It describes a method to classify 3d shape models using 2d image classification networks. While the authors have open-sourced their matlab implementation on [GitHub](https://github.com/suhangpro/mvcnn), here I'll try to implement this network with Caffe.

In the first half of this two-part blog, I'll quickly explain the core idea of this paper. After that I'll try to implement a naive version of this network. While in part II I'll go through the details on implementing MVCNN as the paper describes.

The complete codes and scripts in this blog can be found at my [GitHub repo](https://github.com/unclejimbo/mvcnn-caffe).

![](https://camo.githubusercontent.com/f505454fa4d971db8b85b35ad7cac63795d3eaa0/687474703a2f2f7669732d7777772e63732e756d6173732e6564752f6d76636e6e2f696d616765732f6d76636e6e2e706e67)
(*The network architecture of MVCNN*)

<!--- more --->

# The Power of Multi-View

Traditional 3d shape recognition algorithms are generally based on heuristic descriptors such as Spherical Harmonics. More recent advances like ShapeNets tried to voxelize the model and train a deep neural network. On the other hand, MVCNN tries to leverage the power of image classification CNNs, because public image datasets such as ImageNet is much larger than 3d model datasets and state-of-the-art networks on ilsvrc have achieved pretty high precision on classification tasks.

So how about rendering a 3d shape model under different viewpoints and training a 2d CNN with rendered images? Then you can input the rendered views of an unknown model and try to decide its category. This is exactly what we'll implement in this post. We'll use multiple rendered images as input and simply do a majority vote to decide the final label for the model.

However, the MVCNN is a bit more complicated in the way it combines multi-view representations. The authors train each view with a different network and introduce a view-pooling layer to combine multiple networks into one, as can be seen on the figure above. View-pooling at its core is simply max-pooling, extracting the largest value at each pixel among all views. More on this topic next time, but let's first go ahead and implement the simple one :).

# Training MVCNN Directly without View Pooling

## Prerequisites

- Caffe :white_check_mark:
- Python2.7(Anaconda2) :white_check_mark:
- Scikit-Image :white_check_mark:

## Data Preparation

You can download the rendered images in their [repository](https://github.com/suhangpro/mvcnn). Here I'll use the modelnet40v2 dataset. On the other hand, you can directly download the models from [Princeton ModelNet](http://modelnet.cs.princeton.edu/#) and render by yourself if you want better rendering qualities. Although the authors have argued that rendering method is irrelevant with classification precisions.

Please follow the steps described [here](https://github.com/unclejimbo/mvcnn-caffe/tree/master/modelnet40v2) to preprocess the dataset. Basically all it does is to first pad all images into 256x256, and split a validation set out of training set with ratio 1:9. Then we prepare label text files and compile images into leveldb files to feed into the network. You can sample the inputs if you think the full dataset is too large. I use leveldb rather than lmdb because lmdb seems to have bug with large amount data. If you're not familiar with this procedure, please check out this [tutorial](http://caffe.berkeleyvision.org/gathered/examples/imagenet.html).

## Network Setup

Here we'll try to fine-tune the bvlc-reference-caffenet that caffe provides. The method is pretty much the same as the official [tutorial on fine-tuning flickr style data](http://caffe.berkeleyvision.org/gathered/examples/finetune_flickr_style.html). There are a few things to note. First, because we are using the 40 class version of modelnet so the output of this network should be 40. And I've fixed the learning rate for all conv layers. Moreover, the latest fc layer's learning rate is set 5 times higher. Check the prototxt in my [repo](https://github.com/unclejimbo/mvcnn-caffe) for details.

## Go Ahead and Training

```
caffe train -solver ./mvcnn_caffenet_simple/solver.prototxt -weights your_caffe_root/models/bvlc_reference_caffenet/bvlc_reference_caffenet.caffemodel -gpu 0
```

I've tried several parameter setups and the best could give around 72.5% test accuracy. Go play with the parameters by yourself.

## Use Majority Vote to Classify a 3D Model

We'll now use 80 views as input and take the majority of the predictions as label for this model. The file [classify_model_simple.py](https://github.com/unclejimbo/mvcnn-caffe/blob/master/classify_model_simple.py) contains the source code for this method:
```
import caffe
import os
import sys
import re
import numpy as np

n_views = 80 # change this if you are not using 80 views per model"
n_classes = 40 # change this if you are not using 40 class dataset
model = "./mvcnn_caffenet_simple/deploy.prototxt" # change this to your model defiintion
weights = "./mvcnn_caffenet_simple/caffenet_train_iter_15000.caffemodel" # change this to your trained model
mean_file = "./modelnet40v2/mean.binaryproto" # change this to your mean file
label_name_file = "./modelnet40v2/label_name.txt" # change this to your label_name file

def predict(images, net, transformer):
    if (len(images) != n_views):
        sys.exit("Error: expecting ", n_views, " images in a batch")

    votes = np.zeros(n_views)
    for img in images:
        net.blobs['data'].data[...] = transformer.preprocess('data', img)
        net.forward()
        scores = net.blobs['prob'].data[0].flatten()
        prediction = np.argmax(scores)
        votes[prediction] += 1
    return np.argmax(votes)

if __name__ == '__main__':
    if (len(sys.argv) != 2):
        sys.exit("Usage: python classify_model_simple.py /path/to/image_directory/")

    files = os.listdir(sys.argv[1])
    if (len(files) % n_views != 0):
        sys.exit("Error: num of input images should be divisible by " + str(n_views))

    # Import label-name correspondence
    label_name = {}
    with open(label_name_file) as f:
        for line in f:
            (label, name) = line.split()
            label_name[int(label)] = name

    # Net
    net = caffe.Net(model, weights, caffe.TEST)

    # Convert image mean
    mean_blob = caffe.proto.caffe_pb2.BlobProto()
    mean_bin = open(mean_file, 'rb').read()
    mean_blob.ParseFromString(mean_bin)
    mean = np.array(caffe.io.blobproto_to_array(mean_blob))
    # have to manually slice mean size because caffe assume mean size to be the same as input dims,
    # during training both mean and image sizes could automatically get cropped to network input dims,
    # however when testing only image size could be cropped automatically by deployment input layer
    mean = mean[0, :, 14:-15, 14:-15]

    # Transform data
    transformer = caffe.io.Transformer({'data': net.blobs['data'].data.shape})
    transformer.set_transpose('data', (2, 0, 1))
    transformer.set_mean('data', mean)
    transformer.set_raw_scale('data', 255)
    transformer.set_channel_swap('data', (2, 1, 0))

    # Load images and predict
    count = 0;
    right = 0;
    for i in range(len(files) / n_views):
        images = []
        for j in range(i*n_views, (i+1)*n_views):
            img = caffe.io.load_image(os.path.join(sys.argv[1], files[j]))
            images.append(img)

        # Extract label from file name
        pos = re.search("\d", files[j])
        name = files[j][:pos.start()-1]
        pred_label = predict(images, net, transformer)
        pred_name = label_name[int(pred_label)]
        print("label: " + name + ", " + "prediction: " + pred_name)
        count += 1
        if name == pred_name:
            right += 1

    print("Accuracy: " + str(right / float(count)))
```
The output accuracy is about 78.3%, which is higher than one image classification output.

# Conclusion

Great, this simple network does what we've expected, although the result is much more inferior to the result achieved in the paper. But this isn't a surprise because it is image loss rather than model loss that is minimized during training. This also explains the high training error because some views are quite different from others so this adds a lot of noise in our data.

I'll implement the full mvcnn with view-pooling in the next post, and see if it works better. Stay tuned.

# Reference

Su H, Maji S, Kalogerakis E, et al. Multi-view convolutional neural networks for 3d shape recognition[C]//Proceedings of the IEEE International Conference on Computer Vision. 2015: 945-953.

[http://seiya-kumada.blogspot.jp/2015/10/3d-object-recognition-by-caffe.html](http://seiya-kumada.blogspot.jp/2015/10/3d-object-recognition-by-caffe.html)
