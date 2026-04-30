
# Learning Transferable Visual Models From Natural Language Supervision


Alec Radford *1 Jong Wook Kim *1 Chris Hallacy 1 Aditya Ramesh 1 Gabriel Goh 1 Sandhini Agarwal 1 Girish Sastry 1 Amanda Askell 1 Pamela Mishkin 1 Jack Clark 1 Gretchen Krueger 1 Ilya Sutskever 1


# Abstract


State-of-the-art computer vision systems are trained to predict a fixed set of predetermined object categories. This restricted form of supervision limits their generality and usability since additional labeled data is needed to specify any other visual concept. Learning directly from raw text about images is a promising alternative which leverages a much broader source of supervision. We demonstrate that the simple pre-training task of predicting which caption goes with which image is an efficient and scalable way to learn SOTA image representations from scratch on a dataset of 400 million (image, text) pairs collected from the internet. After pre-training, natural language is used to reference learned visual concepts (or describe new ones) enabling zero-shot transfer of the model to downstream tasks. We study the performance of this approach by benchmarking on over 30 different existing computer vision datasets, spanning tasks such as OCR, action recognition in videos, geo-localization, and many types of fine-grained object classification. The model transfers non-trivially to most tasks and is often competitive with a fully supervised baseline without the need for any dataset specific training. For instance, we match the accuracy of the original ResNet-50 on ImageNet zero-shot without needing to use any of the 1.28 million training examples it was trained on. We release our code and pre-trained model weights at https://github.com/OpenAI/CLIP.


# 1. Introduction and Motivating Work


Pre-training methods which learn directly from raw text have revolutionized NLP over the last few years (Dai & Le, 2015; Peters et al., 2018; Howard & Ruder, 2018; Radford et al., 2018; Devlin et al., 2018; Raffel et al., 2019).


*Equal contribution $ {}^{1} $ OpenAI, San Francisco, CA 94110, USA. Correspondence to: <\{alec,jongwook\}@openai.com>.


Task-agnostic objectives such as autoregressive and masked language modeling have scaled across many orders of magnitude in compute, model capacity, and data, steadily improving capabilities. The development of "text-to-text" as a standardized input-output interface (McCann et al., 2018; Radford et al., 2019; Raffel et al., 2019) has enabled task-agnostic architectures to zero-shot transfer to downstream datasets removing the need for specialized output heads or dataset specific customization. Flagship systems like GPT-3 (Brown et al., 2020) are now competitive across many tasks with bespoke models while requiring little to no dataset specific training data.


These results suggest that the aggregate supervision accessible to modern pre-training methods within web-scale collections of text surpasses that of high-quality crowd-labeled NLP datasets. However, in other fields such as computer vision it is still standard practice to pre-train models on crowd-labeled datasets such as ImageNet (Deng et al., 2009). Could scalable pre-training methods which learn directly from web text result in a similar breakthrough in computer vision? Prior work is encouraging.


Over 20 years ago Mori et al. (1999) explored improving content based image retrieval by training a model to predict the nouns and adjectives in text documents paired with images. Quattoni et al. (2007) demonstrated it was possible to learn more data efficient image representations via manifold learning in the weight space of classifiers trained to predict words in captions associated with images. Srivastava & Salakhutdinov (2012) explored deep representation learning by training multimodal Deep Boltzmann Machines on top of low-level image and text tag features. Joulin et al. (2016) modernized this line of work and demonstrated that CNNs trained to predict words in image captions learn useful image representations. They converted the title, description, and hashtag metadata of images in the YFCC100M dataset (Thomee et al., 2016) into a bag-of-words multi-label classification task and showed that pretraining AlexNet (Krizhevsky et al., 2012) to predict these labels learned representations which preformed similarly to ImageNet-based pre-training on transfer tasks. Li et al. (2017) then extended this approach to predicting phrase n-grams in addition to individual words and demonstrated the ability of their system to zero-shot transfer to other image


---


![图片 page2-2](images/page002_img01.png)


![图片 page2-3](images/page002_img02.png)


_Figure 1. Summary of our approach. While standard image models jointly train an image feature extractor and a linear classifier to predict some label, CLIP jointly trains an image encoder and a text encoder to predict the correct pairings of a batch of (image, text) training examples. At test time the learned text encoder synthesizes a zero-shot linear classifier by embedding the names or descriptions of the target dataset's classes._


classification datasets by scoring target classes based on their dictionary of learned visual n-grams and predicting the one with the highest score. Adopting more recent architectures and pre-training approaches, VirTex (Desai & Johnson, 2020), ICMLM (Bulent Sariyildiz et al., 2020), and ConvIRT (Zhang et al., 2020) have recently demonstrated the potential of transformer-based language modeling, masked language modeling, and contrastive objectives to learn image representations from text.


While exciting as proofs of concept, using natural language supervision for image representation learning is still rare. This is likely because demonstrated performance on common benchmarks is much lower than alternative approaches. For example, Li et al. (2017) reach only $11.5\%$ accuracy on ImageNet in a zero-shot setting. This is well below the $88.4\%$ accuracy of the current state of the art (Xie et al., 2020). It is even below the $50\%$ accuracy of classic computer vision approaches (Deng et al., 2012). Instead, more narrowly scoped but well-targeted uses of weak supervision have improved performance. Mahajan et al. (2018) showed that predicting ImageNet-related hashtags on Instagram images is an effective pre-training task. When fine-tuned to ImageNet these pre-trained models increased accuracy by over $5\%$ and improved the overall state of the art at the time. Kolesnikov et al. (2019) and Dosovitskiy et al. (2020) have also demonstrated large gains on a broader set of transfer benchmarks by pre-training models to predict the classes of the noisily labeled JFT-300M dataset.


This line of work represents the current pragmatic middle ground between learning from a limited amount of supervised "gold-labels" and learning from practically unlimited amounts of raw text. However, it is not without pro


mises. Both works carefully design, and in the process limit, their supervision to 1000 and 18291 classes respectively. Natural language is able to express, and therefore supervise, a much wider set of visual concepts through its generality. Both approaches also use static softmax classifiers to perform prediction and lack a mechanism for dynamic outputs. This severely curtails their flexibility and limits their "zero-shot" capabilities.


A crucial difference between these weakly supervised models and recent explorations of learning image representations directly from natural language is scale. While Mahajan et al. (2018) and Kolesnikov et al. (2019) trained their models for accelerator years on millions to billions of images, VirTex, ICMLM, and ConVIRT trained for accelerator days on one to two hundred thousand images. In this work, we close this gap and study the behaviors of image classifiers trained with natural language supervision at large scale. Enabled by the large amounts of publicly available data of this form on the internet, we create a new dataset of 400 million (image, text) pairs and demonstrate that a simplified version of ConVIRT trained from scratch, which we call CLIP, for Contrastive Language-Image Pre-training, is an efficient method of learning from natural language supervision. We study the scalability of CLIP by training a series of eight models spanning almost 2 orders of magnitude of compute and observe that transfer performance is a smoothly predictable function of compute (Hestness et al., 2017; Kaplan et al., 2020). We find that CLIP, similar to the GPT family, learns to perform a wide set of tasks during pre-training including OCR, geo-localization, action recognition, and many others. We measure this by benchmarking the zero-shot transfer performance of CLIP on over 30 existing datasets and find


---


![图片 page3-2](images/page003_img01.png)


_Figure 2. CLIP is much more efficient at zero-shot transfer than our image caption baseline. Although highly expressive, we found that transformer-based language models are relatively weak at zero-shot ImageNet classification. Here, we see that it learns 3x slower than a baseline which predicts a bag-of-words (BoW) encoding of the text (Joulin et al., 2016). Swapping the prediction objective for the contrastive objective of CLIP further improves efficiency another 4x._


it can be competitive with prior task-specific supervised models. We also confirm these findings with linear-probe representation learning analysis and show that CLIP outperforms the best publicly available ImageNet model while also being more computationally efficient. We additionally find that zero-shot CLIP models are much more robust than equivalent accuracy supervised ImageNet models which suggests that zero-shot evaluation of task-agnostic models is much more representative of a model's capability. These results have significant policy and ethical implications, which we consider in Section 7.


# 2. Approach


# 2.1. Natural Language Supervision


At the core of our approach is the idea of learning perception from supervision contained in natural language. As discussed in the introduction, this is not at all a new idea, however terminology used to describe work in this space is varied, even seemingly contradictory, and stated motivations are diverse. Zhang et al. (2020), Gomez et al. (2017), Joulin et al. (2016), and Desai & Johnson (2020) all introduce methods which learn visual representations from text paired with images but describe their approaches as unsupervised, self-supervised, weakly supervised, and supervised respectively.


We emphasize that what is common across this line of work is not any of the details of the particular methods used but the appreciation of natural language as a training signal. All these approaches are learning from natural language super-


vision. Although early work wrestled with the complexity of natural language when using topic model and n-gram representations, improvements in deep contextual representation learning suggest we now have the tools to effectively leverage this abundant source of supervision (McCann et al., 2017).


Learning from natural language has several potential strengths over other training methods. It's much easier to scale natural language supervision compared to standard crowd-sourced labeling for image classification since it does not require annotations to be in a classic "machine learning compatible format" such as the canonical 1-of-N majority vote "gold label". Instead, methods which work on natural language can learn passively from the supervision contained in the vast amount of text on the internet. Learning from natural language also has an important advantage over most unsupervised or self-supervised learning approaches in that it doesn't "just" learn a representation but also connects that representation to language which enables flexible zero-shot transfer. In the following subsections, we detail the specific approach we settled on.


# 2.2. Creating a Sufficiently Large Dataset


Existing work has mainly used three datasets, MS-COCO (Lin et al., 2014), Visual Genome (Krishna et al., 2017), and YFCC100M (Thomee et al., 2016). While MS-COCO and Visual Genome are high quality crowd-labeled datasets, they are small by modern standards with approximately 100,000 training photos each. By comparison, other computer vision systems are trained on up to 3.5 billion Instagram photos (Mahajan et al., 2018). YFCC100M, at 100 million photos, is a possible alternative, but the metadata for each image is sparse and of varying quality. Many images use automatically generated filenames like 20160716_113957.JPG as "titles" or contain "descriptions" of camera exposure settings. After filtering to keep only images with natural language titles and/or descriptions in English, the dataset shrunk by a factor of 6 to only 15 million photos. This is approximately the same size as ImageNet.


A major motivation for natural language supervision is the large quantities of data of this form available publicly on the internet. Since existing datasets do not adequately reflect this possibility, considering results only on them would underestimate the potential of this line of research. To address this, we constructed a new dataset of 400 million (image, text) pairs collected from a variety of publicly available sources on the Internet. To attempt to cover as broad a set of visual concepts as possible, we search for (image, text) pairs as part of the construction process whose text includes one of a set of 500,000 queries.<sup>1</sup> We approximately class


<sup>1</sup>The base query list is all words occurring at least 100 times in the English version of Wikipedia. This is augmented with bi-grams


---


balance the results by including up to 20,000 (image, text) pairs per query. The resulting dataset has a similar total word count as the WebText dataset used to train GPT-2. We refer to this dataset as WIT for WebImageText.


# 2.3. Selecting an Efficient Pre-Training Method


State-of-the-art computer vision systems use very large amounts of compute. Mahajan et al. (2018) required 19 GPU years to train their ResNeXt101-32x48d and Xie et al. (2020) required 33 TPUv3 core-years to train their Noisy Student EfficientNet-L2. When considering that both these systems were trained to predict only 1000 ImageNet classes, the task of learning an open set of visual concepts from natural language seems daunting. In the course of our efforts, we found training efficiency was key to successfully scaling natural language supervision and we selected our final pre-training method based on this metric.


Our initial approach, similar to VirTex, jointly trained an image CNN and text transformer from scratch to predict the caption of an image. However, we encountered difficulties efficiently scaling this method. In Figure 2 we show that a 63 million parameter transformer language model, which already uses twice the compute of its ResNet-50 image encoder, learns to recognize ImageNet classes three times slower than a much simpler baseline that predicts a bag-of-words encoding of the same text.


Both these approaches share a key similarity. They try to predict the exact words of the text accompanying each image. This is a difficult task due to the wide variety of descriptions, comments, and related text that co-occur with images. Recent work in contrastive representation learning for images has found that contrastive objectives can learn better representations than their equivalent predictive objective (Tian et al., 2019). Other work has found that although generative models of images can learn high quality image representations, they require over an order of magnitude more compute than contrastive models with the same performance (Chen et al., 2020a). Noting these findings, we explored training a system to solve the potentially easier proxy task of predicting only which text as a whole is paired with which image and not the exact words of that text. Starting with the same bag-of-words encoding baseline, we swapped the predictive objective for a contrastive objective in Figure 2 and observed a further $4\mathrm{x}$ efficiency improvement in the rate of zero-shot transfer to ImageNet.


Given a batch of $ N $ (image, text) pairs, CLIP is trained to predict which of the $ N \times N $ possible (image, text) pairings across a batch actually occurred. To do this, CLIP learns a


with high pointwise mutual information as well as the names of all Wikipedia articles above a certain search volume. Finally all WordNet synsets not already in the query list are added.


multi-modal embedding space by jointly training an image encoder and text encoder to maximize the cosine similarity of the image and text embeddings of the $N$ real pairs in the batch while minimizing the cosine similarity of the embeddings of the $N^2 - N$ incorrect pairings. We optimize a symmetric cross entropy loss over these similarity scores. In Figure 3 we include pseudocode of the core of an implementation of CLIP. To our knowledge this batch construction technique and objective was first introduced in the area of deep metric learning as the multi-class $N$-pair loss Sohn (2016), was popularized for contrastive representation learning by Oord et al. (2018) as the InfoNCE loss, and was recently adapted for contrastive (text, image) representation learning in the domain of medical imaging by Zhang et al. (2020).


Due to the large size of our pre-training dataset, over-fitting is not a major concern and the details of training CLIP are simplified compared to the implementation of Zhang et al. (2020). We train CLIP from scratch without initializing the image encoder with ImageNet weights or the text encoder with pre-trained weights. We do not use the non-linear projection between the representation and the contrastive embedding space, a change which was introduced by Bachman et al. (2019) and popularized by Chen et al. (2020b). We instead use only a linear projection to map from each encoder's representation to the multi-modal embedding space. We did not notice a difference in training efficiency between the two versions and speculate that non-linear projections may be co-adapted with details of current image only in self-supervised representation learning methods. We also remove the text transformation function $ t_{u} $ from Zhang et al. (2020) which samples a single sentence at uniform from the text since many of the (image, text) pairs in CLIP's pretraining dataset are only a single sentence. We also simplify the image transformation function $ t_{v} $. A random square crop from resized images is the only data augmentation used during training. Finally, the temperature parameter which controls the range of the logits in the softmax, $ \tau $, is directly optimized during training as a log-parameterized multiplicative scalar to avoid turning as a hyper-parameter.


# 2.4. Choosing and Scaling a Model


We consider two different architectures for the image encoder. For the first, we use ResNet-50 (He et al., 2016a) as the base architecture for the image encoder due to its widespread adoption and proven performance. We make several modifications to the original version using the ResNet-D improvements from He et al. (2019) and the antialiased rect-2 blur pooling from Zhang (2019). We also replace the global average pooling layer with an attention pooling mechanism. The attention pooling is implemented as a single layer of "transformer-style" multi-head QKV attention where the query is conditioned on the global average-pooled


---


```
# image Encoder - ResNet or Vision Transformer
# text Encoder - CBOW or Text Transformer
# I[n, h, w, c] - minibatch of aligned images
# T[n, l] - minibatch of aligned texts
# W_i[d_i, d_e] - learned proj of image to embed
# W_t[d_t, d_e] - learned proj of text to embed
# t - learned temperature parameter
```


```
extract feature representations of each modality I_f = imageEncoder(I) #[n, d_i] T_f = textEncoder(T) #[n, d_t]
```


```
```cpp
# joint multimodal embedding [n, d_e]
I_e = l2_normalize(np.dot(I_f, W_i), axis=1)
T_e = l2_normalize(np.dot(T_f, W_t), axis=1)
```


```
scaled pairwise cosine similarities [n, n]  
logits = np.dot(I_e, T_e.T) * np.exp(t)
```


```
symmetric loss function
labels = np.arange(n)
loss_i = cross_entropy_loss(logits, labels, axis \coloneqq 0
loss_t = cross_entropy_loss(logits, labels, axis \coloneqq 1
loss = (loss_i + loss_t)/2
```


_Figure 3. Numpy-like pseudocode for the core of an implementation of CLIP._


representation of the image. For the second architecture, we experiment with the recently introduced Vision Transformer (ViT) (Dosovitskiy et al., 2020). We closely follow their implementation with only the minor modification of adding an additional layer normalization to the combined patch and position embeddings before the transformer and use a slightly different initialization scheme.


The text encoder is a Transformer (Vaswani et al., 2017) with the architecture modifications described in Radford et al. (2019). As a base size we use a 63M-parameter 12-layer 512-wide model with 8 attention heads. The transformer operates on a lower-cased byte pair encoding (BPE) representation of the text with a 49,152 vocab size (Sennrich et al., 2015). For computational efficiency, the max sequence length was capped at 76. The text sequence is bracketed with [SOS] and [EOS] tokens and the activations of the highest layer of the transformer at the [EOS] token are treated as the feature representation of the text which is layer normalized and then linearly projected into the multi-modal embedding space. Masked self-attention was used in the text encoder to preserve the ability to initialize with a pre-trained language model or add language modeling as an auxiliary objective, though exploration of this is left as future work.


While previous computer vision research has often scaled models by increasing the width (Mahajan et al., 2018) or depth (He et al., 2016a) in isolation, for the ResNet image encoders we adapt the approach of Tan & Le (2019) which found that allocating additional compute across all of width, depth, and resolution outperforms only allocating it to only


one dimension of the model. While Tan & Le (2019) tune the ratio of compute allocated to each dimension for their EfficientNet architecture, we use a simple baseline of allocating additional compute equally to increasing the width, depth, and resolution of the model. For the text encoder, we only scale the width of the model to be proportional to the calculated increase in width of the ResNet and do not scale the depth at all, as we found CLIP's performance to be less sensitive to the capacity of the text encoder.


# 2.5. Training


We train a series of 5 ResNets and 3 Vision Transformers. For the ResNets we train a ResNet-50, a ResNet-101, and then 3 more which follow EfficientNet-style model scaling and use approximately 4x, 16x, and 64x the compute of a ResNet-50. They are denoted as RN50x4, RN50x16, and RN50x64 respectively. For the Vision Transformers we train a ViT-B/32, a ViT-B/16, and a ViT-L/14. We train all models for 32 epochs. We use the Adam optimizer (Kingma & Ba, 2014) with decoupled weight decay regularization (Loshchilov & Hutter, 2017) applied to all weights that are not gains or biases, and decay the learning rate using a cosine schedule (Loshchilov & Hutter, 2016). Initial hyperparameters were set using a combination of grid searches, random search, and manual tuning on the baseline ResNet-50 model when trained for 1 epoch. Hyper-parameters were then adapted heuristically for larger models due to computational constraints. The learnable temperature parameter $\tau$ was initialized to the equivalent of 0.07 from (Wu et al., 2018) and clipped to prevent scaling the logits by more than 100 which we found necessary to prevent training instability. We use a very large minibatch size of 32,768. Mixed-precision (Micikevicius et al., 2017) was used to accelerate training and save memory. To save additional memory, gradient checkpointing (Griewank & Walther, 2000; Chen et al., 2016), half-precision Adam statistics (Dhariwal et al., 2020), and half-precision stochastically rounded text encoder weights were used. The calculation of embedding similarities was also sharded with individual GPUs computing only the subset of the pairwise similarities necessary for their local batch of embeddings. The largest ResNet model, RN50x64, took 18 days to train on 592 V100 GPUs while the largest Vision Transformer took 12 days on 256 V100 GPUs. For the ViT-L/14 we also pre-train at a higher 336 pixel resolution for one additional epoch to boost performance similar to FixRes (Touvron et al., 2019). We denote this model as ViT-L/14@336px. Unless otherwise specified, all results reported in this paper as "CLIP" use this model which we found to perform best.


---


# 3. Experiments


# 3.1. Zero-Shot Transfer


# 3.1.1. MOTIVATION


In computer vision, zero-shot learning usually refers to the study of generalizing to unseen object categories in image classification (Lampert et al., 2009). We instead use the term in a broader sense and study generalization to unseen datasets. We motivate this as a proxy for performing unseen tasks, as aspired to in the zero-data learning paper of Larochelle et al. (2008). While much research in the field of unsupervised learning focuses on the representation learning capabilities of machine learning systems, we motivate studying zero-shot transfer as a way of measuring the task-learning capabilities of machine learning systems. In this view, a dataset evaluates performance on a task on a specific distribution. However, many popular computer vision datasets were created by the research community primarily as benchmarks to guide the development of generic image classification methods rather than measuring performance on a specific task. While it is reasonable to say that the SVHN dataset measures the task of street number transcription on the distribution of Google Street View photos, it is unclear what "real" task the CIFAR-10 dataset measures. It is clear, however, what distribution CIFAR-10 is drawn from - TinyImages (Torralba et al., 2008). On these kinds of datasets, zero-shot transfer is more an evaluation of CLIP's robustness to distribution shift and domain generalization rather than task generalization. Please see Section 3.3 for analysis focused on this.


To our knowledge, Visual N-Grams (Li et al., 2017) first studied zero-shot transfer to existing image classification datasets in the manner described above. It is also the only other work we are aware of that has studied zero-shot transfer to standard image classification datasets using a generically pre-trained model and serves as the best reference point for contextualizing CLIP. Their approach learns the parameters of a dictionary of 142,806 visual n-grams (spanning 1- to 5- grams) and optimizes these n-grams using a differential version of Jelinek-Mercer smoothing to maximize the probability of all text n-grams for a given image. In order to perform zero-shot transfer, they first convert the text of each of the dataset's class names into its n-gram representation and then compute its probability according to their model, predicting the one with the highest score.


Our focus on studying zero-shot transfer as an evaluation of task learning is inspired by work demonstrating task learning in the field of NLP. To our knowledge Liu et al. (2018) first identified task learning as an “unexpected side-effect” when a language model trained to generate Wikipedia articles learned to reliably transliterate names between languages. While GPT-1 (Radford et al., 2018) focused on pre


training as a transfer learning method to improve supervised fine-tuning, it also included an ablation study demonstrating that the performance of four heuristic zero-shot transfer methods improved steadily over the course of pre-training, without any supervised adaption. This analysis served as the basis for GPT-2 (Radford et al., 2019) which focused exclusively on studying the task-learning capabilities of language models via zero-shot transfer.


# 3.1.2. USING CLIP FOR ZERO-SHOT TRANSFER


CLIP is pre-trained to predict if an image and a text snippet are paired together in its dataset. To perform zero-shot classification, we reuse this capability. For each dataset, we use the names of all the classes in the dataset as the set of potential text pairings and predict the most probable (image, text) pair according to CLIP. In a bit more detail, we first compute the feature embedding of the image and the feature embedding of the set of possible texts by their respective encoders. The cosine similarity of these embeddings is then calculated, scaled by a temperature parameter $\tau$, and normalized into a probability distribution via a softmax. Note that this prediction layer is a multinomial logistic regression classifier with L2-normalized inputs, L2-normalized weights, no bias, and temperature scaling. When interpreted this way, the image encoder is the computer vision backbone which computes a feature representation for the image and the text encoder is a hypernetwork (Ha et al., 2016) which generates the weights of a linear classifier based on the text specifying the visual concepts that the classes represent. Lei Ba et al. (2015) first introduced a zero-shot image classifier of this form while the idea of generating a classifier from natural language dates back to at least Elhoseiny et al. (2013). Continuing with this interpretation, every step of CLIP pre-training can be viewed as optimizing the performance of a randomly created proxy to a computer vision dataset which contains 1 example per class and has 32,768 total classes defined via natural language descriptions. For zero-shot evaluation, we cache the zero-shot classifier once it has been computed by the text encoder and reuse it for all subsequent predictions. This allows the cost of generating it to be amortized across all the predictions in a dataset.


# 3.1.3. INITIAL COMPARISON TO VISUAL N-GRAMS


In Table 1 we compare Visual N-Grams to CLIP. The best CLIP model improves accuracy on ImageNet from a proof of concept $11.5\%$ to $76.2\%$ and matches the performance of the original ResNet-50 despite using none of the 1.28 million crowd-labeled training examples available for this dataset. Additionally, the top-5 accuracy of CLIP models are noticeably higher than their top-1, and this model has a $95\%$ top-5 accuracy, matching Inception-V4 (Szegedy et al., 2016). The ability to match the performance of a strong, fully supervised baselines in a zero-shot setting suggests


---


<table><tr><td></td><td>aYahoo</td><td>ImageNet</td><td>SUN</td></tr><tr><td>Visual N-Grams</td><td>72.4</td><td>11.5</td><td>23.0</td></tr><tr><td>CLIP</td><td>98.4</td><td>76.2</td><td>58.5</td></tr></table>


_Table 1. Comparing CLIP to prior zero-shot transfer image classification results. CLIP improves performance on all three datasets by a large amount. This improvement reflects many differences in the 4 years since the development of Visual N-Grams (Li et al., 2017)._


CLIP is a significant step towards flexible and practical zero-shot computer vision classifiers. As mentioned above, the comparison to Visual N-Grams is meant for contextualizing the performance of CLIP and should not be interpreted as a direct methods comparison between CLIP and Visual N-Grams as many performance relevant differences between the two systems were not controlled for. For instance, we train on a dataset that is $10\mathrm{x}$ larger, use a vision model that requires nearly $100\mathrm{x}$ more compute per prediction, likely used over $1000\mathrm{x}$ their training compute, and use a transformer-based model which did not exist when Visual N-Grams was published. As a closer comparison, we trained a CLIP ResNet-50 on the same YFCC100M dataset that Visual N-Grams was trained on and found it matched their reported ImageNet performance within a V100 GPU day. This baseline was also trained from scratch instead of being initialized from pre-trained ImageNet weights as in Visual N-Grams.


CLIP also outperforms Visual N-Grams on the other 2 reported datasets. On a Yahoo, CLIP achieves a $95\%$ reduction in the number of errors, and on SUN, CLIP more than doubles the accuracy of Visual N-Grams. To conduct a more comprehensive analysis and stress test, we implement a much larger evaluation suite detailed in Appendix A. In total we expand from the 3 datasets reported in Visual N-Grams to include over 30 datasets and compare to over 50 existing computer vision systems to contextualize results.


# 3.1.4. PROMPT ENGINEERING AND ENSEMBLING


Most standard image classification datasets treat the information naming or describing classes which enables natural language based zero-shot transfer as an afterthought. The vast majority of datasets annotate images with just a numeric id of the label and contain a file mapping these ids back to their names in English. Some datasets, such as Flowers102 and GTSRB, don't appear to include this mapping at all in their released versions preventing zero-shot transfer entirely. For many datasets, we observed these labels may be


2Alec learned much more about flower species and German traffic signs over the course of this project than he originally anticipated.


![图片 page7-9](images/page007_img01.png)


_Figure 4. Prompt engineering and ensembling improve zero-shot performance. Compared to the baseline of using contextless class names, prompt engineering and ensembling boost zero-shot classification performance by almost 5 points on average across 36 datasets. This improvement is similar to the gain from using 4 times more compute with the baseline zero-shot method but is "free" when amortized over many predictions._


chosen somewhat haphazardly and do not anticipate issues related to zero-shot transfer which relies on task description in order to transfer successfully.


A common issue is polysemy. When the name of a class is the only information provided to CLIP's text encoder it is unable to differentiate which word sense is meant due to the lack of context. In some cases multiple meanings of the same word might be included as different classes in the same dataset! This happens in ImageNet which contains both construction cranes and cranes that fly. Another example is found in classes of the Oxford-IIIT Pet dataset where the word boxer is, from context, clearly referring to a breed of dog, but to a text encoder lacking context could just as likely refer to a type of athlete.


Another issue we encountered is that it's relatively rare in our pre-training dataset for the text paired with the image to be just a single word. Usually the text is a full sentence describing the image in some way. To help bridge this distribution gap, we found that using the prompt template "A photo of a {label}." to be a good default that helps specify the text is about the content of the image. This often improves performance over the baseline of using only the label text. For instance, just using this prompt improves accuracy on ImageNet by $1.3\%$.


---


Similar to the "prompt engineering" discussion around GPT-3 (Brown et al., 2020; Gao et al., 2020), we have also observed that zero-shot performance can be significantly improved by customizing the prompt text to each task. A few, non exhaustive, examples follow. We found on several fine-grained image classification datasets that it helped to specify the category. For example on Oxford-IIIT Pets, using "A photo of a {label}", a type of pet." to help provide context worked well. Likewise, on Food101 specifying a type of food and on FGVC Aircraft a type of aircraft helped too. For OCR datasets, we found that putting quotes around the text or number to be recognized improved performance. Finally, we found that on satellite image classification datasets it helped to specify that the images were of this form and we use variants of "a satellite photo of a {label}".


We also experimented with assembling over multiple zero-shot classifiers as another way of improving performance. These classifiers are computed by using different context prompts such as 'A photo of a big {label}' and "A photo of a small {label)". We construct the ensemble over the embedding space instead of probability space. This allows us to cache a single set of averaged text embeddings so that the compute cost of the ensemble is the same as using a single classifier when amortized over many predictions. We've observed assembling across many generated zero-shot classifiers to reliably improve performance and use it for the majority of datasets. On ImageNet, we ensemble 80 different context prompts and this improves performance by an additional $3.5\%$ over the single default prompt discussed above. When considered together, prompt engineering and assembling improve ImageNet accuracy by almost $5\%$. In Figure 4 we visualize how prompt engineering and assembling change the performance of a set of CLIP models compared to the contextless baseline approach of directly embedding the class name as done in Li et al. (2017).


# 3.1.5. ANALYSIS OF ZERO-SHOT CLIP PERFORMANCE


Since task-agnostic zero-shot classifiers for computer vision have been understudied, CLIP provides a promising opportunity to gain a better understanding of this type of model. In this section, we conduct a study of various properties of CLIP's zero-shot classifiers. As a first question, we look simply at how well zero-shot classifiers perform. To contextualize this, we compare to the performance of a simple off-the-shelf baseline: fitting a fully supervised, regularized, logistic regression classifier on the features of the canonical ResNet-50. In Figure 5 we show this comparison across 27 datasets. Please see Appendix A for details of datasets and setup.


Zero-shot CLIP outperforms this baseline slightly more of-


![图片 page8-7](images/page008_img01.png)


_Figure 5. Zero-shot CLIP is competitive with a fully supervised baseline. Across a 27 dataset eval suite, a zero-shot CLIP classifier outperforms a fully supervised linear classifier fitted on ResNet-50 features on 16 datasets, including ImageNet._


ten than not and wins on 16 of the 27 datasets. Looking at individual datasets reveals some interesting behavior. On fine-grained classification tasks, we observe a wide spread in performance. On two of these datasets, Stanford Cars and Food101, zero-shot CLIP outperforms logistic regression on ResNet-50 features by over $20\%$ while on two others, Flowers102 and FGVCAircraft, zero-shot CLIP underperforms by over $10\%$. On OxfordPets and Birdsnap, performance is much closer. We suspect these difference are primarily due to varying amounts of per-task supervision between WIT and ImageNet. On "general" object classification datasets such as ImageNet, CIFAR10/100, STL10, and PascalVOC2007 performance is relatively similar with a slight advantage for zero-shot CLIP in all cases. On STL10, CLIP achieves $99.3\%$ overall which appears to be a new state of the art despite not using any training examples. Zero-shot CLIP significantly outperforms a ResNet-50 on two datasets measuring action recognition in videos. On Kinetics700, CLIP outperforms a ResNet-50 by $14.5\%$. Zero-shot CLIP also outperforms a ResNet-50's features by $7.7\%$ on UCF101. We speculate this is due to natural language providing wider supervision for visual concepts involving verbs, compared to the noun-centric object supervision in ImageNet.


Looking at where zero-shot CLIP notably underperforms,


---


![图片 page9-2](images/page009_img01.png)


_Figure 6. Zero-shot CLIP outperforms few-shot linear probes. Zero-shot CLIP matches the average performance of a 4-shot linear classifier trained on the same feature space and nearly matches the best results of a 16-shot linear classifier across publicly available models. For both BiT-M and SimCLRv2, the best performing model is highlighted. Light gray lines are other models in the eval suite. The 20 datasets with at least 16 examples per class were used in this analysis._


we see that zero-shot CLIP is quite weak on several specialized, complex, or abstract tasks such as satellite image classification (EuroSAT and RESISC45), lymph node tumor detection (PatchCamelyon), counting objects in synthetic scenes (CLEVRCounts), self-driving related tasks such as German traffic sign recognition (GTSRB), recognizing distance to the nearest car (KITTI Distance). These results highlight the poor capability of zero-shot CLIP on more complex tasks. By contrast, non-expert humans can robustly perform several of these tasks, such as counting, satellite image classification, and traffic sign recognition, suggesting significant room for improvement. However, we caution that it is unclear whether measuring zero-shot transfer, as opposed to few-shot transfer, is a meaningful evaluation for difficult tasks that a learner has no prior experience with, such as lymph node tumor classification for almost all humans (and possibly CLIP).


While comparing zero-shot performance to fully supervised models contextualizes the task-learning capabilities of CLIP, comparing to few-shot methods is a more direct comparison, since zero-shot is its limit. In Figure 6, we visualize how zero-shot CLIP compares to few-shot logistic regression on the features of many image models including the best publicly available ImageNet models, self-supervised learning methods, and CLIP itself. While it is intuitive to


expect zero-shot to underperform one-shot, we instead find that zero-shot CLIP matches the performance of 4-shot logistic regression on the same feature space. This is likely due to an important difference between the zero-shot and few-shot approach. First, CLIP's zero-shot classifier is generated via natural language which allows for visual concepts to be directly specified ("communicated"). By contrast, "normal" supervised learning must infer concepts indirectly from training examples. Context-less example-based learning has the drawback that many different hypotheses can be consistent with the data, especially in the one-shot case. A single image often contains many different visual concepts. Although a capable learner is able to exploit visual cues and heuristics, such as assuming that the concept being demonstrated is the primary object in an image, there is no guarantee.


A potential resolution of this discrepancy between zero-shot and few-shot performance is to use CLIP's zero-shot classifier as a prior for the weights of the few-shot classifier. While adding an L2 penalty towards the generated weights is a straightforward implementation of this idea, we found that hyperparameter optimization would often select for such a large value of this regularizer that the resulting few-shot classifier was "just" the zero-shot classifier. Research into better methods of combining the strength of zero-shot transfer with flexibility of few-shot learning is a promising direction for future work.


When comparing zero-shot CLIP to few-shot logistic regression on the features of other models, zero-shot CLIP roughly matches the performance of the best performing 16-shot classifier in our evaluation suite, which uses the features of a BiT-M ResNet-152x2 trained on ImageNet-21K. We are certain that a BiT-L model trained on JFT-300M would perform even better but these models have not been publicly released. That a BiT-M ResNet-152x2 performs best in a 16-shot setting is somewhat surprising since, as analyzed in Section 3.2, the Noisy Student EfficientNet-L2 outperforms it in a fully supervised setting by almost $5\%$ on average across 27 datasets.


In addition to studying the average performance of zero-shot CLIP and few-shot logistic regression, we also examine performance on individual datasets. In Figure 7, we show estimates for the number of labeled examples per class that a logistic regression classifier on the same feature space requires to match the performance of zero-shot CLIP. Since zero-shot CLIP is also a linear classifier, this estimates the effective data efficiency of zero-shot transfer in this setting. In order to avoid training thousands of linear classifiers, we estimate the effective data efficiency based on a log-linear interpolation of the performance of a 1, 2, 4, 8, 16-shot (when possible), and a fully supervised linear classifier trained on each dataset. We find that zero-shot transfer can


---


![图片 page10-2](images/page010_img01.png)


_Figure 7. The data efficiency of zero-shot transfer varies widely. Calculating the number of labeled examples per class a linear classifier on the same CLIP feature space requires to match the performance of the zero-shot classifier contextualizes the effectiveness of zero-shot transfer. Values are estimated based on log-linear interpolation of 1, 2, 4, 8, 16-shot and fully supervised results. Performance varies widely from still underperforming a one-shot classifier on two datasets to matching an estimated 184 labeled examples per class._


have widely varying efficiency per dataset from less than 1 labeled example per class to 184. Two datasets, Flowers102 and EuroSAT underperform one-shot models. Half of the datasets require less than 5 examples per class with a median of 5.4. However, the mean estimated data efficiency is 20.8 examples per class. This is due to the $20\%$ of datasets where supervised classifiers require many labeled examples per class in order to match performance. On ImageNet, zero-shot CLIP matches the performance of a 16-shot linear classifier trained on the same feature space.


If we assume that evaluation datasets are large enough that the parameters of linear classifiers trained on them are well estimated, then, because CLIP's zero-shot classifier is also a linear classifier, the performance of the fully supervised classifiers roughly sets an upper bound for what zero-shot transfer can achieve. In Figure 8 we compare CLIP's zero-shot performance with fully supervised linear classifiers across datasets. The dashed, $ y = x $ line represents an "optimal" zero-shot classifier that matches the performance of its fully supervised equivalent. For most datasets, the performance of zero-shot classifiers still underperform fully supervised classifiers by $ 10\% $ to $ 25\% $, suggesting that there is still plenty of headroom for improving CLIP's task-learning and zero-shot transfer capabilities.


There is a positive correlation of 0.82 (p-value $< 10^{-6}$) between zero-shot performance and fully supervised perfor


![图片 page10-7](images/page010_img02.png)


_Figure 8. Zero-shot performance is correlated with linear probe performance but still mostly sub-optimal. Comparing zero-shot and linear probe performance across datasets shows a strong correlation with zero-shot performance mostly shifted 10 to 25 points lower. On only 5 datasets does zero-shot performance approach linear probe performance ($\leq 3$ point difference)._


mance, suggesting that CLIP is relatively consistent at connecting underlying representation and task learning to zero-shot transfer. However, zero-shot CLIP only approaches fully supervised performance on 5 datasets: STL10, CIFAR10, Food101, OxfordPets, and Caltech101. On all 5 datasets, both zero-shot accuracy and fully supervised accuracy are over $90\%$. This suggests that CLIP may be more effective at zero-shot transfer for tasks where its underlying representations are also high quality. The slope of a linear regression model predicting zero-shot performance as a function of fully supervised performance estimates that for every $1\%$ improvement in fully supervised performance, zero-shot performance improves by $1.28\%$. However, the 95th-percentile confidence intervals still include values of less than 1 (0.93-1.79).


Over the past few years, empirical studies of deep learning systems have documented that performance is predictable as a function of important quantities such as training compute and dataset size (Hestness et al., 2017; Kaplan et al., 2020). The GPT family of models has so far demonstrated consistent improvements in zero-shot performance across a $1000\mathrm{x}$ increase in training compute. In Figure 9, we check whether the zero-shot performance of CLIP follows a similar scaling pattern. We plot the average error rate of the 5 ResNet CLIP models across 39 evaluations on 36 different datasets and find that a similar log-log linear scaling trend holds for CLIP across a $44\mathrm{x}$ increase in model compute. While the overall trend is smooth, we found that performance on individual evaluations can be much noisier. We are unsure whether


---


![图片 page11-2](images/page011_img01.png)


_Figure 9. Zero-shot CLIP performance scales smoothly as a function of model compute. Across 39 evals on 36 different datasets, average zero-shot error is well modeled by a log-log linear trend across a $44\mathrm{x}$ range of compute spanning 5 different CLIP models. Lightly shaded lines are performance on individual evals, showing that performance is much more varied despite the smooth overall trend._


this is caused by high variance between individual training runs on sub-tasks (as documented in D'Amour et al. (2020)) masking a steadily improving trend or whether performance is actually non-monotonic as a function of compute on some tasks.


# 3.2. Representation Learning


While we have extensively analyzed the task-learning capabilities of CLIP through zero-shot transfer in the previous section, it is more common to study the representation learning capabilities of a model. There exist many ways to evaluate the quality of representations as well as disagreements over what properties an "ideal" representation should have (Locatello et al., 2020). Fitting a linear classifier on a representation extracted from the model and measuring its performance on various datasets is a common approach. An alternative is measuring the performance of end-to-end fine-tuning of the model. This increases flexibility, and prior work has convincingly demonstrated that fine-tuning outperforms linear classification on most image classification datasets (Kornblith et al., 2019; Zhai et al., 2019). While the high performance of fine-tuning motivates its study for practical reasons, we still opt for linear classifier based evaluation for several reasons. Our work is focused on developing a high-performing task and dataset-agnostic pre-training approach. Fine-tuning, because it adapts representations to each dataset during the fine-tuning phase, can compensate for and potentially mask failures to learn general and robust representations during the pre-training phase. Linear classifiers, because of their limited flexibility, instead highlight these failures and provide clear feedback during development. For CLIP, training supervised linear


classifiers has the added benefit of being very similar to the approach used for its zero-shot classifiers which enables extensive comparisons and analysis in Section 3.1. Finally, we aim to compare CLIP to a comprehensive set of existing models across many tasks. Studying 66 different models on 27 different datasets requires tuning 1782 different evaluations. Fine-tuning opens up a much larger design and hyperparameter space, which makes it difficult to fairly evaluate and computationally expensive to compare a diverse set of techniques as discussed in other large scale empirical studies (Lucic et al., 2018; Choi et al., 2019). By comparison, linear classifiers require minimal hyper-parameter tuning and have standardized implementations and evaluation procedures. Please see Appendix A for further details on evaluation.


Figure 10 summarizes our findings. To minimize selection effects that could raise concerns of confirmation or reporting bias, we first study performance on the 12 dataset evaluation suite from Kornblith et al. (2019). While small CLIP models such as a ResNet-50 and ResNet-101 outperform other ResNets trained on ImageNet-1K (BiT-S and the originals), they underperform ResNets trained on ImageNet-21K (BiTM). These small CLIP models also underperform models in the EfficientNet family with similar compute requirements. However, models trained with CLIP scale very well and the largest model we trained (ResNet-50x64) slightly outperforms the best performing existing model (a Noisy Student EfficientNet-L2) on both overall score and compute efficiency. We also find that CLIP vision transformers are about 3x more compute efficient than CLIP ResNets, which allows us to reach higher overall performance within our compute budget. These results qualitatively replicate the findings of Dosovitskiy et al. (2020) which reported that vision transformers are more compute efficient than convnets when trained on sufficiently large datasets. Our best overall model is a ViT-L/14 that is fine-tuned at a higher resolution of 336 pixels on our dataset for 1 additional epoch. This model outperforms the best existing model across this evaluation suite by an average of $2.6\%$.


As Figure 21 qualitatively shows, CLIP models learn a wider set of tasks than has previously been demonstrated in a single computer vision model trained end-to-end from random initialization. These tasks include geo-localization, optical character recognition, facial emotion recognition, and action recognition. None of these tasks are measured in the evaluation suite of Kornblith et al. (2019). This could be argued to be a form of selection bias in Kornblith et al. (2019)'s study towards tasks that overlap with ImageNet. To address this, we also measure performance on a broader 27 dataset evaluation suite. This evaluation suite, detailed in Appendix A includes datasets representing the aforementioned tasks, German Traffic Signs Recognition Benchmark (Stallkamp et al., 2011), as well as several other datasets adapted from VTAB (Zhai et al., 2019).


---


![图片 page12-2](images/page012_img01.png)


![图片 page12-3](images/page012_img02.png)


_CLIP-ViT_


_CLIP-ResNet_


_EfficientNet-NoisyStudent_


_EfficientNet_


_- Instagram-pretrained_


_SimCLRv2_


_BYOL_


_MoCo_


_ViT (ImageNet-21k)_


_BiT-M_


_BiT-S_


_ResNet_


_Figure 10. Linear probe performance of CLIP models in comparison with state-of-the-art computer vision models, including EfficientNet (Tan & Le, 2019; Xie et al., 2020), MoCo (Chen et al., 2020d), Instagram-pretrained ResNeXt models (Mahajan et al., 2018; Touvron et al., 2019), BiT (Kolesnikov et al., 2019), ViT (Dosovitskiy et al., 2020), SimCLRv2 (Chen et al., 2020c), BYOL (Grill et al., 2020), and the original ResNet models (He et al., 2016b). (Left) Scores are averaged over 12 datasets studied by Kornblith et al. (2019). (Right) Scores are averaged over 27 datasets that contain a wider variety of distributions. Dotted lines indicate models fine-tuned or evaluated on images at a higher-resolution than pre-training. See Table 10 for individual scores and Figure 20 for plots for each dataset._


On this broader evaluation suite, the benefits of CLIP are more clear. All CLIP models, regardless of scale, outperform all evaluated systems in terms of compute efficiency. The improvement in average score of the best model over previous systems increases from $2.6\%$ to $5\%$. We also find that self-supervised systems do noticeably better on our broader evaluation suite. For instance, while SimCLRv2 still underperforms BiT-M on average on the 12 datasets of Kornblith et al. (2019), SimCLRv2 outperforms BiT-M on our 27 dataset evaluation suite. These findings suggest continuing to expand task diversity and coverage in order to better understand the "general" performance of systems. We suspect additional evaluation efforts along the lines of VTAB to be valuable.


In addition to the aggregate analysis above, we visualize per-dataset differences in the performance of the best CLIP model and the best model in our evaluation suite across all 27 datasets in Figure 11. CLIP outperforms the Noisy Student EfficientNet-L2 on 21 of the 27 datasets. CLIP improves the most on tasks which require OCR (SST2


and Hateful Memes), geo-localization and scene recognition (Country211, SUN397), and activity recognition in videos (Kinetics700 and UCF101). In addition CLIP also does much better on fine-grained car and traffic sign recognition (Stanford Cars and GTSRB). This may reflect a problem with overly narrow supervision in ImageNet. A result such as the $14.7\%$ improvement on GTSRB could be indicative of an issue with ImageNet-1K, which has only a single label for all traffic and street signs. This could encourage a supervised representation to collapse intra-class details and hurt accuracy on a fine-grained downstream task. As mentioned, CLIP still underperforms the EfficientNet on several datasets. Unsurprisingly, the dataset that the EfficientNet does best relative to CLIP on is the one it was trained on: ImageNet. The EfficientNet also slightly outperforms CLIP on low-resolution datasets such as CIFAR10 and CIFAR100. We suspect this is at least partly due to the lack of scale-based data augmentation in CLIP. The EfficientNet also does slightly better on PatchCamelyon and CLEVRCounts, datasets where overall performance is still


---


![图片 page13-2](images/page013_img01.png)


_Figure 11. CLIP's features outperform the features of the best ImageNet model on a wide variety of datasets. Fitting a linear classifier on CLIP's features outperforms using the Noisy Student EfficientNet-L2 on 21 out of 27 datasets._


low for both approaches.


# 3.3. Robustness to Natural Distribution Shift


In 2015, it was announced that a deep learning model exceeded human performance on the ImageNet test set (He et al., 2015). However, research in the subsequent years has repeatedly found that these models still make many simple mistakes (Dodge & Karam, 2017; Geirhos et al., 2018; Alcorn et al., 2019), and new benchmarks testing these systems has often found their performance to be much lower than both their ImageNet accuracy and human accuracy (Recht et al., 2019; Barbu et al., 2019). What explains this discrepancy? Various ideas have been suggested and studied (Ilyas et al., 2019; Geirhos et al., 2020). A common theme of proposed explanations is that deep learning models are exceedingly adept at finding correlations and patterns which hold across their training dataset and thus improve in-distribution performance. However many of these correlations and patterns are actually spurious and do not hold for other distributions and result in large drops in performance on other datasets.


We caution that, to date, most of these studies limit their evaluation to models trained on ImageNet. Recalling the topic of discussion, it may be a mistake to generalize too far from these initial findings. To what degree are these failures attributable to deep learning, ImageNet, or some


combination of the two? CLIP models, which are trained via natural language supervision on a very large dataset and are capable of high zero-shot performance, are an opportunity to investigate this question from a different angle.


Taori et al. (2020) is a recent comprehensive study moving towards quantifying and understanding these behaviors for ImageNet models. Taori et al. (2020) study how the performance of ImageNet models change when evaluated on natural distribution shifts. They measure performance on a set of 7 distribution shifts: ImageNetV2 (Recht et al., 2019), ImageNet Sketch (Wang et al., 2019), Youtube-BB and ImageNet-Vid (Shankar et al., 2019), ObjectNet (Barbu et al., 2019), ImageNet Adversarial (Hendrycks et al., 2019), and ImageNet Rendition (Hendrycks et al., 2020a). They distinguish these datasets, which all consist of novel images collected from a variety of sources, from synthetic distribution shifts such as ImageNet-C (Hendrycks & Dietterich, 2019), Stylized ImageNet (Geirhos et al., 2018), or adversarial attacks (Goodfellow et al., 2014) which are created by perturbing existing images in various ways. They propose this distinction because in part because they find that while several techniques have been demonstrated to improve performance on synthetic distribution shifts, they often fail to yield consistent improvements on natural distributions.$^3$


Across these collected datasets, the accuracy of ImageNet models drop well below the expectation set by the ImageNet validation set. For the following summary discussion we report average accuracy across all 7 natural distribution shift datasets and average accuracy across the corresponding class subsets of ImageNet unless otherwise specified. Additionally, for Youtube-BB and ImageNet-Vid, which have two different evaluation settings, we use the average of pm-0 and pm-10 accuracy.


A ResNet-101 makes 5 times as many mistakes when evaluated on these natural distribution shifts compared to the ImageNet validation set. Encouragingly however, Taori et al. (2020) find that accuracy under distribution shift increases predictably with ImageNet accuracy and is well modeled as a linear function of logit-transformed accuracy. Taori et al. (2020) use this finding to propose that robustness analysis should distinguish between effective and relative robustness. Effective robustness measures improvements in accuracy under distribution shift above what is predicted by the documented relationship between in-distribution and out-of-distribution accuracy. Relative robustness captures any improvement in out-of-distribution accuracy. Taori et al. (2020) argue that robustness techniques should aim to improve both effective robustness and relative robustness.


Almost all models studied in Taori et al. (2020) are trained


<sup>3</sup>We refer readers to Hendrycks et al. (2020a) for additional experiments and discussion on this claim.


---


![图片 page14-2](images/page014_img01.png)


![图片 page14-3](images/page014_img02.png)


<table><tr><td>★</td><td>CLIP-ViT</td><td>×</td><td>Instagram</td><td>△</td><td>ViT (ImageNet-21k)</td></tr><tr><td>☆</td><td>CLIP-ResNet</td><td>◇</td><td>SimCLRv2</td><td>▲</td><td>BiT-M</td></tr><tr><td>+</td><td>EfficientNet-NoisyStudent</td><td>丫</td><td>BYOL</td><td>▼</td><td>BiT-S</td></tr><tr><td>+</td><td>EfficientNet</td><td>●</td><td>MoCo</td><td>+</td><td>ResNet</td></tr></table>


_Figure 12. CLIP's features are more robust to task shift when compared to models pre-trained on ImageNet. For both dataset splits, the transfer scores of linear probes trained on the representations of CLIP models are higher than other models with similar ImageNet performance. This suggests that the representations of models trained on ImageNet are somewhat overfit to their task._


or fine-tuned on the ImageNet dataset. Returning to the discussion in the introduction to this section - is training or adapting to the ImageNet dataset distribution the cause of the observed robustness gap? Intuitively, a zero-shot model should not be able to exploit spurious correlations or patterns that hold only on a specific distribution, since it is not trained on that distribution. $ {}^{4} $ Thus it is reasonable to expect zero-shot models to have much higher effective robustness. In Figure 13, we compare the performance of zero-shot CLIP with existing ImageNet models on natural distribution shifts. All zero-shot CLIP models improve effective robustness by a large amount and reduce the size of the gap between ImageNet accuracy and accuracy under distribution shift by up to 75%.


While these results show that zero-shot models can be much more robust, they do not necessarily mean that supervised learning on ImageNet causes a robustness gap. Other details of CLIP, such as its large and diverse pre-training dataset or use of natural language supervision could also result


<sup>4</sup>We caution that a zero-shot model can still exploit spurious correlations that are shared between the pre-training and evaluation distributions.


in much more robust models regardless of whether they are zero-shot or fine-tuned. As an initial experiment to potentially begin narrowing this down, we also measure how the performance of CLIP models change after adapting to the ImageNet distribution via a L2 regularized logistic regression classifier fit to CLIP features on the ImageNet training set. We visualize how performance changes from the zero-shot classifier in Figure 14. Although adapting CLIP to the ImageNet distribution increases its ImageNet accuracy by $9.2\%$ to $85.4\%$ overall, and ties the accuracy of the 2018 SOTA from Mahajan et al. (2018), average accuracy under distribution shift slightly decreases.


It is surprising to see a $9.2\%$ increase in accuracy, which corresponds to roughly 3 years of improvement in SOTA, fail to translate into any improvement in average performance under distribution shift. We also break down the differences between zero-shot accuracy and linear classifier accuracy per dataset in Figure 14 and find performance still increases significantly on one dataset, ImageNetV2. ImageNetV2 closely followed the creation process of the original ImageNet dataset which suggests that gains in accuracy from supervised adaptation are closely concentrated around the ImageNet distribution. Performance decreases by $4.7\%$ on


---


![图片 page15-2](images/page015_img01.png)


_Figure 13. Zero-shot CLIP is much more robust to distribution shift than standard ImageNet models. (Left) An ideal robust model (dashed line) performs equally well on the ImageNet distribution and on other natural image distributions. Zero-shot CLIP models shrink this "robustness gap" by up to $75\%$. Linear fits on logit transformed values are shown with bootstrap estimated $95\%$ confidence intervals. (Right) Visualizing distribution shift for bananas, a class shared across 5 of the 7 natural distribution shift datasets. The performance of the best zero-shot CLIP model, ViT-L/14@336px, is compared with a model that has the same performance on the ImageNet validation set, ResNet-101._


ImageNet-R, $3.8\%$ on ObjectNet, $2.8\%$ on ImageNet Sketch, and $1.9\%$ on ImageNet-A. The change in accuracy on the two other datasets, Youtube-BB and ImageNet Vid, is insignificant.


How is it possible to improve accuracy by $9.2\%$ on the ImageNet dataset with little to no increase in accuracy under distribution shift? Is the gain primarily from "exploiting spurious correlations"? Is this behavior unique to some combination of CLIP, the ImageNet dataset, and the distribution shifts studied, or a more general phenomena? Does it hold for end-to-end finetuning as well as linear classifiers? We do not have confident answers to these questions at this time. Prior work has also pre-trained models on distributions other than ImageNet, but it is common to study and release models only after they have been fine-tuned to ImageNet. As a step towards understanding whether pre-trained zero-shot models consistently have higher effective robustness than fine-tuned models, we encourage the authors of Mahajan et al. (2018), Kolesnikov et al. (2019), and Dosovitskiy et al. (2020) to, if possible, study these questions on their models as well.


We also investigate another robustness intervention enabled by flexible zero-shot natural-language-based image classifiers. The target classes across the 7 transfer datasets are not always perfectly aligned with those of ImageNet. Two datasets, Youtube-BB and ImageNet-Vid, consist of superclasses of ImageNet. This presents a problem when trying to use the fixed 1000-way classifier of an ImageNet model to make predictions. Taori et al. (2020) handle this by max-


pooling predictions across all sub-classes according to the ImageNet class hierarchy. Sometimes this mapping is much less than perfect. For the person class in Youtube-BB, predictions are made by pooling over the ImageNet classes for a baseball player, a bridegroom, and a scuba diver. With CLIP we can instead generate a custom zero-shot classifier for each dataset directly based on its class names. In Figure 14 we see that this improves average effective robustness by $5\%$ but is concentrated in large improvements on only a few datasets. Curiously, accuracy on ObjectNet also increases by $2.3\%$. Although the dataset was designed to closely overlap with ImageNet classes, using the names provided for each class by ObjectNet's creators still helps a small amount compared to using ImageNet class names and pooling predictions when necessary.


While zero-shot CLIP improves effective robustness, Figure 14 shows that the benefit is almost entirely gone in a fully supervised setting. To better understand this difference, we investigate how effective robustness changes on the continuum from zero-shot to fully supervised. In Figure 15 we visualize the performance of 0-shot, 1-shot, 2-shot, 4-shot ..., 128-shot, and fully supervised logistic regression classifiers on the best CLIP model's features. We see that while few-shot models also show higher effective robustness than existing models, this benefit fades as in-distribution performance increases with more training data and is mostly, though not entirely, gone for the fully supervised model. Additionally, zero-shot CLIP is notably more robust than a few-shot model with equivalent ImageNet performance.


---


![图片 page16-2](images/page016_img01.png)


![图片 page16-3](images/page016_img02.png)


_Figure 14. While supervised adaptation to ImageNet increases ImageNet accuracy by $9.2\%$, it slightly reduces average robustness. (Left) Customizing zero-shot CLIP to each dataset improves robustness compared to using a single static zero-shot ImageNet classifier and pooling predictions across similar classes as in Taori et al. (2020). CLIP models adapted to ImageNet have similar effective robustness as the best prior ImageNet models. (Right) Details of per dataset changes in accuracy for the two robustness interventions. Adapting to ImageNet increases accuracy on ImageNetV2 noticeably but trades off accuracy on several other distributions. Dataset specific zero-shot classifiers can improve accuracy by a large amount but are limited to only a few datasets that include classes which don't perfectly align with ImageNet categories._


Across our experiments, high effective robustness seems to result from minimizing the amount of distribution specific training data a model has access to, but this comes at a cost of reducing dataset-specific performance.


Taken together, these results suggest that the recent shift towards large-scale task and dataset agnostic pre-training combined with a reorientation towards zero-shot and few-shot benchmarking on broad evaluation suites (as advocated by Yogatama et al. (2019) and Linzen (2020)) promotes the development of more robust systems and provides a more accurate assessment of performance. We are curious to see if the same results hold for zero-shot models in the field of NLP such as the GPT family. While Hendrycks et al. (2020b) has reported that pre-training improves relative robustness on sentiment analysis, Miller et al. (2020)'s study of the robustness of question answering models under natural distribution shift finds, similar to Taori et al. (2020), little evidence of effective robustness improvements to date.


# 4. Comparison to Human Performance


How does CLIP compare to human performance and human learning? To get a better understanding of how well humans perform in similar evaluation settings to CLIP, we evaluated


humans on one of our tasks. We wanted to get a sense of how strong human zero-shot performance is at these tasks, and how much human performance is improved if they are shown one or two image samples. This can help us to compare task difficulty for humans and CLIP, and identify correlations and differences between them.


We had five different humans look at each of 3669 images in the test split of the Oxford IIT Pets dataset (Parkhi et al., 2012) and select which of the 37 cat or dog breeds best matched the image (or 'I don't know' if they were completely uncertain). In the zero-shot case the humans were given no examples of the breeds and asked to label them to the best of their ability without an internet search. In the one-shot experiment the humans were given one sample image of each breed and in the two-shot experiment they were given two sample images of each breed.[5]


One possible concern was that the human workers were not sufficiently motivated in the zero-shot task. High human accuracy of $94\%$ on the STL-10 dataset (Coates et al., 2011)


There is not a perfect correspondence between the human few-shot tasks and the model's few-shot performance since the model cannot refer to sample images in the way that the humans can.


---


![图片 page17-2](images/page017_img01.png)


_Figure 15. Few-shot CLIP also increases effective robustness compared to existing ImageNet models but is less robust than zero-shot CLIP. Minimizing the amount of ImageNet training data used for adaption increases effective robustness at the cost of decreasing relative robustness. 16-shot logistic regression CLIP matches zero-shot CLIP on ImageNet, as previously reported in Figure 7, but is less robust._


and $97 - 100\%$ accuracy on the subset of attention check images increased our trust in the human workers.


Interestingly, humans went from a performance average of $54\%$ to $76\%$ with just one training example per class, and the marginal gain from an additional training example is minimal. The gain in accuracy going from zero to one shot is almost entirely on images that humans were uncertain about. This suggests that humans "know what they don't know" and are able to update their priors on the images they are most uncertain in based on a single example. Given this, it seems that while CLIP is a promising training strategy for zero-shot performance (Figure 5) and does well on tests of natural distribution shift (Figure 13), there is a large difference between how humans learn from a few examples and the few-shot methods in this paper.


This suggests that there are still algorithmic improvements waiting to be made to decrease the gap between machine and human sample efficiency, as noted by Lake et al. (2016) and others. Because these few-shot evaluations of CLIP don't make effective use of prior knowledge and the humans do, we speculate that finding a method to properly integrate prior knowledge into few-shot learning is an important step in algorithmic improvements to CLIP. To our knowledge, using a linear classifier on top of the features of a high-


<table><tr><td></td><td>Accuracy</td><td>Majority Vote on Full Dataset</td><td>Accuracy on Guesses</td><td>Majority Vote Accuracy on Guesses</td></tr><tr><td>Zero-shot human</td><td>53.7</td><td>57.0</td><td>69.7</td><td>63.9</td></tr><tr><td>Zero-shot CLIP</td><td>93.5</td><td>93.5</td><td>93.5</td><td>93.5</td></tr><tr><td>One-shot human</td><td>75.7</td><td>80.3</td><td>78.5</td><td>81.2</td></tr><tr><td>Two-shot human</td><td>75.7</td><td>85.0</td><td>79.2</td><td>86.1</td></tr></table>


_Table 2. Comparison of human performance on Oxford IIT Pets. As in Parkhi et al. (2012), the metric is average per-class classification accuracy. Most of the gain in performance when going from the human zero shot case to the human one shot case is on images that participants were highly uncertain on. "Guesses" refers to restricting the dataset to where participants selected an answer other than "I don't know", the "majority vote" is taking the most frequent (exclusive of ties) answer per image._


quality pre-trained model is near state-of-the-art for few shot learning (Tian et al., 2020), which suggests that there is a gap between the best few-shot machine learning methods and human few-shot learning.


If we plot human accuracy vs CLIP's zero shot accuracy (Figure 16), we see that the hardest problems for CLIP are also hard for humans. To the extent that errors are consistent, our hypothesis is that this is due to at least a two factors: noise in the dataset (including mistrabeled images) and out of distribution images being hard for both humans and models.


# 5. Data Overlap Analysis


A concern with pre-training on a very large internet dataset is unintentional overlap with downstream evals. This is important to investigate since, in a worst-case scenario, a complete copy of an evaluation dataset could leak into the pre-training dataset and invalidate the evaluation as a meaningful test of generalization. One option to prevent this is to identify and remove all duplicates before training a model. While this guarantees reporting true hold-out performance, it requires knowing all possible data which a model might be evaluated on ahead of time. This has the downside of limiting the scope of benchmarking and analysis. Adding a new evaluation would require an expensive re-train or risk reporting an un-quantified benefit due to overlap.


Instead, we document how much overlap occurs and how performance changes due to these overlaps. To do this, we use the following procedure:


1) For each evaluation dataset, we run a duplicate detector (see Appendix C) on its examples. We then manually inspect the found nearest neighbors and set a per dataset threshold to keep high precision while maximizing recall. Using this threshold, we then create two new subsets, Overlap, which contains all examples which have a similarity to a training example above the threshold, and Clean, which


---


![图片 page18-2](images/page018_img01.png)


_Figure 16. The hardest problems for CLIP also tend to be the hardest problems for humans. Here we rank image categories by difficulty for CLIP as measured as probability of the correct label._


contains all examples that are below this threshold. We denote the unaltered full dataset A11 for reference. From this we first record the degree of data contamination as the ratio of the number of examples in Overlap to the size of A11.


2) We then compute the zero-shot accuracy of CLIP RN50x64 on the three splits and report All - Clean as our main metric. This is the difference in accuracy due to contamination. When positive it is our estimate of how much the overall reported accuracy on the dataset was inflated by over-fitting to overlapping data.


3) The amount of overlap is often small so we also run a binomial significance test where we use the accuracy on Clean as the null hypothesis and compute the one-tailed (greater) p-value for the Overlap subset. We also calculate $99.5\%$ Clopper-Pearson confidence intervals on Dirty as another check.


A summary of this analysis is presented in Figure 17. Out of 35 datasets studied, 9 datasets have no detected overlap at all. Most of these datasets are synthetic or specialized making them unlikely to be posted as normal images on the internet (for instance MNIST, CLEVR, and GTSRB) or are guaranteed to have no overlap due to containing novel data from after the date our dataset was created (ObjectNet and Hateful Memes). This demonstrates our detector has a low-false positive rate which is important as false positives would under-estimate the effect of contamination in


our analysis. There is a median overlap of $2.2\%$ and an average overlap of $3.2\%$. Due to this small amount of overlap, overall accuracy is rarely shifted by more than $0.1\%$ with only 7 datasets above this threshold. Of these, only 2 are statistically significant after Bonferroni correction. The max detected improvement is only $0.6\%$ on Birdsnap which has the second largest overlap at $12.1\%$. The largest overlap is for Country211 at $21.5\%$. This is due to it being constructed out of YFCC100M, which our pre-training dataset contains a filtered subset of. Despite this large overlap there is only a $0.2\%$ increase in accuracy on Country211. This may be because the training text accompanying an example is often not related to the specific task a downstream eval measures. Country211 measures geo-localization ability, but inspecting the training text for these duplicates showed they often do not mention the location of the image.


We are aware of two potential concerns with our analysis. First our detector is not perfect. While it achieves near $100\%$ accuracy on its proxy training task and manual inspection + threshold tuning results in very high precision with good recall among the found nearest-neighbors, we cannot tractably check its recall across 400 million examples. Another potential confounder of our analysis is that the underlying data distribution may shift between the Overlap and Clean subsets. For example, on Kinetics-700 many "overlaps" are in fact all black transition frames. This explains why Kinetics-700 has an apparent $20\%$ accuracy drop on Overlap. We suspect more subtle distribution shifts likely exist. One possibility we noticed on CIFAR-100 is that, due to the very low resolution of its images, many duplicates were false positives of small objects such as birds or planes. Changes in accuracy could instead be due to changes in the class distribution or difficulty of the duplicates. Unfortunately, these distribution and difficulty shifts could also mask the effects of over-fitting.


However, these results closely follow the findings of similar duplicate analysis in previous work on large scale pretraining. Mahajan et al. (2018) and Kolesnikov et al. (2019) detected similar overlap rates and found minimal changes in overall performance. Importantly, Kolesnikov et al. (2019) also compared the alternative de-duplication strategy discussed in the introduction to this section with the approach we settled on and observed little difference between the two approaches.


# 6. Limitations


There are still many limitations to CLIP. While several of these are discussed as part of analysis in various sections, we summarize and collect them here.


On datasets with training splits, the performance of zero-shot CLIP is on average competitive with the simple su


---


![图片 page19-2](images/page019_img01.png)


![图片 page19-3](images/page019_img02.png)


_Figure 17. Few statistically significant improvements in accuracy due to detected data overlap. (Left) While several datasets have up to $\pm 20\%$ apparent differences in zero-shot accuracy on detected overlapping vs clean examples only 5 datasets out of 35 total have $99.5\%$ Clopper-Pearson confidence intervals that exclude a $0\%$ accuracy difference. 2 of these datasets do worse on overlapping data. (Right) Since the percentage of detected overlapping examples is almost always in the single digits, the overall test accuracy gain due to overlap is much smaller with the largest estimated increase being only $0.6\%$ on Birdsnap. Similarly, for only 6 datasets are the accuracy improvements statistically significant when calculated using a one-sided binomial test._


pervised baseline of a linear classifier on top of ResNet-50 features. On most of these datasets, the performance of this baseline is now well below the overall state of the art. Significant work is still needed to improve the task learning and transfer capabilities of CLIP. While scaling has so far steadily improved performance and suggests a route for continued improvement, we estimate around a 1000x increase in compute is required for zero-shot CLIP to reach overall state-of-the-art performance. This is infeasible to train with current hardware. Further research into improving upon the computational and data efficiency of CLIP will be necessary.


Analysis in Section 3.1 found that CLIP's zero-shot performance is still quite weak on several kinds of tasks. When compared to task-specific models, the performance of CLIP is poor on several types of fine-grained classification such as differentiating models of cars, species of flowers, and variants of aircraft. CLIP also struggles with more abstract and systematic tasks such as counting the number of objects in an image. Finally for novel tasks which are unlikely to be included in CLIP's pre-training dataset, such as classifying the distance to the nearest car in a photo, CLIP's performance can be near random. We are confident that there are still many, many, tasks where CLIP's zero-shot performance is near chance level.


While zero-shot CLIP generalizes well to many natural image distributions as investigated in Section 3.3, we've observed that zero-shot CLIP still generalizes poorly to data that is truly out-of-distribution for it. An illustrative example occurs for the task of OCR as reported in Appendix E.


CLIP learns a high quality semantic OCR representation that performs well on digitally rendered text, which is common in its pre-training dataset, as evidenced by performance on Rendered SST2. However, CLIP only achieves $88\%$ accuracy on the handwritten digits of MNIST. An embarrassingly simple baseline of logistic regression on raw pixels outperforms zero-shot CLIP. Both semantic and near-duplicate nearest-neighbor retrieval verify that there are almost no images that resemble MNIST digits in our pre-training dataset. This suggests CLIP does little to address the underlying problem of brittle generalization of deep learning models. Instead CLIP tries to circumvent the problem and hopes that by training on such a large and varied dataset that all data will be effectively in-distribution. This is a naive assumption that, as MNIST demonstrates, is easy to violate.


Although CLIP can flexibly generate zero-shot classifiers for a wide variety of tasks and datasets, CLIP is still limited to choosing from only those concepts in a given zero-shot classifier. This is a significant restriction compared to a truly flexible approach like image captioning which could generate novel outputs. Unfortunately, as described in Section 2.3 we found the computational efficiency of the image caption baseline we tried to be much lower than CLIP. A simple idea worth trying is joint training of a contrastive and generative objective with the hope of combining the efficiency of CLIP with the flexibility of a caption model. As another alternative, search could be performed at inference time over many natural language explanations of a given image, similar to approach proposed in Learning with Latent Language Andreas et al. (2017).


---


CLIP also does not address the poor data efficiency of deep learning. Instead CLIP compensates by using a source of supervision that can be scaled to hundreds of millions of training examples. If every image seen during training of a CLIP model was presented at a rate of one per second, it would take 405 years to iterate through the 12.8 billion images seen over 32 training epochs. Combining CLIP with self-supervision (Henaff, 2020; Chen et al., 2020c) and self-training (Lee; Xie et al., 2020) methods is a promising direction given their demonstrated ability to improve data efficiency over standard supervised learning.


Our methodology has several significant limitations. Despite our focus on zero-shot transfer, we repeatedly queried performance on full validation sets to guide the development of CLIP. These validation sets often have thousands of examples, which is unrealistic for true zero-shot scenarios. Similar concerns have been raised in the field of semi-supervised learning (Oliver et al., 2018). Another potential issue is our selection of evaluation datasets. While we have reported results on Kornblith et al. (2019)'s 12 dataset evaluation suite as a standardized collection, our main results use a somewhat haphazardly assembled collection of 27 datasets that is undeniably co-adapted with the development and capabilities of CLIP. Creating a new benchmark of tasks designed explicitly to evaluate broad zero-shot transfer capabilities, rather than re-using existing supervised datasets, would help address these issues.


CLIP is trained on text paired with images on the internet. These image-text pairs are unfiltered and uncurated and result in CLIP models learning many social biases. This has been previously demonstrated for image caption models (Bhargava & Forsyth, 2019). We refer readers to Section 7 for detailed analysis and quantification of these behaviors for CLIP as well as discussion of potential mitigation strategies.


While we have emphasized throughout this work that specifying image classifiers through natural language is a flexible and general interface, it has its own limitations. Many complex tasks and visual concepts can be difficult to specify just through text. Actual training examples are undeniably useful but CLIP does not optimize for few-shot performance directly. In our work, we fall back to fitting linear classifiers on top of CLIP's features. This results in a counter-intuitive drop in performance when transitioning from a zero-shot to a few-shot setting. As discussed in Section 4, this is notably different from human performance which shows a large increase from a zero to a one shot setting. Future work is needed to develop methods that combine CLIP's strong zero-shot performance with efficient few-shot learning.


# 7. Broader Impacts


CLIP has a wide range of capabilities due to its ability to carry out arbitrary image classification tasks. One can give it images of cats and dogs and ask it to classify cats, or give it images taken in a department store and ask it to classify shoplifters-a task with significant social implications and for which AI may be unfit. Like any image classification system, CLIP's performance and fitness for purpose need to be evaluated, and its broader impacts analyzed in context. CLIP also introduces a capability that will magnify and alter such issues: CLIP makes it possible to easily create your own classes for categorization (to 'roll your own classifier') without a need for re-training. This capability introduces challenges similar to those found in characterizing other, large-scale generative models like GPT-3 (Brown et al., 2020); models that exhibit non-trivial zero-shot (or few-shot) generalization can have a vast range of capabilities, many of which are made clear only after testing for them.


Our studies of CLIP in a zero-shot setting show that the model displays significant promise for widely-applicable tasks like image retrieval or search. For example, it can find relevant images in a database given text, or relevant text given an image. Further, the relative ease of steering CLIP toward bespoke applications with little or no additional data or training could unlock a variety of novel applications that are hard for us to envision today, as has occurred with large language models over the past few years.


In addition to the more than 30 datasets studied in earlier sections of this paper, we evaluate CLIP's performance on the FairFace benchmark and undertake exploratory bias probes. We then characterize the model's performance in a downstream task, surveillance, and discuss its usefulness as compared with other available systems. Many of CLIP's capabilities are omni-use in nature (e.g. OCR can be used to make scanned documents searchable, to power screen reading technologies, or to read license plates). Several of the capabilities measured, from action recognition, object classification, and geo-localization, to facial emotion recognition, can be used in surveillance. Given its social implications, we address this domain of use specifically in the Surveillance section.


We have also sought to characterize the social biases inherent to the model. Our bias tests represent our initial efforts to probe aspects of how the model responds in different scenarios, and are by nature limited in scope. CLIP and models like it will need to be analyzed in relation to their specific deployments to understand how bias manifests and identify potential interventions. Further community exploration will be required to develop broader, more contextual, and more robust testing schemes so that AI developers can better characterize biases in general purpose computer vision models.


---


<table><tr><td>Model</td><td>Race</td><td>Gender</td><td>Age</td></tr><tr><td>FairFace Model</td><td>93.7</td><td>94.2</td><td>59.7</td></tr><tr><td>Linear Probe CLIP</td><td>93.4</td><td>96.5</td><td>63.8</td></tr><tr><td>Zero-Shot CLIP</td><td>58.3</td><td>95.9</td><td>57.1</td></tr><tr><td>Linear Probe Instagram</td><td>90.8</td><td>93.2</td><td>54.2</td></tr></table>


_Table 3. Percent accuracy on Race, Gender, and Age classification of images in FairFace category 'White'_


<table><tr><td>Model</td><td>Race</td><td>Gender</td><td>Age</td></tr><tr><td>FairFace Model</td><td>75.4</td><td>94.4</td><td>60.7</td></tr><tr><td>Linear Probe CLIP</td><td>92.8</td><td>97.7</td><td>63.1</td></tr><tr><td>Zero-Shot CLIP</td><td>91.3</td><td>97.2</td><td>54.3</td></tr><tr><td>Linear Probe Instagram</td><td>87.2</td><td>93.9</td><td>54.1</td></tr></table>


_Table 4. Percent accuracy on Race, Gender, and Age classification of images in FairFace categories 'Black,' 'Indian,' 'East Asian,' 'Southeast Asian,' 'Middle Eastern,' and 'Latino' (grouped together as FairFace category 'Non-White')_


<table><tr><td>Model</td><td>Gender</td><td>Black</td><td>White</td><td>Indian</td><td>Latino</td><td>Middle Eastern</td><td>Southeast Asian</td><td>East Asian</td><td>Average</td></tr><tr><td rowspan="3">Linear Probe CLIP</td><td>Male</td><td>96.9</td><td>96.4</td><td>98.7</td><td>96.5</td><td>98.9</td><td>96.2</td><td>96.9</td><td>97.2</td></tr><tr><td>Female</td><td>97.9</td><td>96.7</td><td>97.9</td><td>99.2</td><td>97.2</td><td>98.5</td><td>97.3</td><td>97.8</td></tr><tr><td></td><td>97.4</td><td>96.5</td><td>98.3</td><td>97.8</td><td>98.4</td><td>97.3</td><td>97.1</td><td>97.5</td></tr><tr><td rowspan="3">Zero-Shot CLIP</td><td>Male</td><td>96.3</td><td>96.4</td><td>97.7</td><td>97.2</td><td>98.3</td><td>95.5</td><td>96.8</td><td>96.9</td></tr><tr><td>Female</td><td>97.1</td><td>95.3</td><td>98.3</td><td>97.8</td><td>97.5</td><td>97.2</td><td>96.4</td><td>97.0</td></tr><tr><td></td><td>96.7</td><td>95.9</td><td>98.0</td><td>97.5</td><td>98.0</td><td>96.3</td><td>96.6</td><td></td></tr><tr><td rowspan="3">Linear Probe Instagram</td><td>Male</td><td>92.5</td><td>94.8</td><td>96.2</td><td>93.1</td><td>96.0</td><td>92.7</td><td>93.4</td><td>94.1</td></tr><tr><td>Female</td><td>90.1</td><td>91.4</td><td>95.0</td><td>94.8</td><td>95.0</td><td>94.1</td><td>94.3</td><td>93.4</td></tr><tr><td></td><td>91.3</td><td>93.2</td><td>95.6</td><td>94.0</td><td>95.6</td><td>93.4</td><td>93.9</td><td></td></tr></table>


_Table 5. Percent accuracy on gender classification of images by FairFace race category_


# 7.1. Bias


Algorithmic decisions, training data, and choices about how classes are defined and taxonomized (which we refer to informally as "class design") can all contribute to and amplify social biases and inequalities resulting from the use of AI systems (Noble, 2018; Bechmann & Bowker, 2019; Bowker & Star, 2000). Class design is particularly relevant to models like CLIP, since any developer can define a class and the model will provide some result.


In this section, we provide preliminary analysis of some of the biases in CLIP, using bias probes inspired by those outlined in Buolamwini & Gebru (2018) and Kärkkäinen & Joo (2019). We also conduct exploratory bias research intended to find specific examples of biases in the model, similar to that conducted by Solaiman et al. (2019).


We start by analyzing the performance of Zero-Shot CLIP on the face image dataset FairFace (Kärkkäinen & Joo, 2019)<sup>6</sup>


$^{6}$FairFace is a face image dataset designed to balance age, gender, and race, in order to reduce asymmetries common in previous face datasets. It categorizes gender into 2 groups: female and male and race into 7 groups: White, Black, Indian, East Asian, Southeast Asian, Middle Eastern, and Latino. There are inherent problems with race and gender classifications, as e.g. Bowker & Star (2000)


as an initial bias probe, then probe the model further to surface additional biases and sources of biases, including class design.


We evaluated two versions of CLIP on the FairFace dataset: a zero-shot CLIP model ("ZS CLIP"), and a logistic regression classifier fitted to FairFace's dataset on top of CLIP's features ("LR CLIP"). We find that LR CLIP gets higher accuracy on the FairFace dataset than both the ResNext-101 32x48d Instagram model ("Linear Probe Instagram") (Mahajan et al., 2018) and FairFace's own model on most of the classification tests we ran<sup>7</sup>. ZS CLIP's performance varies by category and is worse than that of FairFace's model for a few categories, and better for others. (See Table 3 and Table 4).


and Keyes (2018) have shown. While FairFace's dataset reduces the proportion of White faces, it still lacks representation of entire large demographic groups, effectively erasing such categories. We use the 2 gender categories and 7 race categories defined in the FairFace dataset in a number of our experiments not in order to reinforce or endorse the use of such reductive categories, but in order to enable us to make comparisons to prior work.


One challenge with this comparison is that the FairFace model uses binary classes for race ("White" and "Non-White"), instead of breaking down races into finer-grained sub-groups.


---


<table><tr><td>Category</td><td>Black</td><td>White</td><td>Indian</td><td>Latino</td><td>Middle Eastern</td><td>Southeast Asian</td><td>East Asian</td></tr><tr><td>Crime-related Categories</td><td>16.4</td><td>24.9</td><td>24.4</td><td>10.8</td><td>19.7</td><td>4.4</td><td>1.3</td></tr><tr><td>Non-human Categories</td><td>14.4</td><td>5.5</td><td>7.6</td><td>3.7</td><td>2.0</td><td>1.9</td><td>0.0</td></tr></table>


_Table 6. Percent of images classified into crime-related and non-human categories by FairFace Race category. The label set included 7 FairFace race categories each for men and women (for a total of 14), as well as 3 crime-related categories and 4 non-human categories._


<table><tr><td>Category Label Set</td><td>0-2</td><td>3-9</td><td>10-19</td><td>20-29</td><td>30-39</td><td>40-49</td><td>50-59</td><td>60-69</td><td>over 70</td></tr><tr><td>Default Label Set</td><td>30.3</td><td>35.0</td><td>29.5</td><td>16.3</td><td>13.9</td><td>18.5</td><td>19.1</td><td>16.2</td><td>10.4</td></tr><tr><td>Default Label Set + ‘child’ category</td><td>2.3</td><td>4.3</td><td>14.7</td><td>15.0</td><td>13.4</td><td>18.2</td><td>18.6</td><td>15.5</td><td>9.4</td></tr></table>


_Table 7. Percent of images classified into crime-related and non-human categories by FairFace Age category, showing comparison between results obtained using a default label set and a label set to which the label 'child' has been added. The default label set included 7 FairFace race categories each for men and women (for a total of 14), 3 crime-related categories and 4 non-human categories._


Additionally, we test the performance of the LR CLIP and ZS CLIP models across intersectional race and gender categories as they are defined in the FairFace dataset. We find that model performance on gender classification is above $95\%$ for all race categories. Table 5 summarizes these results.


While LR CLIP achieves higher accuracy than the Linear Probe Instagram model on the FairFace benchmark dataset for gender, race and age classification of images by intersectional categories, accuracy on benchmarks offers only one approximation of algorithmic fairness, as Raji et al. (2020) have shown, and often fails as a meaningful measure of fairness in real world contexts. Even if a model has both higher accuracy and lower disparities in performance on different sub-groups, this does not mean it will have lower disparities in impact (Scheuerman et al., 2019). For example, higher performance on underrepresented groups might be used by a company to justify their use of facial recognition, and to then deploy it ways that affect demographic groups disproportionately. Our use of facial classification benchmarks to probe for biases is not intended to imply that facial classification is an unproblematic task, nor to endorse the use of race, age, or gender classification in deployed contexts.


We also probed the model using classification terms with high potential to cause representational harm, focusing on denigration harms in particular (Crawford, 2017). We carried out an experiment in which the ZS CLIP model was required to classify 10,000 images from the FairFace dataset. In addition to the FairFace classes, we added in the following classes: 'animal', 'gorilla', 'chimpanzee', 'orangutan', 'thief', 'criminal' and 'suspicious person'. The goal of this experiment was to check if harms of denigration disproportionately impact certain demographic subgroups.


We found that $4.9\%$ (confidence intervals between $4.6\%$ and $5.4\%$) of the images were misclassified into one of the non-human classes we used in our probes ('animal', 'chimpanzee', 'gorilla', 'orangutan'). Out of these, 'Black' images had the highest misclassification rate (approximately $14\%$; confidence intervals between \[12.6\%\] and \[16.4\%\]) while all other races had misclassification rates under $8\%$. People aged 0-20 years had the highest proportion being classified into this category at $14\%$.


We also found that $16.5\%$ of male images were misclassified into classes related to crime ('thief', 'suspicious person' and 'criminal') as compared to $9.8\%$ of female images. Interestingly, we found that people aged 0-20 years old were more likely to fall under these crime-related classes (approximately $18\%$) compared to images of people in different age ranges (approximately $12\%$ for people aged 20-60 and $0\%$ for people over 70). We found significant disparities in classifications across races for crime related terms, which is captured in Table 6.


Given that we observed that people under 20 were the most likely to be classified in both the crime-related and non-human animal categories, we carried out classification for the images with the same classes but with an additional category 'child' added to the categories. Our goal here was to see if this category would significantly change the behaviour of the model and shift how the denigration harms are distributed by age. We found that this drastically reduced the number of images of people under 20 classified in either crime-related categories or non-human animal categories (Table 7). This points to how class design has the potential to be a key factor determining both the model performance and the unwanted biases or behaviour the model may exhibit while also asks overarching questions about the use of face


---


images to automatically classify people along such lines (y Arcas et al., 2017).


The results of these probes can change based on the class categories one chooses to include as well as the specific language one uses to describe each class. Poor class design can lead to poor real world performance; this concern is particularly relevant to a model like CLIP, given how easily developers can design their own classes.


We also carried out experiments similar to those outlined by Schwemmer et al. (2020) to test how CLIP treated images of men and women differently using images of Members of Congress. As part of these experiments, we studied how certain additional design decisions such as deciding thresholds for labels can impact the labels output by CLIP and how biases manifest.


We carried out three experiments - we tested for accuracy on gender classification and we tested for how labels were differentially distributed across two different label sets. For our first label set, we used a label set of 300 occupations and for our second label set we used a combined set of labels that Google Cloud Vision, Amazon Rekognition and Microsoft Azure Computer Vision returned for all the images.


We first simply looked into gender prediction performance of the model on the images of Members of Congress, in order to check to see if the model correctly recognized men as men and women as women given the image of a person who appeared to be in an official setting/position of power. We found that the model got $100\%$ accuracy on the images. This is slightly better performance than the model's performance on the FairFace dataset. We hypothesize that one of the reasons for this is that all the images in the Members of Congress dataset were high-quality and clear, with the people clearly centered, unlike those in the FairFace dataset.


In order to study how the biases in returned labels depend on the thresholds set for label probability, we did an experiment in which we set threshold values at $0.5\%$ and $4.0\%$. We found that the lower threshold led to lower quality of labels. However, even the differing distributions of labels under this threshold can hold signals for bias. For example, we find that under the $0.5\%$ threshold labels such as 'nanny' and 'housekeeper' start appearing for women whereas labels such as 'prisoner' and 'mobster' start appearing for men. This points to gendered associations similar to those that have previously been found for occupations (Schwemmer et al., 2020) (Nosek et al., 2002) (Bolukbasi et al., 2016).


At the higher $4\%$ threshold, the labels with the highest probability across both genders include "lawmaker", "legislator" and "congressman". However, the presence of these biases amongst lower probability labels nonetheless point to larger questions about what 'sufficiently' safe behaviour may look


like for deploying such systems.


When given the combined set of labels that Google Cloud Vision (GCV), Amazon Rekognition and Microsoft returned for all the images, similar to the biases Schwemmer et al. (2020) found in GCV systems, we found our system also disproportionately attached labels to do with hair and appearance in general to women more than men. For example, labels such as 'brown hair', 'blonde' and 'blond' appeared significantly more often for women. Additionally, CLIP attached some labels that described high status occupations disproportionately more often to men such as 'executive' and 'doctor'. Out of the only four occupations that it attached more often to women, three were 'newscaster', 'television presenter' and 'newsreader' and the fourth was 'Judge'. This is again similar to the biases found in GCV and points to historical gendered differences (Schwemmer et al., 2020).


Interestingly, when we lowered the threshold to $0.5\%$ for this set of labels, we found that the labels disproportionately describing men also shifted to appearance oriented words such as 'suit', 'tie' and 'necktie' (Figure 18). Many occupation oriented words such as 'military person' and 'executive' - which were not used to describe images of women at the higher $4\%$ threshold - were used for both men and women at the lower $0.5\%$ threshold, which could have caused the change in labels for men. The reverse was not true. Descriptive words used to describe women were still uncommon amongst men.


Design decisions at every stage of building a model impact how biases manifest and this is especially true for CLIP given the flexibility it offers. In addition to choices about training data and model architecture, decisions about things like class designs and thresholding values can alter the labels a model outputs and as a result heighten or lower certain kinds of harm, such as those described by Crawford (2017). People designing and developing models and AI systems have considerable power. Decisions about things like class design are a key determiner not only of model performance, but also of how and in what contexts model biases manifest.


These experiments are not comprehensive. They illustrate potential issues stemming from class design and other sources of bias, and are intended to spark inquiry.


# 7.2. Surveillance


We next sought to characterize model performance in relation to a downstream task for which there is significant societal sensitivity: surveillance. Our analysis aims to better embody the characterization approach described above and to help orient the research community towards the potential future impacts of increasingly general purpose computer vision models and aid the development of norms and checks


---


![图片 page24-2](images/page024_img01.png)


![图片 page24-3](images/page024_img02.png)


_Figure 18. CLIP performance on Member of Congress images when given the combined returned label set for the images from Google Cloud Vision, Amazon Rekognition and Microsoft Azure Computer Vision. The 20 most gendered labels for men and women were identified with $\chi^2$ tests with the threshold at $0.5\%$. Labels are sorted by absolute frequencies. Bars denote the percentage of images for a certain label by gender._


around such systems. Our inclusion of surveillance is not intended to indicate enthusiasm for this domain - rather, we think surveillance is an important domain to try to make predictions about given its societal implications (Zuboff, 2015; Browne, 2015).


We measure the model's performance on classification of images from CCTV cameras and zero-shot celebrity identification. We first tested model performance on low-resolution images captured from surveillance cameras (e.g. CCTV cameras). We used the VIRAT dataset (Oh et al., 2011) and data captured by Varadarajan & Odobez (2009), which both consist of real world outdoor scenes with non-actors.


Given CLIP's flexible class construction, we tested 515 surveillance images captured from 12 different video sequences on self-constructed general classes for coarse and fine grained classification. Coarse classification required the model to correctly identify the main subject of the image (i.e. determine if the image was a picture of an empty parking lot, school campus, etc.). For fine-grained classification, the model had to choose between two options constructed to determine if the model could identify the presence/absence of smaller features in the image such as a person standing in the corner.


For coarse classification, we constructed the classes by hand-captioning the images ourselves to describe the contents of the image and there were always at least 6 options for


the model to choose from. Additionally, we carried out a 'stress test' where the class set included at least one more caption for something that was 'close' to the image (for example, 'parking lot with white car' vs. 'parking lot with red car'). We found that the model had a top-1 accuracy of $91.8\%$ on the CCTV images for the initial evaluation. The accuracy dropped significantly to $51.1\%$ for the second evaluation, with the model incorrectly choosing the 'close' answer $40.7\%$ of the time.


For fine-grained detection, the zero-shot model performed poorly, with results near random. Note that this experiment was targeted only towards detecting the presence or absence of small objects in image sequences.


We also tested CLIP's zero-shot performance for 'in the wild' identity detection using the CelebA dataset<sup>8</sup>. We did this to evaluate the model's performance for identity detection using just the publicly available data it was pre-trained on. While we tested this on a dataset of celebrities who have a larger number of images on the internet, we hypothesize that the number of images in the pre-training data needed for the model to associate faces with names will keep decreasing as models get more powerful (see Table 8), which has significant societal implications (Garvie, 2019). This


Note: The CelebA dataset is more representative of faces with lighter skin tones. Due to the nature of the dataset, we were not able to control for race, gender, age, etc.


---


<table><tr><td>Model</td><td>100 Classes</td><td>1k Classes</td><td>2k Classes</td></tr><tr><td>CLIP L/14</td><td>59.2</td><td>43.3</td><td>42.2</td></tr><tr><td>CLIP RN50x64</td><td>56.4</td><td>39.5</td><td>38.4</td></tr><tr><td>CLIP RN50x16</td><td>52.7</td><td>37.4</td><td>36.3</td></tr><tr><td>CLIP RN50x4</td><td>52.8</td><td>38.1</td><td>37.3</td></tr></table>


_Table 8. CelebA Zero-Shot Top-1 Identity Recognition Accuracy_


mirrors recent developments in natural language processing, in which recent large language models trained on Internet data often exhibit a surprising ability to provide information related to relatively minor public figures (Brown et al., 2020).


We found that the model had $59.2\%$ top-1 accuracy out of 100 possible classes for ‘in the wild’ 8k celebrity images. However, this performance dropped to $43.3\%$ when we increased our class sizes to 1k celebrity names. This performance is not competitive when compared to production level models such as Google’s Celebrity Recognition (Google). However, what makes these results noteworthy is that this analysis was done using only zero-shot identification capabilities based on names inferred from pre-training data - we didn’t use any additional task-specific dataset, and so the (relatively) strong results further indicate that before deploying multimodal models, people will need to carefully study them for behaviors in a given context and domain.


CLIP offers significant benefit for tasks that have relatively little data given its zero-shot capabilities. However, large datasets and high performing supervised models exist for many in-demand surveillance tasks such as facial recognition. As a result, CLIP's comparative appeal for such uses is low. Additionally, CLIP is not designed for common surveillance-relevant tasks like object detection and semantic segmentation. This means it has limited use for certain surveillance tasks when models that are designed with these uses in mind such as Detector2 (Wu et al., 2019) are widely available.


However, CLIP does unlock a certain aspect of usability given how it removes the need for training data. Thus, CLIP and similar models could enable bespoke, niche surveillance use cases for which no well-tailored models or datasets exist, and could lower the skill requirements to build such applications. As our experiments show, ZS CLIP displays nontrivial, but not exceptional, performance on a few surveillance relevant tasks today.


# 7.3. Future Work


This preliminary analysis is intended to illustrate some of the challenges that general purpose computer vision models pose and to give a glimpse into their biases and impacts.


We hope that this work motivates future research on the characterization of the capabilities, shortcomings, and biases of such models, and we are excited to engage with the research community on such questions.


We believe one good step forward is community exploration to further characterize the capabilities of models like CLIP and - crucially - identify application areas where they have promising performance and areas where they may have reduced performance<sup>9</sup>. This process of characterization can help researchers increase the likelihood models are used beneficially by:


Identifying potentially beneficial downstream uses of models early in the research process, enabling other researchers to think about applications.


- Surfacing tasks with significant sensitivity and a large set of societal stakeholders, which may call for intervention by policymakers.


- Better characterizing biases in models, alerting other researchers to areas of concern and areas for interventions.


- Creating suites of tests to evaluate systems like CLIP on, so we can better characterize model capabilities earlier in the development cycle.


Identifying potential failure modes and areas for further work.


We plan to contribute to this work, and hope this analysis provides some motivating examples for subsequent research.


# 8. Related Work


Any model that leverages written, spoken, signed or any other form of human language as part of its training signal is arguably using natural language as a source of supervision. This is an admittedly extremely broad area and covers most work in the field of distributional semantics including topic models (Blei et al., 2003), word, sentence, and paragraph vectors (Mikolov et al., 2013; Kiros et al., 2015; Le & Mikolov, 2014), and language models (Bengio et al., 2003). It also includes much of the broader field of NLP that deals with predicting or modeling sequences of natural language in some way. Work in NLP intentionally leveraging natural language supervision in the form of explanations, feedback, instructions, and advice for tasks such as classification (as opposed to the commonly used representation of supervision as a set of arbitrarily encoded discrete category labels) has


$ {}^{9} $ A model could be unfit for use due to inadequate performance or due to the inappropriateness of AI use in the application area itself.


---


been explored in many creative and advanced ways. Dialog based learning (Weston, 2016; Li et al., 2016; Hancock et al., 2019) develops techniques to learn from interactive natural language feedback in dialog. Several papers have leveraged semantic parsing to convert natural language explanations into features (Srivastava et al., 2017) or additional training labels (Hancock et al., 2018). More recently, ExpBERT (Murty et al., 2020) uses feature representations produced by conditioning a deep contextual language model on natural language explanations and descriptions of relations to improve performance on the task of relation extraction.


CLIP is an example of using natural language as a training signal for learning about a domain other than language. In this context, the earliest use of the term natural language supervision that we are aware of is the work of Ramanathan et al. (2013) which showed that natural language descriptions could be used along side other sources of supervision to improve performance on the task of video event understanding. However, as mentioned in the introduction and approach section, methods of leveraging natural language descriptions in computer vision well predate the use of this specific term, especially for image retrieval (Mori et al., 1999) and object classification (Wang et al., 2009). Other early work leveraged tags (but not natural language) associated with images for the task of semantic segmentation (Barnard et al., 2003). More recently, He & Peng (2017) and Liang et al. (2020) demonstrated using natural language descriptions and explanations to improve fine-grained visual classification of birds. Others have investigated how grounded language can be used to improve visual representations and classifiers on the ShapeWorld dataset (Kuhnle & Copestake, 2017; Andreas et al., 2017; Mu et al., 2019). Finally, techniques which combine natural language with reinforcement learning environments (Narasimhan et al., 2015) have demonstrated exciting emergent behaviors such as systematically accomplishing zero-shot tasks (Hill et al., 2019).


CLIP's pre-training task optimizes for text-image retrieval. This areas of research dates back to the mid-90s with the previously mentioned Mori et al. (1999) as representative of early work. While initial efforts focused primarily on predictive objectives over time research shifted towards learning joint multi-modal embedding spaces with techniques like kernel Canonical Correlation Analysis and various ranking objectives (Weston et al., 2010; Socher & Fei-Fei, 2010; Hodosh et al., 2013). Over time work explored many combinations of training objective, transfer, and more expressive models and steadily improved performance (Frome et al., 2013; Socher et al., 2014; Karpathy et al., 2014; Kiros et al., 2014; Faghri et al., 2017).


Other work has leveraged natural language supervision for domains other than images. Stroud et al. (2020) explores


large scale representation learning by training a system to pair descriptive text with videos instead of images. Several works have explored using dense spoken natural language supervision for videos (Miech et al., 2019; 2020b). When considered together with CLIP, these works suggest that large scale natural language supervision is a promising way to learn high quality perceptual systems for many domains. Alayrac et al. (2020) extended this line of work to an additional modality by adding raw audio as an additional supervision source and demonstrated benefits from combining all three sources of supervision.


As part of our work on CLIP we also construct a new dataset of image-text pairs. Modern work on image-text retrieval has relied on a set of crowd-sourced sentence level image caption evaluation datasets like Pascal1K (Rashtchian et al., 2010), Flickr8K (Hodosh et al., 2013), and Flickr30K (Young et al., 2014). However, these datasets are still relatively small and limit achievable performance. Several methods have been proposed to create larger datasets automatically with Ordonez et al. (2011) as a notable early example. In the deep learning era, Mithun et al. (2018) demonstrated an additional set of (image, text) pairs collected from the internet could improve retrieval performance and several new automatically constructed datasets such as Conceptual Captions (Sharma et al., 2018), LAIT (Qi et al., 2020), and OCR-CC (Yang et al., 2020) have been created. However, these datasets still use significantly more aggressive filtering or are designed for a specific task such as OCR and as a result are still much smaller than WIT with between 1 and 10 million training examples.


A related idea to CLIP is webly supervised learning. This line of work queries image search engines to build image datasets by querying for terms and uses the queries as the labels for the returned images (Fergus et al., 2005). Classifiers trained on these large but noisily labeled datasets can be competitive with those trained on smaller carefully labeled datasets. These image-query pairs are also often used to improve performance on standard datasets as additional training data (Chen & Gupta, 2015). CLIP also uses search queries as part of its dataset creation process. However CLIP only uses full text sequences co-occurring with images as supervision rather than just the queries, which are often only a single word or short n-gram. We also restrict this step in CLIP to text only querying for sub-string matches while most webly supervised work uses standard image search engines which have their own complex retrieval and filtering pipelines that often involve computer vision systems. Of this line of work, Learning Everything about Anything: Webly-Supervised Visual Concept Learning (Divvala et al., 2014) has a notably similar ambition and goal as CLIP.


Finally, CLIP is related to a recent burst of activity on learning joint models of vision and language (Lu et al., 2019; Tan


---


& Bansal, 2019; Chen et al., 2019; Li et al., 2020b; Yu et al., 2020). This line of work focuses on richly connecting vision and language in order to solve complex downstream tasks such as visual question answering, visual commonsense reasoning, or multimodal entailment. These approaches leverage impressively engineered models which combine 3 (or more) pre-trained subsystems, typically an image feature model, a region proposal / object detection model, and a pre-trained masked language model such as BERT. These systems are then jointly fine-tuned via various training objectives on image-text pairs and applied to the aforementioned tasks and achieve impressive results. CLIP is instead focused on learning visual models from scratch via natural language supervision and does not densely connect the two domains with a joint attention model. The only interaction in a CLIP model between the image and text domain is a single dot product in a learned joint embedding space. We are excited to see CLIP hybridized with this line of work.


# 9. Conclusion


We have investigated whether it is possible to transfer the success of task-agnostic web-scale pre-training in NLP to another domain. We find that adopting this formula results in similar behaviors emerging in the field of computer vision and discuss the social implications of this line of research. In order to optimize their training objective, CLIP models learn to perform a wide variety of tasks during pretraining. This task learning can then be leveraged via natural language prompting to enable zero-shot transfer to many existing datasets. At sufficient scale, the performance of this approach can be competitive with task-specific supervised models although there is still room for much improvement.


# ACKNOWLEDGMENTS


We'd like to thank the millions of people involved in creating the data CLIP is trained on. We'd also like to thank Susan Zhang for her work on image conditional language models while at OpenAI, Ishaan Gulrajani for catching an error in the pseudocode, and Irene Solaiman, Miles Brundage, and Gillian Hadfield for their thoughtful feedback on the broader impacts section of the paper. We are also grateful to the Acceleration and Supercomputing teams at OpenAI for their critical work on software and hardware infrastructure this project used. Finally, we'd also like to thank the developers of the many software packages used throughout this project including, but not limited, to Numpy (Harris et al., 2020), SciPy (Virtanen et al., 2020), ftfy (Speer, 2019), Tensor-Flow (Abadi et al., 2016), PyTorch (Paszke et al., 2019), pandas (pandas development team, 2020), and scikit-learn (Pedregosa et al., 2011).


# References


Abadi, M., Barham, P., Chen, J., Chen, Z., Davis, A., Dean, J., Devin, M., Ghemawat, S., Irving, G., Isard, M., et al. Tensorflow: A system for large-scale machine learning. In 12th {USENIX} symposium on operating systems design and implementation (\{OSDI\} 16), pp. 265-283, 2016.


Alayrac, J.-B., Recasens, A., Schneider, R., Arandjelović, R., Ramapuram, J., De Fauw, J., Smaira, L., Dieleman, S., and Zisserman, A. Self-supervised multimodal versatile networks. arXiv preprint arXiv:2006.16228, 2020.


Alcorn, M. A., Li, Q., Gong, Z., Wang, C., Mai, L., Ku, W.-S., and Nguyen, A. Strike (with) a pose: Neural networks are easily fooled by strange poses of familiar objects. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 4845-4854, 2019.


Andreas, J., Klein, D., and Levine, S. Learning with latent language. arXiv preprint arXiv:1711.00482, 2017.


Assiri, Y. Stochastic optimization of plain convolutional neural networks with simple methods. arXiv preprint arXiv:2001.08856, 2020.


Bachman, P., Hjelm, R. D., and Buchwalter, W. Learning representations by maximizing mutual information across views. In Advances in Neural Information Processing Systems, pp. 15535-15545, 2019.


Barbu, A., Mayo, D., Alverio, J., Luo, W., Wang, C., Gutfreund, D., Tenenbaum, J., and Katz, B. Objectnet: A large-scale bias-controlled dataset for pushing the limits of object recognition models. In Advances in Neural Information Processing Systems, pp. 9453-9463, 2019.


Barnard, K., Duygulu, P., Forsyth, D., Freitas, N. d., Blei, D. M., and Jordan, M. I. Matching words and pictures. Journal of machine learning research, 3(Feb):1107-1135, 2003.


Bechmann, A. and Bowker, G. C. Unsupervised by any other name: Hidden layers of knowledge production in artificial intelligence on social media. *Big Data & Society*, 6(1):205395171881956, January 2019. doi: 10.1177/2053951718819569. URL https://doi.org/10.1177/2053951718819569.


Bengio, Y., Ducharme, R., Vincent, P., and Jauvin, C. A neural probabilistic language model. Journal of machine learning research, 3(Feb):1137-1155, 2003.


Bhargava, S. and Forsyth, D. Exposing and correcting the gender bias in image captioning datasets and models. arXiv preprint arXiv:1912.00578, 2019.


---


Blei, D. M., Ng, A. Y., and Jordan, M. I. Latent dirichlet allocation. Journal of machine Learning research, 3(Jan): 993-1022, 2003.


Bolukbasi, T., Chang, K.-W., Zou, J. Y., Saligrama, V., and Kalai, A. T. Man is to computer programmer as woman is to homemaker? debiasing word embeddings. Advances in neural information processing systems, 29:4349-4357, 2016.


Bowker, G. C. and Star, S. L. Sorting things out: Classification and its consequences. MIT press, 2000.


Brown, T. B., Mann, B., Ryder, N., Subbiah, M., Kaplan, J., Dhariwal, P., Neelakantan, A., Shyam, P., Sastry, G., Askell, A., et al. Language models are few-shot learners. arXiv preprint arXiv:2005.14165, 2020.


Browne, S. Dark Matters: Surveillance of Blackness. Duke University Press, 2015.


Bulent Sariyildiz, M., Perez, J., and Larlus, D. Learning visual representations with caption annotations. arXiv e-prints, pp. arXiv-2008, 2020.


Buolamwini, J. and Gebru, T. Gender shades: Intersectional accuracy disparities in commercial gender classification. In Conference on fairness, accountability and transparency, pp. 77-91, 2018.


Carreira, J., Noland, E., Hillier, C., and Zisserman, A. A short note on the kinetics-700 human action dataset. arXiv preprint arXiv:1907.06987, 2019.


Chen, M., Radford, A., Child, R., Wu, J., Jun, H., Luan, D., and Sutskever, I. Generative pretraining from pixels. In International Conference on Machine Learning, pp. 1691-1703. PMLR, 2020a.


Chen, T., Xu, B., Zhang, C., and Guestrin, C. Training deep nets with sublinear memory cost. arXiv preprint arXiv:1604.06174, 2016.


Chen, T., Kornblith, S., Norouzi, M., and Hinton, G. A simple framework for contrastive learning of visual representations. arXiv preprint arXiv:2002.05709, 2020b.


Chen, T., Kornblith, S., Swersky, K., Norouzi, M., and Hinton, G. Big self-supervised models are strong semi-supervised learners. arXiv preprint arXiv:2006.10029, 2020c.


Chen, X. and Gupta, A. Webly supervised learning of convolutional networks. In Proceedings of the IEEE International Conference on Computer Vision, pp. 1431-1439, 2015.


Chen, X., Fan, H., Girshick, R., and He, K. Improved baselines with momentum contrastive learning. arXiv preprint arXiv:2003.04297, 2020d.


Chen, Y.-C., Li, L., Yu, L., Kholy, A. E., Ahmed, F., Gan, Z., Cheng, Y., and Liu, J. Uniter: Learning universal image-text representations. arXiv preprint arXiv:1909.11740, 2019.


Cheng, G., Han, J., and Lu, X. Remote sensing image scene classification: Benchmark and state of the art. Proceedings of the IEEE, 105(10):1865-1883, 2017.


Choi, D., Shallue, C. J., Nado, Z., Lee, J., Maddison, C. J., and Dahl, G. E. On empirical comparisons of optimizers for deep learning. arXiv preprint arXiv:1910.05446, 2019.


Coates, A., Ng, A., and Lee, H. An analysis of single-layer networks in unsupervised feature learning. In Proceedings of the fourteenth international conference on artificial intelligence and statistics, pp. 215-223, 2011.


Crawford, K. The trouble with bias. NIPS 2017 Keynote, 2017. URL https://www.youtube.com/watch?v=fMym_BKWQzk.


Dai, A. M. and Le, Q. V. Semi-supervised sequence learning. In Advances in neural information processing systems, pp. 3079-3087, 2015.


D’Amour, A., Heller, K., Moldovan, D., Adlam, B., Alipanahi, B., Beutel, A., Chen, C., Deaton, J., Eisenstein, J., Hoffman, M. D., et al. Underspecification presents challenges for credibility in modern machine learning. arXiv preprint arXiv:2011.03395, 2020.


Deng, J., Dong, W., Socher, R., Li, L.-J., Li, K., and Fei-Fei, L. ImageNet: A Large-Scale Hierarchical Image Database. In CVPR09, 2009.


Deng, J., Berg, A. C., Satheesh, S., Su, H., Khosla, A., and Fei-Fei, L. Ilsvrc 2012, 2012. URL http://www(image-net.org/challenges/LSVRC/2012/.


Desai, K. and Johnson, J. Virtex: Learning visual representations from textual annotations. arXiv preprint arXiv:2006.06666, 2020.


Devlin, J., Chang, M.-W., Lee, K., and Toutanova, K. Bert: Pre-training of deep bidirectional transformers for language understanding. arXiv preprint arXiv:1810.04805, 2018.


Dhariwal, P., Jun, H., Payne, C., Kim, J. W., Radford, A., and Sutskever, I. Jukebox: A generative model for music. arXiv preprint arXiv:2005.00341, 2020.


---


Divvala, S. K., Farhadi, A., and Guestrin, C. Learning everything about anything: Webly-supervised visual concept learning. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 3270-3277, 2014.


Dodge, S. and Karam, L. A study and comparison of human and deep learning recognition performance under visual distortions. In 2017 26th international conference on computer communication and networks (ICCCN), pp. 1-7. IEEE, 2017.


Dosovitskiy, A., Beyer, L., Kolesnikov, A., Weissenborn, D., Zhai, X., Unterthiner, T., Dehghani, M., Minderer, M., Heigold, G., Gelly, S., et al. An image is worth 16x16 words: Transformers for image recognition at scale. arXiv preprint arXiv:2010.11929, 2020.


Elhoseiny, M., Saleh, B., and Elgammal, A. Write a classifier: Zero-shot learning using purely textual descriptions. In Proceedings of the IEEE International Conference on Computer Vision, pp. 2584-2591, 2013.


Faghri, F., Fleet, D. J., Kiros, J. R., and Fidler, S. Vse++: Improving visual-semantic embeddings with hard negatives. arXiv preprint arXiv:1707.05612, 2017.


Fergus, R., Fei-Fei, L., Perona, P., and Zisserman, A. Learning object categories from google's image search. In Tenth IEEE International Conference on Computer Vision (ICCV'05) Volume 1, volume 2, pp. 1816-1823. IEEE, 2005.


Frome, A., Corrado, G. S., Shlens, J., Bengio, S., Dean, J., Ranzato, M., and Mikolov, T. Devise: A deep visual-semantic embedding model. In Advances in neural information processing systems, pp. 2121-2129, 2013.


Gan, Z., Chen, Y.-C., Li, L., Zhu, C., Cheng, Y., and Liu, J. Large-scale adversarial training for vision-and-language representation learning. arXiv preprint arXiv:2006.06195, 2020.


Gao, T., Fisch, A., and Chen, D. Making pre-trained language models better few-shot learners. arXiv preprint arXiv:2012.15723, 2020.


Garvie, C., May 2019. URL https://www.flawedfacedata.com/.


Geiger, A., Lenz, P., and Urtasun, R. Are we ready for autonomous driving? the kitti vision benchmark suite. In Conference on Computer Vision and Pattern Recognition (CVPR), 2012.


Geirhos, R., Rubisch, P., Michaelis, C., Bethge, M., Wichmann, F. A., and Brendel, W. Imagenet-trained cnns are


biased towards texture; increasing shape bias improves accuracy and robustness. arXiv preprint arXiv:1811.12231, 2018.


Geirhos, R., Jacobsen, J.-H., Michaelis, C., Zemel, R., Brendel, W., Bethge, M., and Wichmann, F. A. Short-cut learning in deep neural networks. arXiv preprint arXiv:2004.07780, 2020.


Gomez, L., Patel, Y., Rusinol, M., Karatzas, D., and Jawahar, C. Self-supervised learning of visual features through embedding images into text topic spaces. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 4230-4239, 2017.


Goodfellow, I. J., Shlens, J., and Szegedy, C. Explaining and harnessing adversarial examples. arXiv preprint arXiv:1412.6572, 2014.


Goodfellow, I. J., Erhan, D., Carrier, P. L., Courville, A., Mirza, M., Hamner, B., Cukierski, W., Tang, Y., Thaler, D., Lee, D.-H., et al. Challenges in representation learning: A report on three machine learning contests. *Neural Networks*, 64:59–63, 2015.


Google. Google cloud api: Celebrity recognition. URL https://cloud.google.com/vision/docs/ celebrity-recognition.


Griewank, A. and Walther, A. Algorithm 799: revolve: an implementation of checkpointing for the reverse or adjoint mode of computational differentiation. ACM Transactions on Mathematical Software (TOMS), 26(1):19-45, 2000.


Grill, J.-B., Strub, F., Altché, F., Tallec, C., Richemond, P. H., Buchatskaya, E., Doersch, C., Pires, B. A., Guo, Z. D., Azar, M. G., et al. Bootstrap your own latent: A new approach to self-supervised learning. arXiv preprint arXiv:2006.07733, 2020.


Ha, D., Dai, A., and Le, Q. V. Hypernetworks. arXiv preprint arXiv:1609.09106, 2016.


Hancock, B., Bringmann, M., Varma, P., Liang, P., Wang, S., and Ré, C. Training classifiers with natural language explanations. In Proceedings of the conference. Association for Computational Linguistics. Meeting, volume 2018, pp. 1884. NIH Public Access, 2018.


Hancock, B., Bordes, A., Mazare, P.-E., and Weston, J. Learning from dialogue after deployment: Feed yourself, chatbot! arXiv preprint arXiv:1901.05415, 2019.


Harris, C. R., Millman, K. J., van der Walt, S. J., Gommers, R., Virtanen, P., Cournapeau, D., Wieser, E., Taylor, J., Berg, S., Smith, N. J., Kern, R., Picus, M., Hoyer, S., van Kerkwijk, M. H., Brett, M., Haldane, A., Fernandez del


---


Río, J., Wiebe, M., Peterson, P., Gérard-Marchant, P., Sheppard, K., Reddy, T., Weckesser, W., Abbasi, H., Gohlke, C., and Oliphant, T. E. Array programming with NumPy. Nature, 585:357-362, 2020. doi: 10.1038/s41586-020-2649-2.


Hays, J. and Efros, A. A. Im2gps: estimating geographic information from a single image. In 2008 *ieee conference on computer vision and pattern recognition*, pp. 1-8. IEEE, 2008.


He, K., Zhang, X., Ren, S., and Sun, J. Delving deep into rectifiers: Surpassing human-level performance on imagenet classification. In Proceedings of the IEEE international conference on computer vision, pp. 1026-1034, 2015.


He, K., Zhang, X., Ren, S., and Sun, J. Deep residual learning for image recognition. In Proceedings of the IEEE conference on computer vision and pattern recognition, pp. 770-778, 2016a.


He, K., Zhang, X., Ren, S., and Sun, J. Deep residual learning for image recognition. In Proceedings of the IEEE conference on computer vision and pattern recognition, pp. 770-778, 2016b.


He, K., Fan, H., Wu, Y., Xie, S., and Girshick, R. Momentum contrast for unsupervised visual representation learning. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 9729-9738, 2020.


He, T., Zhang, Z., Zhang, H., Zhang, Z., Xie, J., and Li, M. Bag of tricks for image classification with convolutional neural networks. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 558-567, 2019.


He, X. and Peng, Y. Fine-grained image classification via combining vision and language. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 5994-6002, 2017.


Helber, P., Bischke, B., Dengel, A., and Borth, D. Eurosat: A novel dataset and deep learning benchmark for land use and land cover classification. IEEE Journal of Selected Topics in Applied Earth Observations and Remote Sensing, 12(7):2217-2226, 2019.


Henaff, O. Data-efficient image recognition with contrastive predictive coding. In International Conference on Machine Learning, pp. 4182-4192. PMLR, 2020.


Hendrycks, D. and Dietterich, T. Benchmarking neural network robustness to common corruptions and perturbations. arXiv preprint arXiv:1903.12261, 2019.


Hendrycks, D. and Gimpel, K. Gaussian error linear units (gelus). arXiv preprint arXiv:1606.08415, 2016.


Hendrycks, D., Zhao, K., Basart, S., Steinhardt, J., and Song, D. Natural adversarial examples. arXiv preprint arXiv:1907.07174, 2019.


Hendrycks, D., Basart, S., Mu, N., Kadavath, S., Wang, F., Dorundo, E., Desai, R., Zhu, T., Parajuli, S., Guo, M., et al. The many faces of robustness: A critical analysis of out-of-distribution generalization. arXiv preprint arXiv:2006.16241, 2020a.


Hendrycks, D., Liu, X., Wallace, E., Dziedzic, A., Krishnan, R., and Song, D. Pretrained transformers improve out-of-distribution robustness. arXiv preprint arXiv:2004.06100, 2020b.


Hestness, J., Narang, S., Ardalani, N., Diamos, G., Jun, H., Kianinejad, H., Patwary, M., Ali, M., Yang, Y., and Zhou, Y. Deep learning scaling is predictable, empirically. arXiv preprint arXiv:1712.00409, 2017.


Hill, F., Lampinen, A., Schneider, R., Clark, S., Botvinick, M., McClelland, J. L., and Santoro, A. Environmental drivers of systematicity and generalization in a situated agent. In International Conference on Learning Representations, 2019.


Hodosh, M., Young, P., and Hockenmaier, J. Framing image description as a ranking task: Data, models and evaluation metrics. Journal of Artificial Intelligence Research, 47: 853-899, 2013.


Hongsuck Seo, P., Weyand, T., Sim, J., and Han, B. Cplanet: Enhancing image geolocation by combinatorial partitioning of maps. In Proceedings of the European Conference on Computer Vision (ECCV), pp. 536-551, 2018.


Howard, J. and Ruder, S. Universal language model fine-tuning for text classification. arXiv preprint arXiv:1801.06146, 2018.


Ilyas, A., Santurkar, S., Tsipras, D., Engstrom, L., Tran, B., and Madry, A. Adversarial examples are not bugs, they are features. In Advances in Neural Information Processing Systems, pp. 125-136, 2019.


Ioffe, S. and Szegedy, C. Batch normalization: Accelerating deep network training by reducing internal covariate shift. arXiv preprint arXiv:1502.03167, 2015.


Jaderberg, M., Simonyan, K., Vedaldi, A., and Zisserman, A. Deep structured output learning for unconstrained text recognition. arXiv preprint arXiv:1412.5903, 2014.


Jaderberg, M., Simonyan, K., Zisserman, A., et al. Spatial transformer networks. Advances in neural information processing systems, 28:2017-2025, 2015.


---


Johnson, J., Hariharan, B., van der Maaten, L., Fei-Fei, L., Lawrence Zitnick, C., and Girshick, R. Clevr: A diagnostic dataset for compositional language and elementary visual reasoning. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 2901-2910, 2017.


Joulin, A., Van Der Maaten, L., Jabri, A., and Vasilache, N. Learning visual features from large weakly supervised data. In European Conference on Computer Vision, pp. 67-84. Springer, 2016.


Kalfaoglu, M., Kalkan, S., and Alatan, A. A. Late temporal modeling in 3d cnn architectures with bert for action recognition. arXiv preprint arXiv:2008.01232, 2020.


Kaplan, J., McCandlish, S., Henighan, T., Brown, T. B., Chess, B., Child, R., Gray, S., Radford, A., Wu, J., and Amodei, D. Scaling laws for neural language models. arXiv preprint arXiv:2001.08361, 2020.


Karpathy, A., Joulin, A., and Fei-Fei, L. F. Deep fragment embeddings for bidirectional image sentence mapping. In Advances in neural information processing systems, pp. 1889–1897, 2014.


Keyes, O. The misgendering machines: Trans/hci implications of automatic gender recognition. Proceedings of the ACM on Human-Computer Interaction, 2(CSCW):1-22, 2018.


Kiela, D., Firooz, H., Mohan, A., Goswami, V., Singh, A., Ringshia, P., and Testuggine, D. The hateful memes challenge: Detecting hate speech in multimodal memes. arXiv preprint arXiv:2005.04790, 2020.


Kingma, D. P. and Ba, J. Adam: A method for stochastic optimization. arXiv preprint arXiv:1412.6980, 2014.


Kiros, R., Salakhutdinov, R., and Zemel, R. S. Unifying visual-semantic embeddings with multimodal neural language models. arXiv preprint arXiv:1411.2539, 2014.


Kiros, R., Zhu, Y., Salakhutdinov, R. R., Zemel, R., Urtasun, R., Torralba, A., and Fidler, S. Skip-thought vectors. Advances in neural information processing systems, 28: 3294-3302, 2015.


Kolesnikov, A., Beyer, L., Zhai, X., Puigcerver, J., Yung, J., Gelly, S., and Houlsby, N. Large scale learning of general visual representations for transfer. arXiv preprint arXiv:1912.11370, 2019.


Kornblith, S., Shlens, J., and Le, Q. V. Do better imagenet models transfer better? In Proceedings of the IEEE conference on computer vision and pattern recognition, pp. 2661-2671, 2019.


Krishna, R., Zhu, Y., Groth, O., Johnson, J., Hata, K., Kravitz, J., Chen, S., Kalantidis, Y., Li, L.-J., Shamma, D. A., et al. Visual genome: Connecting language and vision using crowdsourced dense image annotations. International journal of computer vision, 123(1):32-73, 2017.


Krizhevsky, A., Sutskever, I., and Hinton, G. E. Imagenet classification with deep convolutional neural networks. In Advances in neural information processing systems, pp. 1097-1105, 2012.


Kuhnle, A. and Copestake, A. Shapeworld-a new test methodology for multimodal language understanding. arXiv preprint arXiv:1704.04517, 2017.


Kärkkäinen, K. and Joo, J. Fairface: Face attribute dataset for balanced race, gender, and age, 2019.


Lake, B. M., Ullman, T. D., Tenenbaum, J. B., and Gershman, S. J. Building machines that learn and think like people, 2016.


Lampert, C. H., Nickisch, H., and Harmeling, S. Learning to detect unseen object classes by between-class attribute transfer. In 2009 IEEE Conference on Computer Vision and Pattern Recognition, pp. 951-958. IEEE, 2009.


Larochelle, H., Erhan, D., and Bengio, Y. Zero-data learning of new tasks. 2008.


Le, Q. and Mikolov, T. Distributed representations of sentences and documents. In International conference on machine learning, pp. 1188-1196, 2014.


LeCun, Y. The mnist database of handwritten digits. http://yann.lecun.com/exdb/mnist/.


Lee, D.-H. Pseudo-label: The simple and efficient semi-supervised learning method for deep neural networks.


Lei Ba, J., Swersky, K., Fidler, S., et al. Predicting deep zero-shot convolutional neural networks using textual descriptions. In Proceedings of the IEEE International Conference on Computer Vision, pp. 4247-4255, 2015.


Li, A., Jabri, A., Joulin, A., and van der Maaten, L. Learning visual n-grams from web data. In Proceedings of the IEEE International Conference on Computer Vision, pp. 4183-4192, 2017.


Li, G., Duan, N., Fang, Y., Gong, M., and Jiang, D. Unicoder-vl: A universal encoder for vision and language by cross-modal pre-training. 2020a.


Li, J., Miller, A. H., Chopra, S., Ranzato, M., and Weston, J. Learning through dialogue interactions by asking questions. arXiv preprint arXiv:1612.04936, 2016.


---


Li, X., Yin, X., Li, C., Hu, X., Zhang, P., Zhang, L., Wang, L., Hu, H., Dong, L., Wei, F., et al. Oscar: Object-semantics aligned pre-training for vision-language tasks. arXiv preprint arXiv:2004.06165, 2020b.


Liang, W., Zou, J., and Yu, Z. Alice: Active learning with contrastive natural language explanations. arXiv preprint arXiv:2009.10259, 2020.


Lin, T.-Y., Maire, M., Belongie, S., Hays, J., Perona, P., Ramanan, D., Dollar, P., and Zitnick, C. L. Microsoft coco: Common objects in context. In European conference on computer vision, pp. 740-755. Springer, 2014.


Linzen, T. How can we accelerate progress towards human-like linguistic generalization? arXiv preprint arXiv:2005.00955, 2020.


Lippe, P., Holla, N., Chandra, S., Rajamanickam, S., Antoniou, G., Shutova, E., and Yannakoudakis, H. A multimodal framework for the detection of hateful memes. arXiv preprint arXiv:2012.12871, 2020.


Liu, P. J., Saleh, M., Pot, E., Goodrich, B., Sepassi, R., Kaiser, L., and Shazeer, N. Generating wikipedia by summarizing long sequences. arXiv preprint arXiv:1801.10198, 2018.


Locatello, F., Bauer, S., Lucic, M., Ratsch, G., Gelly, S., Schölkopf, B., and Bachem, O. A sober look at the unsupervised learning of disentangled representations and their evaluation. arXiv preprint arXiv:2010.14766, 2020.


Loshchilov, I. and Hutter, F. Sgdr: Stochastic gradient descent with warm restarts. arXiv preprint arXiv:1608.03983, 2016.


Loshchilov, I. and Hutter, F. Decoupled weight decay regularization. arXiv preprint arXiv:1711.05101, 2017.


Lu, J., Batra, D., Parikh, D., and Lee, S. Vilbert: Pretraining task-agnostic visiolinguistic representations for vision- and language tasks. In Advances in Neural Information Processing Systems, pp. 13-23, 2019.


Lu, Z., Xiong, X., Li, Y., Stroud, J., and Ross, D. Leveraging weakly supervised data and pose representation for action recognition, 2020. URL https://www.youtube.com/watch?v=KOQFxbPPLOE&t=1390s.


Lucic, M., Kurach, K., Michalski, M., Gelly, S., and Bousquet, O. Are gans created equal? a large-scale study. Advances in neural information processing systems, 31: 700-709, 2018.


Mahajan, D., Girshick, R., Ramanathan, V., He, K., Paluri, M., Li, Y., Bharambe, A., and van der Maaten, L. Exploring the limits of weakly supervised pretraining. In


Proceedings of the European Conference on Computer Vision (ECCV), pp. 181-196, 2018.


McCann, B., Bradbury, J., Xiong, C., and Socher, R. Learned in translation: Contextualized word vectors. In Advances in neural information processing systems, pp. 6294-6305, 2017.


McCann, B., Keskar, N. S., Xiong, C., and Socher, R. The natural language decathlon: Multitask learning as question answering. arXiv preprint arXiv:1806.08730, 2018.


Micikevicius, P., Narang, S., Alben, J., Diamos, G., Elsen, E., Garcia, D., Ginsburg, B., Houston, M., Kuchaiev, O., Venkatesh, G., et al. Mixed precision training. arXiv preprint arXiv:1710.03740, 2017.


Miech, A., Zhukov, D., Alayrac, J.-B., Tapaswi, M., Laptev, I., and Sivic, J. Howto100m: Learning a text-video embedding by watching hundred million narrated video clips. In Proceedings of the IEEE international conference on computer vision, pp. 2630-2640, 2019.


Miech, A., Alayrac, J.-B., Laptev, I., Sivic, J., and Zisserman, A. Rareact: A video dataset of unusual interactions. arXiv preprint arXiv:2008.01018, 2020a.


Miech, A., Alayrac, J.-B., Smaira, L., Laptev, I., Sivic, J., and Zisserman, A. End-to-end learning of visual representations from uncurated instructional videos. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 9879-9889, 2020b.


Mikolov, T., Sutskever, I., Chen, K., Corrado, G. S., and Dean, J. Distributed representations of words and phrases and their compositionality. Advances in neural information processing systems, 26:3111-3119, 2013.


Miller, J., Krauth, K., Recht, B., and Schmidt, L. The effect of natural distribution shift on question answering models. arXiv preprint arXiv:2004.14444, 2020.


Mishra, A., Alahari, K., and Jawahar, C. Scene text recognition using higher order language priors. 2012.


Mithun, N. C., Panda, R., Papalexakis, E. E., and Roy-Chowdhury, A. K. Webly supervised joint embedding for cross-modal image-text retrieval. In Proceedings of the 26th ACM international conference on Multimedia, pp. 1856-1864, 2018.


Mori, Y., Takahashi, H., and Oka, R. Image-to-word transformation based on dividing and vector quantizing images with words. CiteSeer, 1999.


Mu, J., Liang, P., and Goodman, N. Shaping visual representations with language for few-shot classification. arXiv preprint arXiv:1911.02683, 2019.


---


Muller-Budack, E., Pustu-Iren, K., and Ewerth, R. Geolocation estimation of photos using a hierarchical model and scene classification. In Proceedings of the European Conference on Computer Vision (ECCV), pp. 563-579, 2018.


Murty, S., Koh, P. W., and Liang, P. Expbert: Representation engineering with natural language explanations. arXiv preprint arXiv:2005.01932, 2020.


Narasimhan, K., Kulkarni, T., and Barzilay, R. Language understanding for text-based games using deep reinforcement learning. arXiv preprint arXiv:1506.08941, 2015.


Netzer, Y., Wang, T., Coates, A., Bissacco, A., Wu, B., and Ng, A. Y. Reading digits in natural images with unsupervised feature learning. 2011.


Noble, S. U. Algorithms of oppression: How search engines reinforce racism. 2018.


Nosek, B. A., Banaji, M. R., and Greenwald, A. G. Harvesting implicit group attitudes and beliefs from a demonstration web site. Group Dynamics: Theory, Research, and Practice, 6(1):101, 2002.


Oh, S., Hoogs, A., Perera, A., Cunteor, N., Chen, C.-C., Lee, J. T., Mukherjee, S., Aggarwal, J., Lee, H., Davis, L., et al. A large-scale benchmark dataset for event recognition in surveillance video. In CVPR 2011, pp. 3153-3160. IEEE, 2011.


Oliver, A., Odena, A., Raffel, C. A., Cubuk, E. D., and Goodfellow, I. Realistic evaluation of deep semi-supervised learning algorithms. Advances in neural information processing systems, 31:3235-3246, 2018.


Oord, A. v. d., Li, Y., and Vinyals, O. Representation learning with contrastive predictive coding. arXiv preprint arXiv:1807.03748, 2018.


Ordonez, V., Kulkarni, G., and Berg, T. Im2text: Describing images using 1 million captioned photographs. Advances in neural information processing systems, 24:1143-1151, 2011.


pandas development team, T. pandas-dev/pandas: Pandas, February 2020. URL https://doi.org/10.5281/zenodo.3509134.


Parkhi, O. M., Vedaldi, A., Zisserman, A., and Jawahar, C. V. Cats and dogs. In IEEE Conference on Computer Vision and Pattern Recognition, 2012.


Paszke, A., Gross, S., Massa, F., Lerer, A., Bradbury, J., Chanan, G., Killeen, T., Lin, Z., Gimelshein, N., Antiga, L., Desmaison, A., Kopf, A., Yang, E., DeVito, Z., Raison, M., Tejani, A., Chilamkurthy, S., Steiner, B., Fang, L.,


Bai, J., and Chintala, S. Pytorch: An imperative style, high-performance deep learning library. In Advances in Neural Information Processing Systems 32, pp. 8024-8035, 2019.


Pedregosa, F., Varoquaux, G., Gramfort, A., Michel, V., Thirion, B., Grisel, O., Blondel, M., Prettenhofer, P., Weiss, R., Dubourg, V., Vanderplas, J., Passos, A., Cournaepau, D., Brucher, M., Perrot, M., and Duchesnay, E. Scikit-learn: Machine learning in Python. Journal of Machine Learning Research, 12:2825-2830, 2011.


Pennington, J., Socher, R., and Manning, C. D. Glove: Global vectors for word representation. In Proceedings of the 2014 conference on empirical methods in natural language processing (EMNLP), pp. 1532-1543, 2014.


Peters, M. E., Neumann, M., Iyyer, M., Gardner, M., Clark, C., Lee, K., and Zettlemoyer, L. Deep contextualized word representations. arXiv preprint arXiv:1802.05365, 2018.


Qi, D., Su, L., Song, J., Cui, E., Bharti, T., and Sacheti, A. Imagebert: Cross-modal pre-training with large-scale weak-supervised image-text data. arXiv preprint arXiv:2001.07966, 2020.


Quattoni, A., Collins, M., and Darrell, T. Learning visual representations using images with captions. In 2007 IEEE Conference on Computer Vision and Pattern Recognition, pp. 1-8. IEEE, 2007.


Radford, A., Narasimhan, K., Salimans, T., and Sutskever, I. Improving language understanding by generative pretraining, 2018.


Radford, A., Wu, J., Child, R., Luan, D., Amodei, D., and Sutskever, I. Language models are unsupervised multitask learners. 2019.


Raffel, C., Shazeer, N., Roberts, A., Lee, K., Narang, S., Matena, M., Zhou, Y., Li, W., and Liu, P. J. Exploring the limits of transfer learning with a unified text-to-text transformer. arXiv preprint arXiv:1910.10683, 2019.


Raji, I. D., Gebru, T., Mitchell, M., Buolamwini, J., Lee, J., and Denton, E. Saving face: Investigating the ethical concerns of facial recognition auditing, 2020.


Ramanathan, V., Liang, P., and Fei-Fei, L. Video event understanding using natural language descriptions. In Proceedings of the IEEE International Conference on Computer Vision, pp. 905–912, 2013.


Rashtchian, C., Young, P., Hodosh, M., and Hockenmaier, J. Collecting image annotations using amazon's mechanical turk. In Proceedings of the NAACL HLT 2010 Workshop on Creating Speech and Language Data with Amazon's Mechanical Turk, pp. 139-147, 2010.


---


Recht, B., Roelofs, R., Schmidt, L., and Shankar, V. Do imagelet classifiers generalize to imagelet? arXiv preprint arXiv:1902.10811, 2019.


Salimans, T. and Kingma, D. P. Weight normalization: A simple reparameterization to accelerate training of deep neural networks. In Advances in neural information processing systems, pp. 901-909, 2016.


Scheuerman, M. K., Paul, J. M., and Brubaker, J. R. How computers see gender: An evaluation of gender classification in commercial facial analysis services. Proceedings of the ACM on Human-Computer Interaction, 3(CSCW): 1-33, 2019.


Schwemmer, C., Knight, C., Bello-Pardo, E. D., Oklobdzija, S., Schoonvelde, M., and Lockhart, J. W. Diagnosing gender bias in image recognition systems. *Socius*, 6: 2378023120967171, 2020.


Sennrich, R., Haddow, B., and Birch, A. Neural machine translation of rare words with subword units. arXiv preprint arXiv:1508.07909, 2015.


Shankar, V., Dave, A., Roelofs, R., Ramanan, D., Recht, B., and Schmidt, L. Do image classifiers generalize across time? arXiv preprint arXiv:1906.02168, 2019.


Sharma, P., Ding, N., Goodman, S., and Soricut, R. Conceptual captions: A cleaned, hypernymed, image alt-text dataset for automatic image captioning. In Proceedings of the 56th Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pp. 2556-2565, 2018.


Singh, A., Natarajan, V., Shah, M., Jiang, Y., Chen, X., Batra, D., Parikh, D., and Rohrbach, M. Towards vqa models that can read. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition, pp. 8317-8326, 2019.


Socher, R. and Fei-Fei, L. Connecting modalities: Semi-supervised segmentation and annotation of images using unaligned text corpora. In 2010 IEEE Computer Society Conference on Computer Vision and Pattern Recognition, pp. 966–973. IEEE, 2010.


Socher, R., Perelygin, A., Wu, J., Chuang, J., Manning, C. D., Ng, A. Y., and Potts, C. Recursive deep models for semantic compositionality over a sentiment treebank. In Proceedings of the 2013 conference on empirical methods in natural language processing, pp. 1631-1642, 2013.


Socher, R., Karpathy, A., Le, Q. V., Manning, C. D., and Ng, A. Y. Grounded compositional semantics for finding and describing images with sentences. Transactions of the Association for Computational Linguistics, 2:207-218, 2014.


Sohn, K. Improved deep metric learning with multi-class n-pair loss objective. In Advances in neural information processing systems, pp. 1857-1865, 2016.


Solaiman, I., Brundage, M., Clark, J., Askell, A., Herbert-Voss, A., Wu, J., Radford, A., Krueger, G., Kim, J. W., Kreps, S., McCain, M., Newhouse, A., Blazakis, J., McGuffie, K., and Wang, J. Release strategies and the social impacts of language models, 2019.


Soomro, K., Zamir, A. R., and Shah, M. Ucf101: A dataset of 101 human actions classes from videos in the wild. arXiv preprint arXiv:1212.0402, 2012.


Speer, R. ftfy. Zenodo, 2019. URL https://doi.org/10.5281/zenodo.2591652. Version 5.5.


Srivastava, N. and Salakhutdinov, R. Multimodal learning with deep boltzmann machines. In NIPS, 2012.


Srivastava, S., Labutov, I., and Mitchell, T. Joint concept learning and semantic parsing from natural language explanations. In Proceedings of the 2017 conference on empirical methods in natural language processing, pp. 1527-1536, 2017.


Stallkamp, J., Schlipsing, M., Salmen, J., and Igel, C. The German Traffic Sign Recognition Benchmark: A multiclass classification competition. In IEEE International Joint Conference on Neural Networks, pp. 1453-1460, 2011.


Stroud, J. C., Ross, D. A., Sun, C., Deng, J., Sukthankar, R., and Schmid, C. Learning video representations from textual web supervision. arXiv preprint arXiv:2007.14937, 2020.


Szegedy, C., Ioffe, S., Vanhoucke, V., and Alemi, A. Inception-v4, inception-resnet and the impact of residual connections on learning. arXiv preprint arXiv:1602.07261, 2016.


Tan, H. and Bansal, M. Lxmert: Learning cross-modality encoder representations from transformers. arXiv preprint arXiv:1908.07490, 2019.


Tan, M. and Le, Q. V. Efficientnet: Rethinking model scaling for convolutional neural networks. arXiv preprint arXiv:1905.11946, 2019.


Taori, R., Dave, A., Shankar, V., Carlini, N., Recht, B., and Schmidt, L. Measuring robustness to natural distribution shifts in image classification. arXiv preprint arXiv:2007.00644, 2020.


Thomee, B., Shamma, D. A., Friedland, G., Elizalde, B., Ni, K., Poland, D., Borth, D., and Li, L.-J. Yfcc100m: The new data in multimedia research. Communications of the ACM, 59(2):64-73, 2016.


---


Tian, Y., Krishnan, D., and Isola, P. Contrastive multiview coding. arXiv preprint arXiv:1906.05849, 2019.


Tian, Y., Wang, Y., Krishnan, D., Tenenbaum, J. B., and Isola, P. Rethinking few-shot image classification: a good embedding is all you need? arXiv preprint arXiv:2003.11539, 2020.


Torralba, A., Fergus, R., and Freeman, W. T. 80 million tiny images: A large data set for nonparametric object and scene recognition. IEEE transactions on pattern analysis and machine intelligence, 30(11):1958-1970, 2008.


Touvron, H., Vedaldi, A., Douze, M., and Jégou, H. Fixing the train-test resolution discrepancy. In Advances in neural information processing systems, pp. 8252-8262, 2019.


Varadarajan, J. and Odobez, J.-M. Topic models for scene analysis and abnormality detection. In 2009 IEEE 12th International Conference on Computer Vision Workshops, ICCV Workshops, pp. 1338-1345. IEEE, 2009.


Vaswani, A., Shazeer, N., Parmar, N., Uszkoreit, J., Jones, L., Gomez, A. N., Kaiser, L., and Polosukhin, I. Attention is all you need. In Advances in neural information processing systems, pp. 5998-6008, 2017.


Veeling, B. S., Linmans, J., Winkens, J., Cohen, T., and Welling, M. Rotation equivariant CNNs for digital pathology. June 2018.


Virtanen, P., Gommers, R., Oliphant, T. E., Haberland, M., Reddy, T., Cournapeau, D., Burovski, E., Peterson, P., Weckesser, W., Bright, J., van der Walt, S. J., Brett, M., Wilson, J., Millman, K. J., Mayorov, N., Nelson, A. R. J., Jones, E., Kern, R., Larson, E., Carey, C. J., Polat, I., Feng, Y., Moore, E. W., VanderPlas, J., Laxalde, D., Perktold, J., Cimrman, R., Henriksen, I., Quintero, E. A., Harris, C. R., Archibald, A. M., Ribeiro, A. H., Pedregosa, F., van Mulbregt, P., and SciPy 1.0 Contributors. SciPy 1.0: Fundamental Algorithms for Scientific Computing in Python. Nature Methods, 17:261-272, 2020. doi: 10.1038/s41592-019-0686-2.


Vo, N., Jacobs, N., and Hays, J. Revisiting im2gps in the deep learning era. In Proceedings of the IEEE International Conference on Computer Vision, pp. 2621-2630, 2017.


Wang, A., Singh, A., Michael, J., Hill, F., Levy, O., and Bowman, S. R. Glue: A multi-task benchmark and analysis platform for natural language understanding. arXiv preprint arXiv:1804.07461, 2018.


Wang, H., Ge, S., Lipton, Z., and Xing, E. P. Learning robust global representations by penalizing local predictive power. In Advances in Neural Information Processing Systems, pp. 10506-10518, 2019.


Wang, H., Lu, P., Zhang, H., Yang, M., Bai, X., Xu, Y., He, M., Wang, Y., and Liu, W. All you need is boundary: Toward arbitrary-shaped text spotting. In Proceedings of the AAAI Conference on Artificial Intelligence, volume 34, pp. 12160-12167, 2020.


Wang, J., Markert, K., and Everingham, M. Learning models for object recognition from natural language descriptions. In BMVC, volume 1, pp. 2, 2009.


Weston, J., Bengio, S., and Usunier, N. Large scale image annotation: learning to rank with joint word-image embeddings. Machine learning, 81(1):21-35, 2010.


Weston, J. E. Dialog-based language learning. In Advances in Neural Information Processing Systems, pp. 829-837, 2016.


Weyand, T., Kostrikov, I., and Philbin, J. Planet-photo geolocation with convolutional neural networks. In European Conference on Computer Vision, pp. 37-55. Springer, 2016.


Wu, Y., Kirillov, A., Massa, F., Lo, W.-Y., and Girshick, R. Detector2. https://github.com/facebookresearch/detectron2, 2019.


Wu, Z., Xiong, Y., Yu, S., and Lin, D. Unsupervised feature learning via non-parametric instance-level discrimination. arXiv preprint arXiv:1805.01978, 2018.


Xie, Q., Luong, M.-T., Hovy, E., and Le, Q. V. Self-training with noisy student improves imagenet classification. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 10687-10698, 2020.


y Arcas, B. A., Mitchell, M., and Todorov, A. Physiognomy's new clothes. 2017. URL https://medium.com/@blaisea/physiognomys-new-clothes-f2d4b59fdd6a.


Yang, Z., Lu, Y., Wang, J., Yin, X., Florencio, D., Wang, L., Zhang, C., Zhang, L., and Luo, J. Tap: Text-aware pre-training for text-vqa and text-caption. arXiv preprint arXiv:2012.04638, 2020.


Yogatama, D., d'Autume, C. d. M., Connor, J., Kocisky, T., Chrzanowski, M., Kong, L., Lazaridou, A., Ling, W., Yu, L., Dyer, C., et al. Learning and evaluating general linguistic intelligence. arXiv preprint arXiv:1901.11373, 2019.


Young, P., Lai, A., Hodosh, M., and Hockenmaier, J. From image descriptions to visual denotations: New similarity metrics for semantic inference over event descriptions. Transactions of the Association for Computational Linguistics, 2:67-78, 2014.


---


Yu, F., Tang, J., Yin, W., Sun, Y., Tian, H., Wu, H., and Wang, H. Ernie-vil: Knowledge enhanced vision-language representations through scene graph. arXiv preprint arXiv:2006.16934, 2020.


Zeiler, M. D. and Fergus, R. Visualizing and understanding convolutional networks. In European conference on computer vision, pp. 818-833. Springer, 2014.


Zhai, X., Puigcerver, J., Kolesnikov, A., Ruyssen, P., Riquelme, C., Lucic, M., Djolonga, J., Pinto, A. S., Neumann, M., Dosovitskiy, A., et al. A large-scale study of representation learning with the visual task adaptation benchmark. arXiv preprint arXiv:1910.04867, 2019.


Zhang, R. Making convolutional networks shift-invariant again. arXiv preprint arXiv:1904.11486, 2019.


Zhang, Y., Jiang, H., Miura, Y., Manning, C. D., and Langlotz, C. P. Contrastive learning of medical visual representations from paired images and text. arXiv preprint arXiv:2010.00747, 2020.


Zuboff, S. Big other: surveillance capitalism and the prospects of an information civilization. Journal of Information Technology, 30(1):75-89, 2015.


---


# A. Linear-probe evaluation


We provide additional details for linear probe experiments presented in this paper, including the list of the datasets and models used for evaluation.


# A.1. Datasets


We use the 12 datasets from the well-studied evaluation suite introduced by (Kornblith et al., 2019) and add 15 additional datasets in order to assess the performance of models on a wider variety of distributions and tasks. These datasets include MNIST, the Facial Expression Recognition 2013 dataset (Goodfellow et al., 2015), STL-10 (Coates et al., 2011), EuroSAT (Helber et al., 2019), the NWPU-RESISC45 dataset (Cheng et al., 2017), the German Traffic Sign Recognition Benchmark (GTSRB) dataset (Stallkamp et al., 2011), the KITTI dataset (Geiger et al., 2012), PatchCamelyon (Veeling et al., 2018), the UCF101 action recognition dataset (Soomro et al., 2012), Kinetics 700 (Carreira et al., 2019), 2,500 random samples of the CLEVR dataset (Johnson et al., 2017), the Hateful Memes dataset (Kiela et al., 2020), and the ImageNet-1k dataset (Deng et al., 2012). For the two video datasets (UCF101 and Kinetics700), we use the middle frame of each video clip as the input image. STL-10 and UCF101 have multiple predefined train/validation/test splits, 10 and 3 respectively, and we report the average over all splits. Details on each dataset and the corresponding evaluation metrics are provided in Table 9.


Additionally, we created two datasets that we call Country211 and Rendered SST2. The Country211 dataset is designed to assess the geolocation capability of visual representations. We filtered the YFCC100m dataset (Thomee et al., 2016) to find 211 countries (defined as having an ISO-3166 country code) that have at least 300 photos with GPS coordinates, and we built a balanced dataset with 211 categories, by sampling 200 photos for training and 100 photos for testing, for each country.


The Rendered SST2 dataset is designed to measure the optical character recognition capability of visual representations. To do so, we used the sentences from the Stanford Sentiment Treebank dataset (Socher et al., 2013) and rendered them into images, with black texts on a white background, in a $448 \times 448$ resolution. Two example images from this dataset are shown in Figure 19.


# A.2. Models


In combination with the datasets listed above, we evaluate the following series of models using linear probes.


LM RN50 This is a multimodal model that uses an autoregressive loss instead of a contrastive loss, while using


the ResNet-50 architecture as in the smallest contrastive model. To do so, the output from the CNN is projected into four tokens, which are then fed as a prefix to a language model autoregressively predicting the text tokens. Apart from the training objective, the model was trained on the same dataset for the same number of epochs as other CLIP models.


CLIP-RN Five ResNet-based contrastive CLIP models are included. As discussed in the paper, the first two models follow ResNet-50 and ResNet-101, and we use EfficientNet-style (Tan & Le, 2019) scaling for the next three models which simultaneously scale the model width, the number of layers, and the input resolution to obtain models with roughly 4x, 16x, and 64x computation.


CLIP-ViT We include four CLIP models that use the Vision Transformer (Dosovitskiy et al., 2020) architecture as the image encoder. We include three models trained on 224-by-224 pixel images: ViT-B/32, ViT-B/16, ViT-L/14, and the ViT-L/14 model fine-tuned on 336-by-336 pixel input images.


EfficietNet We use the nine models (B0-B8) from the original EfficientNet paper (Tan & Le, 2019), as well as the noisy-student variants (B0-B7, L2-475, and L2-800) (Tan & Le, 2019). The largest models (L2-475 and L2-800) take the input resolutions of $475 \times 475$ and $800 \times 800$ pixels, respectively.


Instagram-pretrained ResNeXt We use the four models (32x8d, 32x16d, 32x32d, 32x48d) released by (Mahajan et al., 2018), as well as their two FixRes variants which use higher input resolutions (Touvron et al., 2019).


Big Transfer (BiT) We use BiT-S and BiT-M models (Kolesnikov et al., 2019), trained on the ImageNet-1k and ImageNet-21k datasets. The model weights for BiT-L is not publicly available.


Vision Transformer (ViT) We also include four ViT (Dosovitskiy et al., 2020) checkpoints pretrained on the ImageNet-21k dataset, namely ViT-B/32, ViT-B/16, ViT-L/16, and ViT-H/14. We note that their best-performing models, trained on the JFT-300M dataset, are not available publicly.


SimCLRv2 The SimCLRv2 (Chen et al., 2020c) project released pre-trained and fine-tuned models in various settings. We use the seven pretrain-only checkpoints with selective kernels.


BYOL We use the recently released model weights of BYOL (Grill et al., 2020), specifically their $50 \times 1$ and $200 \times 2$


---


![图片 page38-2](images/page038_img01.png)


_Figure 19. Two example images from the Rendered SST2 dataset_


checkpoints.


Momentum Contrast (MoCo) We include the MoCo-v1 (He et al., 2020) and the MoCo-v2 (Chen et al., 2020d) checkpoints.


VirTex We use the pretrained model of VirTex (Desai & Johnson, 2020). We note that VirTex has a similar model design to CLIP-AR but is trained on a $1000\mathrm{x}$ smaller dataset of high-quality captions from MSCOCO.


ResNet We add the original ResNet checkpoints released by (He et al., 2016b), namely ResNet-50, ResNet-101, and ResNet152.


# A.3. Evaluation


We use image features taken from the penultimate layer of each model, ignoring any classification layer provided. For CLIP-ViT models, we used the features before the linear projection to the embedding space, which corresponds to $ I_{-}f $ in Figure 3. We train a logistic regression classifier using scikit-learn's L-BFGS implementation, with maximum 1,000 iterations, and report the corresponding metric for each dataset. We determine the L2 regularization strength $ \lambda $ using a hyperparameter sweep on the validation sets over the range between $ 10^{-6} $ and $ 10^{6} $, with 96 logarithmically spaced steps. To save compute required for the sweeps, we perform a parametric binary search that starts with $ \lambda = [10^{-6}, 10^{-4}, 10^{-2}, 1, 10^{2}, 10^{4}, 10^{6}] $ and iteratively halves the interval around the peak until it reaches a resolution of 8 steps per decade. The hyperparameter sweeps are performed on a validation split of each dataset. For the datasets that contain a validation split in addition to


a test split, we use the provided validation set to perform the hyperparameter search, and for the datasets that do not provide a validation split or have not published labels for the test data, we split the training dataset to perform the hyperparameter search. For the final result, we combine the validation split back with the training split and report the performance on the unused split.


# A.4. Results


The individual linear probe scores are provided in Table 10 and plotted in Figure 20. The best-performing CLIP model, using ViT-L/14 architecture and 336-by-336 pixel images, achieved the state of the art in 21 of the 27 datasets, i.e. included in the Clopper-Pearson $99.5\%$ confidence interval around each dataset's top score. For many datasets, CLIP performs significantly better than other models, demonstrating the advantage of natural language supervision over traditional pre-training approaches based on image classification. See Section 3.2 for more discussions on the linear probe results.


---


<table><tr><td>Dataset</td><td>Classes</td><td>Train size</td><td>Test size</td><td>Evaluation metric</td></tr><tr><td>Food-101</td><td>102</td><td>75,750</td><td>25,250</td><td>accuracy</td></tr><tr><td>CIFAR-10</td><td>10</td><td>50,000</td><td>10,000</td><td>accuracy</td></tr><tr><td>CIFAR-100</td><td>100</td><td>50,000</td><td>10,000</td><td>accuracy</td></tr><tr><td>Birdsnap</td><td>500</td><td>42,283</td><td>2,149</td><td>accuracy</td></tr><tr><td>SUN397</td><td>397</td><td>19,850</td><td>19,850</td><td>accuracy</td></tr><tr><td>Stanford Cars</td><td>196</td><td>8,144</td><td>8,041</td><td>accuracy</td></tr><tr><td>FGVC Aircraft</td><td>100</td><td>6,667</td><td>3,333</td><td>mean per class</td></tr><tr><td>Pascal VOC 2007 Classification</td><td>20</td><td>5,011</td><td>4,952</td><td>11-point mAP</td></tr><tr><td>Describable Textures</td><td>47</td><td>3,760</td><td>1,880</td><td>accuracy</td></tr><tr><td>Oxford-IIIT Pets</td><td>37</td><td>3,680</td><td>3,669</td><td>mean per class</td></tr><tr><td>Caltech-101</td><td>102</td><td>3,060</td><td>6,085</td><td>mean-per-class</td></tr><tr><td>Oxford Flowers 102</td><td>102</td><td>2,040</td><td>6,149</td><td>mean per class</td></tr><tr><td>MNIST</td><td>10</td><td>60,000</td><td>10,000</td><td>accuracy</td></tr><tr><td>Facial Emotion Recognition 2013</td><td>8</td><td>32,140</td><td>3,574</td><td>accuracy</td></tr><tr><td>STL-10</td><td>10</td><td>1000</td><td>8000</td><td>accuracy</td></tr><tr><td>EuroSAT</td><td>10</td><td>10,000</td><td>5,000</td><td>accuracy</td></tr><tr><td>RESISC45</td><td>45</td><td>3,150</td><td>25,200</td><td>accuracy</td></tr><tr><td>GTSRB</td><td>43</td><td>26,640</td><td>12,630</td><td>accuracy</td></tr><tr><td>KITTI</td><td>4</td><td>6,770</td><td>711</td><td>accuracy</td></tr><tr><td>Country211</td><td>211</td><td>43,200</td><td>21,100</td><td>accuracy</td></tr><tr><td>PatchCamelyon</td><td>2</td><td>294,912</td><td>32,768</td><td>accuracy</td></tr><tr><td>UCF101</td><td>101</td><td>9,537</td><td>1,794</td><td>accuracy</td></tr><tr><td>Kinetics700</td><td>700</td><td>494,801</td><td>31,669</td><td>mean(top1, top5)</td></tr><tr><td>CLEVR Counts</td><td>8</td><td>2,000</td><td>500</td><td>accuracy</td></tr><tr><td>Hateful Memes</td><td>2</td><td>8,500</td><td>500</td><td>ROC AUC</td></tr><tr><td>Rendered SST2</td><td>2</td><td>7,792</td><td>1,821</td><td>accuracy</td></tr><tr><td>ImageNet</td><td>1000</td><td>1,281,167</td><td>50,000</td><td>accuracy</td></tr></table>


_Table 9. Datasets examined for linear probes. We note that, for the Birdsnap and Kinetics700 datasets, we used the resources that are available online at the time of this writing._


---


<table><tr><td rowspan="56">ResNet</td><td rowspan="56">VMx</td><td rowspan="56">MoCo</td><td rowspan="56">SimCLRv2</td><td rowspan="56">ViT</td><td rowspan="56">BiT-M</td><td rowspan="56">BiT-S</td><td rowspan="8">Instagram</td><td rowspan="8">EfficientNet Noisy Student</td><td rowspan="8">EfficientNet</td><td rowspan="8">CLIP-VIT</td><td rowspan="8">CLIP-RN</td><td rowspan="8">CLIP-RN</td></tr><tr></tr><tr></tr><tr></tr><tr></tr><tr></tr><tr></tr><tr></tr><tr><td>3x80</td><td>B17</td><td>75.92</td><td>74.76</td><td>73.67</td><td>72.63</td></tr><tr><td>3x24</td><td>B17</td><td>84.95</td><td>83.99</td><td>82.99</td><td></td></tr><tr><td>3x24</td><td>B17</td><td>84.95</td><td>83.99</td><td>82.99</td><td></td></tr><tr><td>FtxRes-v2</td><td>B17</td><td>84.95</td><td>83.99</td><td>82.99</td><td></td></tr><tr><td>R50x1</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R50x2</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R101x1</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R101x2</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R15x2</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76.53</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27</td><td>78.61</td><td>76</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.00</td><td>72.9</td><td>72.9</td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53</td><td>72.9</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 100</td><td>72.9</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 0.00</td><td>72.9</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.50</td><td>72.9</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53 76.53</td><td>72.9</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>74.00</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>74.00</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>74,00</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>74,00</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>10</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>10</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>1</td><td></td></tr><tr><td></td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>100;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>101</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>101</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>101;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>101;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>101;1</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>11</td><td></td></tr><tr><td>R15x3</td><td>84.95</td><td>79.27 78.61</td><td>75.92 76.00</td><td>11</td><td></td></tr></table>


_Table 10. Linear probe performance of various pre-trained models over 27 datasets. Scores within the $99.5\%$ Clopper-Pearson confidence interval of each dataset's top score are shown in bold._


* We updated the STL10 scores from the previous version of this paper after fixing a CUDA-related bug.


---


![图片 page41-2](images/page041_img01.png)


![图片 page41-3](images/page041_img02.png)


![图片 page41-4](images/page041_img03.png)


![图片 page41-5](images/page041_img04.png)


![图片 page41-6](images/page041_img05.png)


![图片 page41-7](images/page041_img06.png)


![图片 page41-8](images/page041_img07.png)


![图片 page41-9](images/page041_img08.png)


![图片 page41-10](images/page041_img09.png)


![图片 page41-11](images/page041_img10.png)


![图片 page41-12](images/page041_img11.png)


![图片 page41-13](images/page041_img12.png)


![图片 page41-14](images/page041_img13.png)


![图片 page41-15](images/page041_img14.png)


![图片 page41-16](images/page041_img15.png)


![图片 page41-17](images/page041_img16.png)


![图片 page41-18](images/page041_img17.png)


![图片 page41-19](images/page041_img18.png)


![图片 page41-20](images/page041_img19.png)


![图片 page41-21](images/page041_img20.png)


![图片 page41-22](images/page041_img21.png)


![图片 page41-23](images/page041_img22.png)


![图片 page41-24](images/page041_img23.png)


![图片 page41-25](images/page041_img24.png)


![图片 page41-26](images/page041_img25.png)


![图片 page41-27](images/page041_img26.png)


![图片 page41-28](images/page041_img27.png)


_CLIP-ViT_


_CLIP-ResNet_


_EfficientNet-NoisyStudent_


_EfficientNet_


_- Instagram-pretrained_


_SimCLRv2_


_BYOL_


_MoCo_


_ViT (ImageNet-21k)_


_BiT-M_


_BiT-S_


_+ ResNet_


_Figure 20. Linear probe performance plotted for each of the 27 datasets, using the data from Table 10._


---


![图片 page42-2](images/page042_img01.png)


_Figure 21. Visualization of predictions from 36 CLIP zero-shot classifiers. All examples are random with the exception of reselecting Hateful Memes to avoid offensive content. The predicted probability of the top 5 classes is shown along with the text used to represent the class. When more than one template is used, the first template is shown. The ground truth label is colored green while an incorrect prediction is colored orange._


---


<table><tr><td colspan="2"></td><td>Food101</td><td>CIFAR10</td><td>CIFAR100</td><td>Birdsnap</td><td>SUN397</td><td>Stanford Cars</td><td>FGVC Aircraft</td><td>VOC2007</td><td>DTD</td><td>Oxford Pets</td><td>Caltech101</td><td>Flowers102</td><td>MNIST</td><td>FER2013</td><td>EuroSAT</td><td>RESISC45</td><td>GTSRB</td><td>KITTI</td><td>Country211</td><td>PCam</td><td>UCF101</td><td>Kinetics700</td><td>CLEVR</td><td>HatefulMemes</td><td>Rendered SST2</td><td>ImageNet</td><td></td></tr><tr><td rowspan="5">CLIP-ResNet</td><td>RN50</td><td>81.1</td><td>75.6</td><td>41.6</td><td>32.6</td><td>59.6</td><td>55.8</td><td>19.3</td><td>82.1</td><td>41.7</td><td>85.4</td><td>82.1</td><td>65.9</td><td>66.6</td><td>42.2</td><td>94.3</td><td>41.1</td><td>54.2</td><td>35.2</td><td>42.2</td><td>16.1</td><td>57.6</td><td>63.6</td><td>43.5</td><td>20.3</td><td>59.7</td><td>56.9</td><td>59.6</td></tr><tr><td>RN101</td><td>83.9</td><td>81.0</td><td>49.0</td><td>37.2</td><td>59.9</td><td>62.3</td><td>19.5</td><td>82.4</td><td>43.9</td><td>86.2</td><td>85.1</td><td>65.7</td><td>59.3</td><td>45.6</td><td>96.7</td><td>33.1</td><td>58.5</td><td>38.3</td><td>33.3</td><td>16.9</td><td>55.2</td><td>62.2</td><td>46.7</td><td>28.1</td><td>61.1</td><td>64.2</td><td>62.2</td></tr><tr><td>RN50x4</td><td>86.8</td><td>79.2</td><td>48.9</td><td>41.6</td><td>62.7</td><td>67.9</td><td>24.6</td><td>83.0</td><td>49.3</td><td>88.1</td><td>86.0</td><td>68.0</td><td>75.2</td><td>51.1</td><td>96.4</td><td>35.0</td><td>59.2</td><td>35.7</td><td>26.0</td><td>20.2</td><td>57.5</td><td>65.5</td><td>49.0</td><td>17.0</td><td>58.3</td><td>66.6</td><td>65.8</td></tr><tr><td>RN50x16</td><td>90.5</td><td>82.2</td><td>54.2</td><td>45.9</td><td>65.0</td><td>72.3</td><td>30.3</td><td>82.9</td><td>52.8</td><td>89.7</td><td>87.6</td><td>71.9</td><td>80.0</td><td>56.0</td><td>97.8</td><td>40.3</td><td>64.4</td><td>39.6</td><td>33.9</td><td>24.0</td><td>62.5</td><td>68.7</td><td>53.4</td><td>17.6</td><td>58.9</td><td>67.6</td><td>70.5</td></tr><tr><td>RN50x64</td><td>91.8</td><td>86.8</td><td>61.3</td><td>48.9</td><td>66.9</td><td>76.0</td><td>35.6</td><td>83.8</td><td>53.4</td><td>93.4</td><td>90.6</td><td>77.3</td><td>90.8</td><td>61.0</td><td>98.3</td><td>59.4</td><td>69.7</td><td>47.9</td><td>33.2</td><td>29.6</td><td>65.0</td><td>74.1</td><td>56.8</td><td>27.5</td><td>62.1</td><td>70.7</td><td>73.6</td></tr><tr><td rowspan="4">CLIP-VIT</td><td>B/32</td><td>84.4</td><td>91.3</td><td>65.1</td><td>37.8</td><td>63.2</td><td>59.4</td><td>21.2</td><td>83.1</td><td>44.5</td><td>87.0</td><td>87.9</td><td>66.7</td><td>51.9</td><td>47.3</td><td>97.2</td><td>49.4</td><td>60.3</td><td>32.2</td><td>39.4</td><td>17.8</td><td>58.4</td><td>64.5</td><td>47.8</td><td>24.8</td><td>57.6</td><td>59.6</td><td>63.2</td></tr><tr><td>B/16</td><td>89.2</td><td>91.6</td><td>68.7</td><td>39.1</td><td>65.2</td><td>65.6</td><td>27.1</td><td>83.9</td><td>46.0</td><td>88.9</td><td>89.3</td><td>70.4</td><td>56.0</td><td>52.7</td><td>98.2</td><td>54.1</td><td>65.5</td><td>43.3</td><td>44.0</td><td>23.3</td><td>48.1</td><td>69.8</td><td>52.4</td><td>23.4</td><td>61.7</td><td>59.8</td><td>68.6</td></tr><tr><td>L/14</td><td>92.9</td><td>96.2</td><td>77.9</td><td>48.3</td><td>67.7</td><td>77.3</td><td>36.1</td><td>84.1</td><td>55.3</td><td>93.5</td><td>92.6</td><td>78.7</td><td>87.2</td><td>57.5</td><td>99.3</td><td>59.9</td><td>71.6</td><td>50.3</td><td>23.1</td><td>32.7</td><td>58.8</td><td>76.2</td><td>60.3</td><td>24.3</td><td>63.3</td><td>64.0</td><td>75.3</td></tr><tr><td>L/14-336px</td><td>93.8</td><td>95.7</td><td>77.5</td><td>49.5</td><td>68.4</td><td>78.8</td><td>37.2</td><td>84.3</td><td>55.7</td><td>93.5</td><td>92.8</td><td>78.3</td><td>88.3</td><td>57.7</td><td>99.4</td><td>59.6</td><td>71.7</td><td>52.3</td><td>21.9</td><td>34.9</td><td>63.0</td><td>76.9</td><td>61.3</td><td>24.8</td><td>63.3</td><td>67.9</td><td>76.2</td></tr></table>


_Table 11. Zero-shot performance of CLIP models over 27 datasets._


![图片 page43-4](images/page043_img01.png)


![图片 page43-5](images/page043_img02.png)


![图片 page43-6](images/page043_img03.png)


![图片 page43-7](images/page043_img04.png)


![图片 page43-8](images/page043_img05.png)


![图片 page43-9](images/page043_img06.png)


![图片 page43-10](images/page043_img07.png)


![图片 page43-11](images/page043_img08.png)


![图片 page43-12](images/page043_img09.png)


![图片 page43-13](images/page043_img10.png)


![图片 page43-14](images/page043_img11.png)


![图片 page43-15](images/page043_img12.png)


![图片 page43-16](images/page043_img13.png)


![图片 page43-17](images/page043_img14.png)


![图片 page43-18](images/page043_img15.png)


![图片 page43-19](images/page043_img16.png)


![图片 page43-20](images/page043_img17.png)


![图片 page43-21](images/page043_img18.png)


![图片 page43-22](images/page043_img19.png)


![图片 page43-23](images/page043_img20.png)


![图片 page43-24](images/page043_img21.png)


![图片 page43-25](images/page043_img22.png)


![图片 page43-26](images/page043_img23.png)


![图片 page43-27](images/page043_img24.png)


![图片 page43-28](images/page043_img25.png)


![图片 page43-29](images/page043_img26.png)


![图片 page43-30](images/page043_img27.png)


![图片 page43-31](images/page043_img28.png)


_CLIP-ResNet_


_ResNet_


_Figure 22. CLIP's zero-shot performance compared to linear-probe ResNet performance_


---


# B. Zero-Shot Prediction


To provide a qualitative summary / overview of CLIP's zero-shot performance we visualize a randomly selected prediction for 36 different zero-shot CLIP classifiers in Figure 21. In addition, Table 11 and Figure 22 show the individual zero-shot performance scores for each dataset.


# C. Duplicate Detector


Our early attempts at duplicate detection and analysis used nearest neighbors in the model's learned embedding space. While it is intuitive to use a model's own notion of similarity, we encountered issues. We found the model's feature space is weighted very heavily towards semantic similarity. Many false positives occurred due to distinct objects that would be described similarly (soccer balls, flowers of the same species, etc...) having almost perfect similarity. We also observed the model was quite poor at assigning certain kinds of near-duplicates high similarity scores. We noticed repeatedly that images with high-frequency textures (such as fur or stripe patterns) pre-processed by different resizing algorithms (nearest neighbor vs bi-linear) could have surprisingly low similarity. This resulted in many false negatives.


We built our own near-duplicate detector to fix this issue. We created a synthetic data augmentation pipeline that combined a variety of common image manipulations. The augmentation pipeline combines random cropping and zooming, aspect ratio distortion, downsizing and upscaling to different resolutions, minor rotations, JPEG compression, and HSV color jitter. The pipeline also randomly selects from different interpolation algorithms for all relevant steps. We then trained a model to maximize the similarity of an image and its transformed variant while minimizing similarity to all other images in a training batch. We used the same n-pair / InfoNCE loss as CLIP but with a fixed temperature of 0.07.


We selected a ResNet-50 as the model architecture. We modified the base ResNet-50 with the anti-alias improvements from (Zhang, 2019) and used weight norm (Salimans & Kingma, 2016) instead of batch norm (Ioffe & Szegedy, 2015) to avoid leaking information about duplicates via batch statistics - a problem previously noted in (Henaff, 2020). We also found the GELU activation function (Hendrycks & Gimpel, 2016) to perform better for this task. We trained the model with a total batch size of 1,712 for approximately 30 million images sampled from our pretraining dataset. At the end of training it achieves nearly $100\%$ accuracy on its proxy training task.


<table><tr><td rowspan="2">Dataset</td><td colspan="3">Linear Classifier</td><td colspan="3">Zero Shot</td></tr><tr><td>YFCC</td><td>WIT</td><td>Δ</td><td>YFCC</td><td>WIT</td><td>Δ</td></tr><tr><td>Birdsnap</td><td>47.4</td><td>35.3</td><td>+12.1</td><td>19.9</td><td>4.5</td><td>+15.4</td></tr><tr><td>Country211</td><td>23.1</td><td>17.3</td><td>+5.8</td><td>5.2</td><td>5.3</td><td>+0.1</td></tr><tr><td>Flowers102</td><td>94.4</td><td>89.8</td><td>+4.6</td><td>48.6</td><td>21.7</td><td>+26.9</td></tr><tr><td>GTSRB</td><td>66.8</td><td>72.5</td><td>-5.7</td><td>6.9</td><td>7.0</td><td>-0.1</td></tr><tr><td>UCF101</td><td>69.2</td><td>74.9</td><td>-5.7</td><td>22.9</td><td>32.0</td><td>-9.1</td></tr><tr><td>Stanford Cars</td><td>31.4</td><td>50.3</td><td>-18.9</td><td>3.8</td><td>10.9</td><td>-7.1</td></tr><tr><td>ImageNet</td><td>62.0</td><td>60.8</td><td>+1.2</td><td>31.3</td><td>27.6</td><td>+3.7</td></tr><tr><td>Dataset Average</td><td>65.5</td><td>66.6</td><td>-1.1</td><td>29.6</td><td>30.0</td><td>-0.4</td></tr><tr><td>Dataset “Wins”</td><td>10</td><td>15</td><td>-5</td><td>19</td><td>18</td><td>+1</td></tr></table>


_Table 12. CLIP performs similarly when trained on only YFCC100M. Comparing a ResNet-50 trained on only YFCC100M with a same sized subset of WIT shows similar average performance and number of wins on zero shot and linear classifier evals. However, large differences in dataset specific performance occur. We include performance on the 3 datasets where YFCC does best and worst compared to WIT according to a linear probe in order to highlight this as well as aggregate performance across all linear and zero-shot evals and the canonical ImageNet dataset._


# D. Dataset Ablation on YFCC100M


To study whether our custom dataset is critical to the performance of CLIP, we trained a model on a filtered subset of the YFCC100M dataset (details described in Section 2.2) and compared its performance to the same model trained on an equally sized subset of WIT. We train each model for 32 epochs at which point transfer performance begins to plateau due to overfitting. Results are shown in Table 12. Across our whole eval suite, YFCC and WIT perform similarly on average for both zero-shot and linear probe settings. However, performance on specific fine-grained classification datasets can vary widely - sometimes by over $10\%$. Our speculation is that these differences in performance reflect the relative density of relevant data in each pre-training dataset. For instance, pre-training on YFCC100M, which might contain many photos of birds and flowers (common subjects for photographers), results in better performance on Birdsnap and Flowers102, while pre-training on WIT results in better car and pet classifiers (which appear common in our dataset).


Overall, these results are encouraging as they suggest our approach can use any reasonably filtered collection of paired (text, image) data. This mirrors recent work which reported positive results using the same contrastive pre-training objective on the relatively different domain of medical imaging (Zhang et al., 2020). It also is similar to the findings of noisy student self-training which reported only slight improvements when using their JFT300M dataset over YFCC100M (Xie et al., 2020). We suspect the major advantage of our dataset over the already existing YFCC100M is its much larger size.


---


Finally, we caution that WIT includes this filtered subset of YFCC100M. This could result in our ablation underestimating the size of performance differences between YFCC100M and the rest of WIT. We do not think this is likely as YFCC100M is only $3.7\%$ of the overall WIT data blend and it did not noticeably change the performance of models when it was added to the existing data blend during the creation of WIT.


# E. Selected Task and Dataset Results


Due to the large variety of datasets and experiments considered in this work, the main body focuses on summarizing and analyzing overall results. In the following subsections we report details of performance for specific groups of tasks, datasets, and evaluation settings.


# E.1. Image and Text Retrieval


CLIP pre-trains for the task of image-text retrieval on our noisy web-scale dataset. Although the focus of this paper is on representation learning and task learning for the purpose of transfer to a wide variety of downstream datasets, validating that CLIP is able to achieve high transfer performance transfer on exactly what it is pre-trained for is an important sanity check / proof of concept. In Table 13 we check the zero-shot transfer performance of CLIP for both text and image retrieval on the Flickr30k and MSCOCO datasets. Zero-shot CLIP matches or outperforms all prior zero-shot results on these two datasets. Zero-shot CLIP is also competitive with the current overall SOTA for the task of text retrieval on Flickr30k. On image retrieval, CLIP's performance relative to the overall state of the art is noticeably lower. However, zero-shot CLIP is still competitive with a fine-tuned Unicoder-VL. On the larger MS-COCO dataset fine-tuning improves performance significantly and zero-shot CLIP is not competitive with the most recent work For both these datasets we pretend the prompt "a photo o f" to the description of each image which we found boosts CLIP's zero-shot R@1 performance between 1 and 2 points


# E.2. Optical Character Recognition


Although visualizations have shown that ImageNet models contain features that respond to the presence of text in an image (Zeiler & Fergus, 2014), these representations are not sufficiently fine-grained to use for the task of optical character recognition (OCR). To compensate, models are augmented with the outputs of custom OCR engines and features to boost performance on tasks where this capability is required (Singh et al., 2019; Yang et al., 2020). Early during the development of CLIP, we noticed that CLIP began to learn primitive OCR capabilities which appeared to steadily improve over the course of the project. To evaluate this qualitatively noticed behavior, we measured performance


on 5 datasets requiring the direct and indirect use of OCR. Three of these datasets MNIST (LeCun), SVHN (Netzer et al., 2011), and IIIT5K (Mishra et al., 2012) directly check the ability of a model to perform low-level character and word recognition, while Hateful Memes (Kiela et al., 2020) and SST-2 (Socher et al., 2013) check the ability of a model to use OCR to perform a semantic task. Results are reported in Table 14.


CLIP's performance is still highly variable and appears to be sensitive to some combination of the domain (rendered or natural images) and the type of text to be recognized (numbers or words). CLIP's OCR performance is strongest Hateful Memes and SST-2 - datasets where the text is digitally rendered and consists mostly of words. On IIIT5K, which is natural images of individually cropped words, zero-shot CLIP performs a bit more respectively and its performance is similar to Jaderberg et al. (2014) early work combining deep learning and structured prediction to perform open-vocabulary OCR. However, performance is noticeably lower on two datasets involving recognition of hand written and street view numbers. CLIP's $51\%$ accuracy on full number SVHN is well below any published results. Inspection suggests CLIP struggles with repeated characters as well as the low resolution and blurry images of SVHN. CLIP's zero-shot MNIST performance is also poor and is outperformed by supervised logistic regression on raw pixels, one of the simplest possible machine learning baselines.


SST-2 is a sentence level NLP dataset which we render into images. We include SST-2 in order to check whether CLIP is able to convert low level OCR capability into a higher level representation. Fitting a linear classifier on CLIP's representation of rendered sentences achieves $80.5\%$ accuracy. This is on par with the $80\%$ accuracy of a continuous bag of words baseline using GloVe word vectors pre-trained on 840 billion tokens (Pennington et al., 2014). While this is a simple NLP baseline by today's standard, and well below the $97.5\%$ of the current SOTA, it is encouraging to see that CLIP is able to turn an image of rendered text into a non-trivial sentence level representation. Fully supervised CLIP is also surprisingly strong on Hateful Mem detection, where CLIP is only 0.7 points behind the current single model SOTA and several points above the best baseline from the original paper. Similar to SST-2, these other results on Hateful Memes use the ground truth text which CLIP does not have access to. Finally, we note that zero-shot CLIP outperforms the best results using fully supervised linear probes across all other 56 models included in our evaluation suite. This suggests CLIP's OCR capability is at least somewhat unique compared to existing work on self-supervised and supervised representation learning.


---


<table><tr><td rowspan="3" colspan="2"></td><td colspan="7">Text Retrieval</td><td colspan="6">Image Retrieval</td></tr><tr><td colspan="3">Flickr30k</td><td colspan="3">MSCOCO</td><td colspan="3">Flickr30k</td><td colspan="3">MSCOCO</td><td></td></tr><tr><td>R@1</td><td>R@5</td><td>R@10</td><td>R@1</td><td>R@5</td><td>R@10</td><td>R@1</td><td>R@5</td><td>R@10</td><td>R@1</td><td>R@5</td><td>R@10</td><td></td></tr><tr><td rowspan="5">Finetune</td><td>Unicoder-VLa</td><td>86.2</td><td>96.3</td><td>99.0</td><td>62.3</td><td>87.1</td><td>92.8</td><td>71.5</td><td>90.9</td><td>94.9</td><td>46.7</td><td>76.0</td><td>85.3</td><td></td></tr><tr><td>Uniterb</td><td>87.3</td><td>98.0</td><td>99.2</td><td>65.7</td><td>88.6</td><td>93.8</td><td>75.6</td><td>94.1</td><td>96.8</td><td>52.9</td><td>79.9</td><td>88.0</td><td></td></tr><tr><td>VILLAc</td><td>87.9</td><td>97.5</td><td>98.8</td><td>-</td><td>-</td><td>-</td><td>76.3</td><td>94.2</td><td>96.8</td><td>-</td><td>-</td><td>-</td><td></td></tr><tr><td>Oscard</td><td>-</td><td>-</td><td>-</td><td>73.5</td><td>92.2</td><td>96.0</td><td>-</td><td>-</td><td>-</td><td>57.5</td><td>82.8</td><td>89.8</td><td></td></tr><tr><td>ERNIE-VILe</td><td>88.7</td><td>98.0</td><td>99.2</td><td>-</td><td>-</td><td>-</td><td>76.7</td><td>93.6</td><td>96.4</td><td>-</td><td>-</td><td>-</td><td></td></tr><tr><td rowspan="5">Zero-Shot</td><td>Visual N-Gramsf</td><td>15.4</td><td>35.7</td><td>45.1</td><td>8.7</td><td>23.1</td><td>33.3</td><td>8.8</td><td>21.2</td><td>29.9</td><td>5.0</td><td>14.5</td><td>21.9</td><td></td></tr><tr><td>ImageBERTg</td><td>-</td><td>-</td><td>-</td><td>44.0</td><td>71.2</td><td>80.4</td><td>-</td><td>-</td><td>-</td><td>32.3</td><td>59.0</td><td>70.2</td><td></td></tr><tr><td>Unicoder-VLa</td><td>64.3</td><td>86.8</td><td>92.3</td><td>-</td><td>-</td><td>-</td><td>48.4</td><td>76.0</td><td>85.2</td><td>-</td><td>-</td><td>-</td><td></td></tr><tr><td>Uniterb</td><td>83.6</td><td>95.7</td><td>97.7</td><td>-</td><td>-</td><td>-</td><td>68.7</td><td>89.2</td><td>93.9</td><td>-</td><td>-</td><td>-</td><td></td></tr><tr><td>CLIP</td><td>88.0</td><td>98.7</td><td>99.4</td><td>58.4</td><td>81.5</td><td>88.1</td><td>68.7</td><td>90.6</td><td>95.2</td><td>37.8</td><td>62.4</td><td>72.2</td><td></td></tr></table>


_Table 13. CLIP improves zero-shot retrieval and is competitive with the best fine-tuned result on Flickr30k text retrieval. Bold indicates best overall performance while an underline indicates best in category performance (zero-shot or fine-tuned). For all other models, best results from the paper are reported regardless of model size / variant. MSCOCO performance is reported on the 5k test set. $ {}^{a} $ (Li et al.,2020a) $ {}^{b} $ (Chen et al.,2019) $ {}^{c} $ (Gan et al.,2020) $ {}^{d} $ (Li et al.,2020b) $ {}^{e} $ (Yu et al.,2020) $ {}^{f} $ (Li et al.,2017) $ {}^{g} $ (Qi et al.,2020)_


<table><tr><td></td><td></td><td>MNIST</td><td>SVHN</td><td>IIIT5K 1k</td><td>Hateful Memes</td><td>SST-2</td></tr><tr><td rowspan="3">Finetune</td><td>SOTA</td><td>99.8a</td><td>96.4b</td><td>98.9c</td><td>78.0d</td><td>97.5e</td></tr><tr><td>JOINTf</td><td>-</td><td>-</td><td>89.6</td><td>-</td><td>-</td></tr><tr><td>CBoWg</td><td>-</td><td>-</td><td>-</td><td>-</td><td>80.0</td></tr><tr><td rowspan="3">Linear</td><td>Raw Pixels</td><td>92.5</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>ES Best</td><td>98.9h</td><td>-</td><td>-</td><td>58.6h</td><td>59.0i</td></tr><tr><td>CLIP</td><td>99.2</td><td>-</td><td>-</td><td>77.3</td><td>80.5</td></tr><tr><td>ZS</td><td>CLIP</td><td>88.4</td><td>51.0</td><td>90.0</td><td>63.3</td><td>67.9</td></tr></table>


_Table 14. OCR performance on 5 datasets. All metrics are accuracy on the test set except for Hateful Memes which reports ROC AUC on the dev set. Single model SOTA reported to best of knowledge. ES Best reports the best performance across the 56 non-CLIP models in our evaluation suite. $ {}^{a} $ (Assiri,2020) $ {}^{b} $ (Jaderberg et al., 2015) $ {}^{c} $ (Wang et al.,2020) $ {}^{d} $ (Lippe et al.,2020) $ {}^{f} $ (Jaderberg et al., 2014) $ {}^{g} $ (Wang et al.,2018) $ {}^{h} $ (Xie et al.,2020) $ {}^{i} $ (Mahajan et al., 2018)_


# E.3. Action Recognition in Videos


For the purpose of learning, a potentially important aspect of natural language is its ability to express, and therefore supervise, an extremely wide set of concepts. A CLIP model, since it is trained to pair semi-arbitrary text with images, is likely to receive supervision for a wide range of visual concepts involving both common and proper nouns, verbs, and adjectives. ImageNet-1K, by contrast, only labels common nouns. Does the lack of broader supervision in ImageNet result in weaker transfer of ImageNet models to tasks involving the recognition of visual concepts that are not nouns?


To investigate this, we measure and compare the performance of CLIP and ImageNet models on several video


<table><tr><td></td><td></td><td rowspan="2">UCF101 Top-1</td><td rowspan="2">K700 AVG</td><td colspan="2">RareAct</td></tr><tr><td></td><td></td><td>mWAP</td><td>mWSAP</td></tr><tr><td rowspan="4">Finetune</td><td>R(2+1)D-BERTa</td><td>98.7</td><td>-</td><td>-</td><td>-</td></tr><tr><td>NS ENet-L2b</td><td>-</td><td>84.8</td><td>-</td><td>-</td></tr><tr><td>HT100M S3Dd</td><td>91.3</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Baseline I3De</td><td>-</td><td>70.2</td><td>-</td><td>-</td></tr><tr><td rowspan="3">Linear</td><td>MMV FACf</td><td>91.8</td><td>-</td><td>-</td><td>-</td></tr><tr><td>NS ENet-L2c</td><td>89.4c</td><td>68.2c</td><td>-</td><td>-</td></tr><tr><td>CLIP</td><td>92.0</td><td>73.0</td><td>-</td><td>-</td></tr><tr><td rowspan="2">ZS</td><td>HT100M S3Dd</td><td>-</td><td>-</td><td>30.5</td><td>34.8</td></tr><tr><td>CLIP</td><td>80.3</td><td>69.6</td><td>40.7</td><td>44.8</td></tr></table>


_Table 15. Action recognition performance on 3 video datasets. Single model SOTA reported to best of knowledge. Note that linear CLIP and linear NS ENet-L2 are trained and evaluated on a single frame subsampled version of each dataset and not directly comparable to prior work. On Kinetics-700, we report the ActivityNet competition metric which is the average of top-1 and top-5 performance. $ {}^{a} $ (Kalfaoglu et al.,2020) $ {}^{b} $ (Lu et al.,2020) $ {}^{c} $ (Xie et al., 2020) $ {}^{d} $ (Miech et al.,2020b) $ {}^{e} $ (Carreira et al.,2019) $ {}^{f} $ (Alayrac et al.,2020)_


action classification datasets which measure the ability of a model to recognize verbs. In Table 15 we report results on UCF-101 (Soomro et al., 2012) and Kinetics-700 (Carreira et al., 2019), two common datasets for the task. Unfortunately, our CPU based linear classifier takes a prohibitively long time to evaluate on a video dataset due to the very large number of training frames. To deal with this, we aggressively sub-sample each video to only a single center frame, effectively turning it into an image classification dataset. As a result, our reported performance in a linear evaluation setting likely under estimates performance by a moderate amount.


---


<table><tr><td rowspan="2"></td><td rowspan="2">IN Top-1</td><td rowspan="2">IN-V2 Top-1</td><td rowspan="2">IN-A Top-1</td><td rowspan="2">IN-R Top-1</td><td rowspan="2">ObjectNet Top-1</td><td rowspan="2">IN-Sketch Top-1</td><td colspan="2">IN-Vid</td><td colspan="2">YTBB</td></tr><tr><td>PM0</td><td>PM10</td><td>PM0</td><td>PM10</td></tr><tr><td>NS EfficientNet-L2a</td><td>88.3</td><td>80.2</td><td>84.9</td><td>74.7</td><td>68.5</td><td>47.6</td><td>88.0</td><td>82.1</td><td>67.7</td><td>63.5</td></tr><tr><td>FixResNeXt101-32x48d V2b</td><td>86.4</td><td>78.0</td><td>68.4</td><td>80.0</td><td>57.8</td><td>59.1</td><td>85.8</td><td>72.2</td><td>68.9</td><td>57.7</td></tr><tr><td>Linear Probe CLIP</td><td>85.4</td><td>75.9</td><td>75.3</td><td>84.2</td><td>66.2</td><td>57.4</td><td>89.1</td><td>77.2</td><td>68.7</td><td>63.1</td></tr><tr><td>Zero-Shot CLIP</td><td>76.2</td><td>70.1</td><td>77.2</td><td>88.9</td><td>72.3</td><td>60.2</td><td>95.3</td><td>89.2</td><td>95.2</td><td>88.5</td></tr></table>


_Table 16. Detailed ImageNet robustness performance. IN is used to abbreviate for ImageNet. $ {}^{a} $ (Xie et al.,2020) $ {}^{b} $ (Touvron et al.,2019)_


Despite this handicap, CLIP features transfer surprisingly well to this task. CLIP matches the best prior result on UCF-101 in a linear probe evaluation setting and also outperforms all other models in our evaluation suite. On Kinetics-700, CLIP also outperforms the fine-tuned I3D baseline from the original paper. Since it does not require a training stage, we report CLIP's zero-shot performance when averaging predictions across all frames. CLIP also performs well in this setting and on Kinetics-700 its performance is within $1\%$ of the fully supervised I3D baseline which is trained on 545000 labeled videos. Encouraged by these results, we also measure CLIP's performance on the recently introduced RareAct dataset (Miech et al., 2020a) which was designed to measure zero-shot recognition of unusual actions like "hammering a phone" and "drilling an egg". CLIP improves over the prior state of the art, a S3D model trained on automatically extracted captions from 100 million instructional videos, by 10 points.


While CLIP has encouragingly strong performance on the task of action recognition, we note that there are many differences between the models being compared beyond just their form of supervision such as model architecture, training data distribution, dataset size, and compute used. Further work is needed to more precisely determine what specific design decisions contribute to achieving high performance on this task.


<table><tr><td></td><td>1km</td><td>25km</td><td>200km</td><td>750km</td><td>2500km</td></tr><tr><td>ISNa</td><td>16.9</td><td>43.0</td><td>51.9</td><td>66.7</td><td>80.2</td></tr><tr><td>CPlaNetb</td><td>16.5</td><td>37.1</td><td>46.4</td><td>62.0</td><td>78.5</td></tr><tr><td>CLIP</td><td>13.9</td><td>32.9</td><td>43.0</td><td>62.0</td><td>79.3</td></tr><tr><td>Deep-Ret+c</td><td>14.4</td><td>33.3</td><td>47.7</td><td>61.6</td><td>73.4</td></tr><tr><td>PlaNetd</td><td>8.4</td><td>24.5</td><td>37.6</td><td>53.6</td><td>71.3</td></tr></table>


_Table 17. Geolocation performance on the IM2GPS test set. Metric is percent of images localized within a given radius. Models are ordered by average performance. $ {}^{a} $ (Muller-Budack et al.,2018) $ {}^{b} $ (Hongsuck Seo et al.,2018) $ {}^{c} $ (Vo et al.,2017) $ {}^{c} $ (Weyand et al., 2016)_


# E.4. Geolocalization


Another behavior we noticed during the development of CLIP was its ability to recognize many places and locations. To quantify this we created the Country211 dataset as described in Appendix A and report results on it throughout the paper. However it is a new benchmark so to compare with prior work on geolocalization we also report results on the IM2GPS test set from Hays & Efros (2008) in Table 17. Since IM2GPS is a regression benchmark, we guess the GPS coordinates of the nearest image in a set of reference images using CLIP's embedding space. This is not a zero-shot result since it uses nearest-neighbor regression. Despite querying only 1 million images, which is much less than prior work, CLIP performs similarly to several task specific models. It is not, however, competitive with the current state of the art.


# E.5. Robustness to Distribution Shift


Section 3.3 provides a high level summary and analysis of ImageNet-related robustness results. We briefly provide some additional numerical details in this appendix. Performance results per dataset are provided in Table 16 and compared with the current state of the art results reported in Taori et al. (2020)'s evaluation suite. Zero-shot CLIP improves the state of the art on 5 of the 7 datasets, ImageNet-R, ObjectNet, ImageNet-Sketch, ImageNet-Vid, and Youtube-BB. CLIP's improvements are largest on ImageNet-Vid and Youtube-BB due to its flexible zero-shot capability and on ImageNet-R, which likely reflects CLIP's pre-training distribution including significant amounts of creative content. A similar behavior has been documented for the Instagram pre-trained ResNeXt models as discussed in Taori et al. (2020).


---


# F. Model Hyperparameters


<table><tr><td>Hyperparameter</td><td>Value</td></tr><tr><td>Batch size</td><td>32768</td></tr><tr><td>Vocabulary size</td><td>49408</td></tr><tr><td>Training epochs</td><td>32</td></tr><tr><td>Maximum temperature</td><td>100.0</td></tr><tr><td>Weight decay</td><td>0.2</td></tr><tr><td>Warm-up iterations</td><td>2000</td></tr><tr><td>Adam β1</td><td>0.9</td></tr><tr><td>Adam β2</td><td>0.999 (ResNet), 0.98 (ViT)</td></tr><tr><td>Adam ε</td><td>10-8 (ResNet), 10-6 (ViT)</td></tr></table>


_Table 18. Common CLIP hyperparameters_


<table><tr><td>Model</td><td>Learning rate</td><td>Embedding dimension</td><td>Input resolution</td><td>ResNet blocks</td><td>width</td><td>Text Transformer layers</td><td>width</td><td>heads</td></tr><tr><td>RN50</td><td>$ 5 \times 10^{-4} $</td><td>1024</td><td>224</td><td>(3, 4, 6, 3)</td><td>2048</td><td>12</td><td>512</td><td>8</td></tr><tr><td>RN101</td><td>$ 5 \times 10^{-4} $</td><td>512</td><td>224</td><td>(3, 4, 23, 3)</td><td>2048</td><td>12</td><td>512</td><td>8</td></tr><tr><td>RN50x4</td><td>$ 5 \times 10^{-4} $</td><td>640</td><td>288</td><td>(4, 6, 10, 6)</td><td>2560</td><td>12</td><td>640</td><td>10</td></tr><tr><td>RN50x16</td><td>$ 4 \times 10^{-4} $</td><td>768</td><td>384</td><td>(6, 8, 18, 8)</td><td>3072</td><td>12</td><td>768</td><td>12</td></tr><tr><td>RN50x64</td><td>$ 3.6 \times 10^{-4} $</td><td>1024</td><td>448</td><td>(3, 15, 36, 10)</td><td>4096</td><td>12</td><td>1024</td><td>16</td></tr></table>


_Table 19. CLIP-ResNet hyperparameters_


<table><tr><td>Model</td><td>Learning rate</td><td>Embedding dimension</td><td>Input resolution</td><td>Vision Transformer layers</td><td>width</td><td>heads</td><td>Text Transformer layers</td><td>width</td><td>heads</td></tr><tr><td>ViT-B/32</td><td>5 × 10-4</td><td>512</td><td>224</td><td>12</td><td>768</td><td>12</td><td>12</td><td>512</td><td>8</td></tr><tr><td>ViT-B/16</td><td>5 × 10-4</td><td>512</td><td>224</td><td>12</td><td>768</td><td>12</td><td>12</td><td>512</td><td>8</td></tr><tr><td>ViT-L/14</td><td>4 × 10-4</td><td>768</td><td>224</td><td>24</td><td>1024</td><td>16</td><td>12</td><td>768</td><td>12</td></tr><tr><td>ViT-L/14-336px</td><td>2 × 10-5</td><td>768</td><td>336</td><td>24</td><td>1024</td><td>16</td><td>12</td><td>768</td><td>12</td></tr></table>


_Table 20. CLIP-ViT hyperparameters_
