## [Simple Baseline for Visual Question Answering](http://arxiv.org/abs/1512.02167)


Code: https://github.com/metalbubble/VQAbaseline.git

```
We describe a very simple bag-of-words baseline for visual question answering.This baseline concatenates the word features from the question and CNN features from the image to predict the answer.  When evaluated on the challenging VQA dataset , it shows comparable performance to many recent approaches using recurrent neural networks.  To explore the strength and weakness of the trained model, we also provide an interactive web demo1, and open-source code2.
```

> Interesting BoW is better than LSTM ?


```
Framework of the iBOWIMG. Features from the question sentence and image are concatenated then feed into softmax to predict the answer.

Interestingly, they notice that in one of the earliest VQA papers, the simple baseline Bag-of-words + image feature (referred to as BOWIMG baseline) outperforms the LSTM-based models on a synthesized visual QA dataset built up on top of the image captions of COCO dataset.  For the recent much larger COCO VQA dataset, the BOWIMG baseline performs worse than the LSTM-based models.
```

<p align="center"><img src="https://dl.dropboxusercontent.com/s/mjiarn9pcg2jvmp/Screenshot%20from%202016-05-25%2020%3A45%3A45.png?dl=0" width="500" ></p>


```
In most of the recent proposed models, visual QA is simplified to a classification task: 

the number of the different answers in the training set is the number of the final classes the models need to learn to predict.  

The general pipeline of those models is that the word feature extracted from the question sentence is concatenated with the visual feature extracted from the image, then they are fed into a softmax layer to predict the answer class.  
```



```
The visual feature is usually taken from the top of the VGG network or GoogLeNet, while the word features of the question sentence are usually the popular LSTM-based features.

In our iBOWIMG model, we simply use naive bag-of-words as the text feature, and use the deep features from GoogLeNet as the visual features. Figure 1 shows the framework of the iBOWIMG model, which can be implemented in Torch with no more than 10 lines of code. The input question is first converted to a one-hot vector, which is transformed to word feature via a word embedding layer and then is concatenated with the image feature from CNN. The combined feature is sent to the softmax layer to predict the answer class, which essentially is a multi-class logistic regression model.
```


<p align="center"><img src="https://camo.githubusercontent.com/09160b2cc33cc4ac974780c8a6fa90acecc5e2ca/687474703a2f2f76697375616c71612e637361696c2e6d69742e6564752f6578616d706c652e6a7067" width="500" ></p>
