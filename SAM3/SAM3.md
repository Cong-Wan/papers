
# SAM 3: Segment Anything with Concepts


Nicolas Carion*, Laura Gustafson*, Yuan-Ting Hu*, Shoubhik Debnath*, Ronghang Hu*, Didac Suris*, Chaitanya Ryali*, Kalyan Vasudev Alwala*, Haitham Khedr*, Andrew Huang, Jie Lei, Tengyu Ma, Baishan Guo, Arpit Kalla, Markus Marks, Joseph Greer, Meng Wang, Peize Sun, Roman Rädle, Triantafyllos Afouras, Effrosyni Mavroudi, Katherine Xu°, Tsung-Han Wu°, Yu Zhou°, Liliane Momeni°, Rishi Hazra°, Shuangrui Ding°, Sagar Vaze°, Francois Porcher°, Feng Li°, Siyuan Li°, Aishwarya Kamath°, Ho Kei Cheng°, Piotr Dollár†, Nikhila Ravi†, Kate Saenko†, Pengchuan Zhang†, Christoph Feichtenhofer†


Meta Superintelligence Labs


*core contributor, °intern, †project lead, order is random within groups


We present Segment Anything Model (SAM) 3, a unified model that detects, segments, and tracks objects in images and videos based on concept prompts, which we define as either short noun phrases (e.g., "yellow school bus"), image exemplars, or a combination of both. Promptable Concept Segmentation (PCS) takes such prompts and returns segmentation masks and unique identities for all matching object instances. To advance PCS, we build a scalable data engine that produces a high-quality dataset with 4M unique concept labels, including hard negatives, across images and videos. Our model consists of an image-level detector and a memory-based video tracker that share a single backbone. Recognition and localization are decoupled with a presence head, which boosts detection accuracy. SAM 3 doubles the accuracy of existing systems in both image and video PCS, and improves previous SAM capabilities on visual segmentation tasks. We open source SAM 3 along with our new Segment Anything with Concepts (SA-Co) benchmark for promptable concept segmentation.


Demo: https://segment-anything.com


Code: https://github.com/facebookresearch/sam3


Website: https://ai.meta.com/sam3


Meta


# 1 Introduction


The ability to find and segment anything in a visual scene is foundational for multimodal AI, powering applications in robotics, content creation, augmented reality, data annotation, and broader sciences. The SAM series (Kirillov et al., 2023; Ravi et al., 2024) introduced the promptable segmentation task for images and videos, focusing on Promptable Visual Segmentation (PVS) with points, boxes or masks to segment a single object per prompt. While these methods achieved a breakthrough, they did not address the general task of finding and segmenting all instances of a concept appearing anywhere in the input (e.g., all "cats" in a video).


_PROMPTABLE VISUAL SEGMENTATION_


![图片 page1-13](images/page001_img01.png)


_PROMPTABLE CONCEPT SEGMENTATION_


![图片 page1-15](images/page001_img02.png)


![图片 page1-16](images/page001_img03.png)


![图片 page1-17](images/page001_img04.png)


_Figure 1 SAM 3 improves over SAM 2 on promptable visual segmentation with clicks (left) and introduces the new promptable concept segmentation capability (right). Users can segment all instances of a visual concept specified by a short noun phrase, image exemplars (positive or negative), or a combination of both._


---


![图片 page2-0](images/page002_img01.png)


_Figure 2 Examples of SAM 3 improving segmentation of open-vocabulary concepts compared to OWLv2 (Minderer et al., 2024), on the SA-Co benchmark. See §F.6.1 for additional SAM 3 outputs._


To fill this gap, we present SAM 3, a model that achieves a step change in promptable segmentation in images and videos, improving PVS relative to SAM 2 and setting a new standard for Promptable Concept Segmentation (PCS). We formalize the PCS task (§2) as taking text and/or image exemplars as input, and predicting instance and semantic masks for every single object matching the concept, while preserving object identities across video frames (see Fig. 1). To focus on recognizing atomic visual concepts, we constrain text to simple noun phrases (NPs) such as "red apple" or "striped cat". While SAM 3 is not designed for long referring expressions or queries requiring reasoning, we show that it can be straightforwardly combined with a Multimodal Large Language Model (MLLM) to handle more complex language prompts. Consistent with previous SAM versions, SAM 3 is fully interactive, allowing users to resolve ambiguities by adding refinement prompts to guide the model towards their intended output.


Our model (§3) consists of a detector and a tracker that share a vision encoder (Bolya et al., 2025). The detector is a DETR-based (Carion et al., 2020) model conditioned on text, geometry, and image exemplars. To address the challenge of open-vocabulary concept detection, we introduce a separate presence head to decouple recognition and localization, which is especially effective when training with challenging negative phrases. The tracker inherits the SAM 2 transformer encoder-decoder architecture, supporting video segmentation and interactive refinement. The decoupled design for detection and tracking avoids task conflict, as the detector needs to be identity agnostic, while the tracker's main objective is to separate identities in the video.


To unlock major performance gains, we build a human- and model-in-the-loop data engine (§4) that annotates a large and diverse training dataset. We innovate upon prior data engines in three key ways: (i) media curation: we curate more diverse media domains than past approaches that rely on homogeneous web sources, (ii) label curation: we significantly increase label diversity and difficulty by leveraging an ontology and multimodal LLMs as “AI annotators” to generate noun phrases and hard negatives, (iii) label verification: we double annotation throughput by fine-tuning MLLMs to be effective “AI verifiers” that achieve near-human accuracy.


Starting from noisy media-phrase-mask pseudo-labels, our data engine checks mask quality and exhaustivity using both human and AI verifiers, filtering out correctly labeled examples and identifying challenging error cases. Human annotators then focus on fixing these errors by manually correcting masks. This enables us to annotate high-quality training data with 4M unique phrases and 52M masks, and a synthetic dataset with 38M phrases and 1.4B masks. We additionally create the Segment Anything with Concepts (SA-Co) benchmark for PCS ($\S 5$) containing 207K unique concepts with exhaustive masks in 120K images and 1.7K videos, $>50\times$ more concepts than existing benchmarks.


Our experiments (§6) show that SAM 3 sets a new state-of-the-art in promptable segmentation, e.g., reaching a zero-shot mask AP of 48.8 on LVIS vs. the current best of 38.5, surpassing baselines on our new SA-Co benchmark by at least $2 \times$ (see examples in Fig. 2), and improving upon SAM 2 on visual prompts. Ablations (§A) verify that the choice of backbone, novel presence head, and adding hard negatives all boost results, and establish scaling laws on the PCS task for both our high-quality and synthetic datasets. We open-source the SA-Co benchmark and release the SAM 3 checkpoints and inference code. On an H200 GPU, SAM 3 runs in 30 ms for a single image with $100+$ detected objects. In video, the inference latency scales with the number of objects, sustaining near real-time performance for $\sim 5$ concurrent objects. We review related work in §7; next, we dive into the task.


---


![图片 page3-0](images/page003_img01.png)


![图片 page3-1](images/page003_img02.png)


![图片 page3-2](images/page003_img03.png)


![图片 page3-3](images/page003_img04.png)


_Figure 3 Illustration of supported initial and optional interactive refinement prompts in the PCS task._


# 2 Promptable Concept Segmentation (PCS)


We define the Promptable Concept Segmentation task as follows: given an image or short video ($\leq 30$ secs), detect, segment and track all instances of a visual concept specified by a short text phrase, image exemplars, or a combination of both. We restrict concepts to those defined by simple noun phrases (NPs) consisting of a noun and optional modifiers. Noun-phrase prompts (when provided) are global to all frames of the image/video, while image exemplars can be provided on individual frames as positive or negative bounding boxes to iteratively refine the target masks (see Fig. 3).


All prompts must be consistent in their category definition, or the model's behavior is undefined; e.g., "fish" cannot be refined with subsequent exemplar prompts of just the tail; instead the text prompt should be updated. Exemplar prompts are particularly useful when the model initially misses some instances, or when the concept is rare.


Our vocabulary includes any simple noun phrase groundable in a visual scene, which makes the task intrinsically ambiguous. There can be multiple interpretations of phrases arising from polysemy ("mouse" device vs. animal), subjective descriptors ("cozy", "large"), vague or context-dependent phrases that may not even be groundable ("brand identity"), boundary ambiguity (whether 'mirror' includes the frame) and factors such as occlusion and blur that obscure the extent of the object. While similar issues appear in large closed-vocabulary corpora (e.g., LVIS (Gupta et al., 2019)), they are alleviated by carefully curating the vocabulary and setting a clear definition of all the classes of interest. We address the ambiguity problem by collecting test annotations from three experts, adapting the evaluation protocol to allow multiple valid interpretations ($\S$E.3), designing the data pipeline/guidelines to minimize ambiguity in annotation, and an ambiguity module in the model ($\S$C.2).


# 3 Model


SAM 3 is a generalization of SAM 2, supporting the new PCS task (§2) as well as the PVS task. It takes concept prompts (simple noun phrases, image exemplars) or visual prompts (points, boxes, masks) to define the objects to be (individually) segmented spatio-temporally. Image exemplars and visual prompts can be iteratively added on individual frames to refine the target masks—false positive and false negative objects can be removed or added respectively using image exemplars and an individual mask(let) can be refined using PVS in the style of SAM 2. Our architecture is broadly based on the SAM and (M)DETR (Carion et al., 2020; Kamath et al., 2021) series. Fig. 4 shows the SAM 3 architecture, consisting of a dual encoder-decoder transformer—a detector for image-level capabilities—which is used in combination with a tracker and memory for video. The detector and tracker ingest vision-language inputs from an aligned Perception Encoder (PE) backbone (Bolya et al., 2025). We present an overview below, see §C for details.


Detector Architecture. The architecture of the detector follows the general DETR paradigm. The image and text prompt are first encoded by PE and image exemplars, if present, are encoded by an exemplar encoder. We refer to the image exemplar tokens and text tokens jointly as "prompt tokens". The fusion encoder then accepts the unconditioned embeddings from the image encoder and conditions them by cross-attending to the prompt tokens. The fusion is followed by a DETR-like decoder, where learned object queries cross-attend to the conditioned image embeddings from the fusion encoder.


---


![图片 page4-0](images/page004_img01.png)


_Figure 4 SAM 3 architecture overview. See Fig. 10 for a more detailed diagram._


Each decoder layer predicts a classification logit for each object query (in our case, a binary label of whether the object corresponds to the prompt), and a delta from the bounding box predicted by the previous level, following Zhu et al. (2020). We use box-region positional bias (Lin et al., 2023) to help focalize the attention on each object, but unlike recent DETR models, we stick to vanilla attention. During training, we adopt dual supervision from DAC-DETR (Hu et al., 2023), and the Align loss (Cai et al., 2024). The mask head is adapted from MaskFormer (Cheng et al., 2021). In addition, we also have a semantic segmentation head, which predicts a binary label for every pixel in the image, indicating whether or not it corresponds to the prompt. See §C for details.


Presence Token. It can be difficult for each of the proposal queries to both recognize (what) and localize (where) an object in the image/frame. For the recognition component, contextual cues from the entire image are important. However, forcing proposal queries to understand the global context can be counterproductive, as it conflicts with the inherently local nature of the localization objective. We decouple the recognition and localization steps by introducing a learned global presence token. This token is solely responsible for predicting whether the target concept in the form of a noun phrase (NP) is present in the image/frame, i.e. $ p(\mathrm{NP}$ is present in input). Each proposal query $ q_{i} $ only needs to solve the localization problem $ p(q_{i}\text{is a match}|\mathrm{NP}$ is present in input). The final score for each proposal query is the product of its own score and the presence score.


Image Exemplars and Interactivity. SAM 3 supports image exemplars, given as a pair—a bounding box and an associated binary label (positive or negative)—which can be used in isolation or to supplement the text prompt. The model then detects all the instances that match the prompt. For example, given a positive bounding box on a dog, the model will detect all dogs in the image. This is different from the PVS task in SAM 1 and 2, where a visual prompt yields only a single object instance. Each image exemplar is encoded separately by the exemplar encoder using an embedding for the position, an embedding for the label, and ROI-pooled visual features, then concatenated and processed by a small transformer. The resulting prompt is concatenated to the text prompt to comprise the prompt tokens. Image exemplars can be interactively provided based on errors in current detections to refine the output.


Tracker and Video Architecture. Given a video and a prompt $ P $, we use the detector and a tracker (see Fig. 4) to detect and track objects corresponding to the prompt throughout the video. On each frame, the detector finds new objects $ \mathcal{O}_t $ and the tracker propagates masklets $ \mathcal{M}_{t-1} $ (spatial-temporal masks) from frames at the previous time $ t-1 $ to their new locations $ \hat{\mathcal{M}}_t $ on the current frame at time $ t $. We use a matching function to associate propagated masklets $ \hat{\mathcal{M}}_t $ with new object masks emerging in the current frame $ \mathcal{O}_t $,


$$
\hat {\mathcal {M}} _ {t} = \mathrm {p r o p a g a t e} \left(\mathcal {M} _ {t - 1}\right), \qquad \mathcal {O} _ {t} = \mathrm {d e t e c t} \left(I _ {t}, P\right), \qquad \mathcal {M} _ {t} = \mathrm {m a t c h \_ a n d \_ u p d a t e} \left(\hat {\mathcal {M}} _ {t}, \mathcal {O} _ {t}\right).
$$


Tracking an Object with SAM 2 Style Propagation. A masklet is initialized for every object detected on the first frame. Then, on each subsequent frame, the tracker module predicts the new masklet locations $\hat{\mathcal{M}}_t$ of those already-tracked objects based on their previous locations $\mathcal{M}_{t-1}$ through a single-frame propagation step similar to the video object segmentation task in SAM 2. The tracker shares the same image/frame encoder (PE backbone) as the detector. After training the detector, we freeze PE and train the tracker as in SAM 2, including a prompt encoder, mask decoder, memory encoder, and a memory bank that encodes the


---


![图片 page5-0](images/page005_img01.png)


_Figure 5 Overview of the final SAM 3 data engine. See §E.1 for details of collected data._


object's appearance using features from the past frames and conditioning frames (frames where the object is first detected or user-prompted). The memory encoder is a transformer with self-attention across visual features on the current frame and cross-attention from the visual features to the spatial memory features in the memory bank. We describe details of our video approach in §C.3.


During inference, we only retain frames where the object is confidently present in the memory bank. The mask decoder is a two-way transformer between the encoder hidden states and the output tokens. To handle ambiguity, we predict three output masks for every tracked object on each frame along with their confidence, and select the most confident output as the predicted mask on the current frame.


Matching and Updating Based on Detections. After obtaining the tracked masks $\hat{\mathcal{M}}_t$, we match them with the current frame detections $\mathcal{O}_t$ through a simple IoU based matching function (§C.3) and add them to $\mathcal{M}_t$ on the current frame. We further spawn new masklets for all newly detected objects that are not matched. The merging might suffer from ambiguities, especially in crowded scenes. We address this with two temporal disambiguation strategies outlined next.


First, we use temporal information in the form of a masklet detection score (§C.3) to measure how consistently a masklet is matched to a detection within a temporal window (based on the number of past frames where it was matched to a detection). If a masklet's detection score falls below a threshold, we suppress it. Second, we use the detector outputs to resolve specific failure modes of the tracker due to occlusions or distractors. We periodically re-prompt the tracker with high-confidence detection masks $\mathcal{O}_t$, replacing the tracker's own predictions $\hat{\mathcal{M}}_t$. This ensures that the memory bank has recent and reliable references (other than the tracker's own predictions).


Instance Refinement with Visual Prompts. After obtaining the initial set of masks (or masklets), SAM 3 allows refining individual masks(lets) using positive and negative clicks. Specifically, given the user clicks, we apply the prompt encoder to encode them, and feed the encoded prompt into the mask decoder to predict an adjusted mask. In videos the mask is then propagated across the entire video to obtain a refined masklet.


**Training Stages.** We train SAM 3 in four stages that progressively add data and capabilities: 1) Perception Encoder (PE) pre-training, 2) detector pre-training, 3) detector fine-tuning, and 4) tracker training with a frozen backbone. See §C.4.1 for details.


# 4 Data Engine


Achieving a step change in PCS with SAM 3 requires training on a large, diverse set of concepts and visual domains, beyond existing datasets (see Fig. 12). We build an efficient data engine that iteratively generates annotated data via a feedback loop with SAM 3, human annotators, and AI annotators, actively mining media-phrase pairs on which the current version of SAM 3 fails to produce high-quality training data to further improve the model. By delegating certain tasks to AI annotators—models that match or surpass human accuracy—we more than double the throughput compared to a human-only annotation pipeline. We develop the data engine in four phases, with each phase increasing the use of AI models to steer human effort to the most challenging failure cases, alongside expanding visual domain coverage. Phases 1-3 focus only on images, with Phase 4 expanding to videos. We describe the key steps here; details and metrics are in §D.


---


![图片 page6-0](images/page006_img01.png)


_Figure 6 Example video (top) and images (bottom) from SA-Co with annotated phrases and instance masks/IDs._


Data Engine Components (Fig. 5). Media inputs (image or video) are mined from a large pool with the help of a curated ontology. An AI model proposes noun phrases (NPs) describing visual concepts, followed by another model (e.g., SAM 3) that generates candidate instance masks for each proposed NP. The proposed masks are verified by a two-step process: first, in Mask Verification (MV) annotators accept or reject masks based on their quality and relevance to the NP. Second, in Exhaustivity Verification (EV) annotators check if all instances of the NP have been masked in the input. Any media-NP pairs that did not pass the exhaustivity check are sent to a manual correction stage, where humans add, remove or edit masks (using SAM 1 in a browser based tool), or use "group" masks for small, hard to separate objects. Annotations may reject ungroundable or ambiguous phrases.


Phase 1: Human Verification. We first randomly sample images and NP proposal with a simple captioner and parser. The initial mask proposal model is SAM 2 prompted with the output of an off-the-shelf open-vocabulary detector, and initial verifiers are human. In this phase, we collected 4.3M image-NP pairs as the initial SA-Co/HQ dataset. We train SAM 3 on this data and use it as the mask proposal model for the next phase.


Phase 2: Human + AI Verification. In this next phase, we use human accept/reject labels from the MV and EV tasks collected in Phase 1 to fine-tune Llama 3.2 (Dubey et al., 2024) to create AI verifiers that automatically perform the MV and EV tasks. These models receive image-phrase-mask triplets and output multiple-choice ratings of mask quality or exhaustivity. This new auto-verification process allows our human effort to be focused on the most challenging cases. We continue to re-train SAM 3 on newly collected data and update it 6 times. As SAM 3 and AI verifiers improve, a higher proportion of labels are auto-generated, further accelerating data collection. The introduction of AI verifiers for MV and EV roughly doubles the data engine's throughput vs. human annotators. We refer to §A.4 for detailed analysis of how AI verifiers improve the data engine's throughput. We further upgrade the NP proposal step to a Llama-based pipeline that also proposes hard negative NPs adversarial to SAM 3. Phase 2 adds 122M image-NP pairs to SA-Co/HQ.


Phase 3: Scaling and Domain Expansion. In the third phase, we use AI models to mine increasingly challenging cases and broaden domain coverage in SA-Co/HQ to 15 datasets (Fig. 15). A domain is a unique distribution of text and visual data. In new domains, the MV AI verifier performs well zero-shot, but the EV AI verifier needs to be improved with modest domain-specific human supervision. We also expand concept coverage to long-tail, fine-grained concepts by extracting NPs from the image alt-text where available and by mining concepts from a 22.4M node SA-Co ontology (§D.2) based on Wikidata (17 top-level categories, 72 sub-categories). We iterate SAM 3 training 7 times and AI verifiers 3 times, and add 19.5M image-NP pairs to SA-Co/HQ.


---


Phase 4: Video Annotation. This phase extends the data engine to video. We use a mature image SAM 3 to collect targeted quality annotations that capture video-specific challenges. The data mining pipeline applies scene/motion filters, content balancing, ranking, and targeted searches. Video frames are sampled (randomly or by object density) and sent to the image annotation flow (from phase 3). Masklets (spatio-temporal masks) are produced with SAM 3 (now extended to video) and post-processed via dedduplication and removal of trivial masks. Because video annotation is more difficult, we concentrate humans on likely failures by favoring clips with many crowded objects and tracking failures. The collected video data SA-Co/VIDEO consists of 52.5K videos and 467K masklets. See §D.6 for details.


# 5 Segment Anything with Concepts (SA-Co) Dataset


Training Data. We collect three image datasets for the PCS task: (i) SA-Co/HQ, the high-quality image data collected from the data engine in phases 1-4, (ii) SA-Co/SYN, a synthetic dataset of images labeled by a mature data engine (phase 3) without human involvement, and (iii) SA-Co/EXT, 15 external datasets that have instance mask annotations, enriched with hard negatives using our ontology pipeline. Notably in the SA-Co/HQ dataset we annotate 5.2M images and 4M unique NPs, making it the largest high-quality open-vocab segmentation dataset. We also annotate a video dataset, SA-Co/VIDEO, containing 52.5K videos and 24.8K unique NPs, forming 134K video-NP pairs. The videos on average have 84.1 frames at 6 fps. See §E.1 for details including full statistics, comparison with existing datasets and the distribution of concepts.


SA-Co Benchmark. The SA-Co evaluation benchmark has 207K unique phrases, 121K images and videos, and over 3M media-phrase pairs with hard negative labels to test open-vocabulary recognition. It has 4 splits: SA-Co/Gold has seven domains and each image-NP pair is annotated by three different annotators (used to measure human performance); SA-Co/Silver has ten domains and only one human annotation per image-NP pair; SA-Co/Bronze and SA-Co/Bio are nine existing datasets either with existing mask annotations or masks generated by using boxes as prompts to SAM 2. The SA-Co/VEval benchmark has three domains and one annotator per video-NP pair. See Tab. 28 for dataset statistics and Fig. 6 for example annotations.


Metrics. We aim to measure the usefulness of the model in downstream applications. Detection metrics such as average precision (AP) do not account for calibration, which means that models can be difficult to use in practice. To remedy this, we only evaluate predictions with confidence above 0.5, effectively introducing a threshold that mimics downstream usages and enforces good calibration. The PCS task can be naturally split into two sub-tasks, localization and classification. We evaluate localization using positive micro $ F1 $ ($ \mathrm{pmF}_1 $) on positive media-phrase pairs with at least one ground-truth mask. Classification is measured with image-level Matthews Correlation Coefficient (IL_MCC) which ranges in $[-1,1]$ and evaluates binary prediction at the image level ("is the object present?") without regard for mask quality. Our main metric, classification-gated $ F1 $ ($ \mathrm{cgF}_1 $), combines these as follows: $ \mathrm{cgF}_1 = 100 * \mathrm{pmF}_1 * \mathrm{IL_MCC} $. Full definitions are in §E.3.


Handling Ambiguity. We collect 3 annotations per NP on SA-Co/Gold. We measure oracle accuracy comparing each prediction to all ground truths and selecting the best score. See §E.3.


# 6 Experiments


We evaluate SAM 3 across image and video segmentation, few-shot adaptation to detection and counting benchmarks, and segmentation with complex language queries with SAM 3 + MLLM. We also show a subset of ablations, with more in §A. References, more results and details are in §F.


Image PCS with Text. We evaluate instance segmentation, box detection, and semantic segmentation on external and our benchmarks. SAM 3 is prompted with a single NP at a time, and predicts instance masks, bounding boxes, or semantic masks. As baselines, we evaluate OWLv2, GroundingDino (gDino), and LLMDet on box detection, and prompt SAM 1 with their boxes to evaluate segmentation. We also compare to APE, DINO-X, and Gemini 2.5 Flash, a generalist LLM. Tab. 1 shows that zero-shot, SAM 3 sets a new state-of-the-art on closed-vocabulary COCO, COCO-O and on LVIS boxes, and is significantly better on LVIS masks. On open-vocabulary SA-Co/Gold SAM 3 achieves more than double the $\mathrm{cgF}_1$ score of the strongest baseline OWLv2\*, and $74\%$ of the estimated human performance. The improvements are even higher on the other SA-Co splits. Open vocabulary semantic segmentation results on ADE-847, PascalConcept-59, and Cityscapes show that SAM 3 outperforms APE, a strong specialist baseline. See §F.1 for details.


---


<table><tr><td rowspan="3">Model</td><td colspan="6">Instance Segmentation</td><td colspan="9">Box Detection</td><td colspan="3">Semantic Segmentation</td></tr><tr><td colspan="2">LVIS</td><td colspan="4">SA-Co</td><td colspan="2">LVIS</td><td colspan="2">COCO</td><td colspan="5">SA-Co</td><td>ADE-847</td><td>PC-59</td><td>Cityscapes</td></tr><tr><td>cgF1</td><td>AP</td><td>Gold cgF1</td><td>Silver cgF1</td><td>Bronze cgF1</td><td>Bio pmF1</td><td>cgF1</td><td>AP</td><td>AP</td><td>APo</td><td>Gold cgF1</td><td>Silver cgF1</td><td>Bronze cgF1</td><td>Bio pmF1</td><td>mIoU</td><td>mIoU</td><td>mIoU</td><td></td></tr><tr><td>Human</td><td>-</td><td>-</td><td>72.8</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>74.0</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>OWLv2</td><td>20.1</td><td>-</td><td>17.3</td><td>7.6</td><td>3.9</td><td>0.64</td><td>19.9</td><td>35.2</td><td>38.2</td><td>42.4</td><td>16.9</td><td>7.1</td><td>4.1</td><td>0.95</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>OWLv2*</td><td>29.3</td><td>43.4</td><td>24.6</td><td>11.5</td><td>11.7</td><td>0.04</td><td>30.2</td><td>45.5</td><td>46.1</td><td>23.9</td><td>24.5</td><td>11.0</td><td>12.0</td><td>0.08</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>gDino-T</td><td>14.7</td><td>-</td><td>3.3</td><td>2.7</td><td>7.0</td><td>0.34</td><td>15.1</td><td>20.5</td><td>45.7</td><td>35.3</td><td>3.4</td><td>2.5</td><td>7.6</td><td>0.35</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>LLMDet-L</td><td>35.1</td><td>36.3</td><td>6.5</td><td>7.1</td><td>12.5</td><td>0.15</td><td>39.3</td><td>42.0</td><td>55.6</td><td>49.8</td><td>6.8</td><td>6.7</td><td>14.0</td><td>0.17</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>APE-D*</td><td>-</td><td>53.0†</td><td>16.4</td><td>7.3</td><td>12.4</td><td>0.00</td><td>-</td><td>59.6†</td><td>58.3†</td><td>-</td><td>17.3</td><td>7.7</td><td>14.3</td><td>0.00</td><td>9.2†</td><td>58.5†</td><td>44.2†</td><td></td></tr><tr><td>DINO-X</td><td>-</td><td>38.5†</td><td>21.3δ</td><td>-</td><td>-</td><td>-</td><td>-</td><td>52.4†</td><td>56.0†</td><td>-</td><td>22.5δ</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Gemini 2.5</td><td>13.4</td><td>-</td><td>13.0</td><td>8.3</td><td>7.3</td><td>10.7</td><td>16.1</td><td>-</td><td>-</td><td>-</td><td>14.4</td><td>9.4</td><td>8.2</td><td>12.4</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>SAM 3</td><td>37.2</td><td>48.5</td><td>54.1</td><td>49.6</td><td>42.6</td><td>55.4</td><td>40.6</td><td>53.6</td><td>56.4</td><td>55.7</td><td>55.7</td><td>50.0</td><td>47.1</td><td>56.3</td><td>13.8</td><td>60.8</td><td>65.2</td><td></td></tr></table>


_Table 1 Evaluation on image concept segmentation with text. $\mathrm{AP_o}$ corresponds to COCO-O accuracy, $\star$: partially trained on LVIS, †: from original papers, $\delta$: from DINO-X API. Gray numbers indicate usage of respective closed set training data (LVIS/COCO). See §F.1 for more baselines and results and §E.4 for details of human performance._


<table><tr><td rowspan="2">Model</td><td colspan="2">ODinW13</td><td colspan="2">RF-100VL</td></tr><tr><td>AP0</td><td>AP10</td><td>AP0</td><td>AP10</td></tr><tr><td>Gemini2.5-Pro</td><td>33.7</td><td>-</td><td>11.6</td><td>9.8</td></tr><tr><td>gDino-T</td><td>49.7</td><td>-</td><td>15.7</td><td>33.7</td></tr><tr><td>gDino1.5-Pro</td><td>58.7</td><td>67.9</td><td>-</td><td>-</td></tr><tr><td>SAM 3</td><td>61.0</td><td>71.8</td><td>15.2</td><td>36.5</td></tr></table>


_Table 2 Zero-shot and 10-shot transfer on in-the-wild datasets._


<table><tr><td rowspan="2">Model</td><td colspan="4">COCO</td><td colspan="4">LVIS</td><td colspan="4">ODinW13</td></tr><tr><td>AP T</td><td>AP+ T</td><td>AP+ I</td><td>AP+ T+I</td><td>AP T</td><td>AP+ T</td><td>AP+ I</td><td>AP+ T+I</td><td>AP T</td><td>AP+ T</td><td>AP+ I</td><td>AP+ T+I</td></tr><tr><td>T-Rex2</td><td>52.2</td><td>-</td><td>58.5</td><td>-</td><td>45.8</td><td>-</td><td>65.8</td><td>-</td><td>50.3</td><td>-</td><td>61.8</td><td>-</td></tr><tr><td>SAM 3</td><td>56.4</td><td>58.8</td><td>76.8</td><td>78.1</td><td>52.4</td><td>54.7</td><td>76.0</td><td>78.4</td><td>61.1</td><td>63.1</td><td>82.2</td><td>81.8</td></tr></table>


_Table 3 Prompting with 1 exemplar on COCO, LVIS and ODinW13. Evaluation per prompt type: T (text-only), I (image-only), and $\mathrm{T} + \mathrm{I}$ (combined text and image). $\mathrm{AP^{+}}$ is evaluated only on positives examples._


Few-Shot Adaptation. We evaluate zero- and few-shot transfer of SAM 3 on ODinW13 and RF100-VL, with their original labels as prompts. We do not perform any prompt tuning. We fine-tune SAM 3 without mask loss, and report average bbox mAP in Tab. 2. SAM 3 achieves state-of-the-art 10-shot performance, surpassing in-context prompting in Gemini and object detection experts (gDino); more details in §F.3. RF-100VL contains domains with specialized prompts that are out of SAM 3's current scope, but SAM 3 adapts through fine-tuning more efficiently than baselines.


PCS with 1 Exemplar. We first evaluate image exemplars using a single input box sampled at random from the ground truth. This can be done only on "positive" data, where each prompted object appears in the image. We report the corresponding $\mathrm{AP}^{+}$ in Tab. 3 across three settings: text prompt (T), exemplar image (I), and both text and image $(\mathrm{T} + \mathrm{I})$; SAM 3 outperforms prior state-of-the-art T-Rex2 by a healthy margin on COCO (+18.3), LVIS (+10.3), and ODinW (+20.5). See §F.2 for more details and results on SA-Co/Gold.


PCS with K Exemplars. Next, we evaluate SAM 3 in an interactive setting, simulating collaboration with a human annotator. Starting with a text prompt, we iteratively add one exemplar prompt at a time: missed ground truths are candidate positive prompts, false positive detections are candidate negative prompts. Results (Fig. 7) are compared to a perfect PVS baseline, where we simulate the user manually fixing errors using ideal box-to-mask corrections. SAM 3's PCS improves


![图片 page8-9](images/page008_img01.png)


_Figure 7 $\mathrm{cgF}_1$ vs. # of interactive box prompts for SAM 3 compared to the ideal PVS baseline, averaged over SA-Co/Gold phrases._


$\mathrm{cgF}_1$ more quickly, as it generalizes from exemplars (e.g., detecting or suppressing similar objects), while PVS only corrects individual instances. After 3 clicks, interactive PCS outperforms text-only by $+21.6 \mathrm{cgF}_1$ points and PVS refinement by $+2.0$. Performance plateaus after 4 clicks, as exemplars cannot fix poor-quality masks. Simulating a hybrid switch to PVS at this point yields gains, showing complementary.


---


Object Counting. We evaluate on object counting benchmarks CountBench and PixMo-Count to compare with several MLLMs using Accuracy $(\%)$ and Mean Absolute Error (MAE) from previous technical reports and our own evaluations. See Tab. 4 for results and §F.4 for more evaluation details. Compared to MLLMs, SAM 3 not only achieves good object counting accuracy, but also provides object segmentation that most MLLMs cannot provide.


<table><tr><td rowspan="2">Model</td><td colspan="2">CountBench</td><td colspan="2">PixMo-Count</td></tr><tr><td>MAE ↓</td><td>Acc ↑</td><td>MAE ↓</td><td>Acc ↑</td></tr><tr><td>DINO-X</td><td>0.62</td><td>82.9</td><td>0.21</td><td>85.0</td></tr><tr><td>Qwen2-VL-72B</td><td>0.28</td><td>86.7</td><td>0.61</td><td>63.7</td></tr><tr><td>Molmo-72B</td><td>0.27</td><td>92.4</td><td>0.17</td><td>88.8</td></tr><tr><td>Gemini 2.5 Pro</td><td>0.24</td><td>92.4</td><td>0.38</td><td>78.2</td></tr><tr><td>SAM 3</td><td>0.12</td><td>93.8</td><td>0.21</td><td>86.2</td></tr></table>


_Table 4 Accuracy on counting benchmarks. Gray indicates usage of training sets._


Video PCS with Text. We evaluate video segmentation with text prompts on both our SA-Co/VEval benchmark and existing public benchmarks. For SA-Co/VEval, we report $\mathrm{cgF}_1$ and pHOTA metrics (defined in §F.5) across its subsets (SA-V, YT-Temporal-1B, SmartGlasses). For public benchmarks, we use their official metrics. Baselines include GLEE, an open-vocabulary image and video segmentation model, "LLMDet + SAM 3 Tracker" (replacing our detector with LLMDet), and "SAM 3 Detector + T-by-D" (replacing our tracker with an association module based on the tracking-by-detection paradigm). In Tab. 5, SAM 3 largely outperforms these baselines, especially on benchmarks with a very large number of noun phrases. On SA-Co/VEval it reaches over $80\%$ of human pHOTA. See §F.5 for more details.


<table><tr><td rowspan="3">Model</td><td colspan="6">SA-Co/VEval benchmark test split</td><td colspan="4">Public benchmarks</td></tr><tr><td colspan="2">SA-V (2.0K NPs)</td><td colspan="2">YT-Temporal-1B (1.7K NPs)</td><td colspan="2">SmartGlasses (2.4K NPs)</td><td rowspan="2">LVVIS (1.2K NPs) test mAP</td><td rowspan="2">BURST (482 NPs) test HOTA</td><td rowspan="2">YTVIS21 (40 NPs) val mAP</td><td rowspan="2">OVIS (25 NPs) val mAP</td></tr><tr><td>$cgF_1$</td><td>pHOTA</td><td>$cgF_1$</td><td>pHOTA</td><td>$cgF_1$</td><td>pHOTA</td></tr><tr><td>Human</td><td>53.1</td><td>70.5</td><td>71.2</td><td>78.4</td><td>58.5</td><td>72.3</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>$\text{GLEE}^\dagger$ (all NPs at once)</td><td>0.1</td><td>8.7</td><td>1.6</td><td>16.7</td><td>0.0</td><td>4.7</td><td>20.8</td><td>28.4</td><td>62.2</td><td>38.7</td></tr><tr><td>$\text{GLEE}^\dagger$ (one NP at a time)</td><td>0.1</td><td>11.8</td><td>2.2</td><td>18.9</td><td>0.1</td><td>5.6</td><td>9.3</td><td>20.2</td><td>56.5</td><td>32.4</td></tr><tr><td>$\text{LLMDet}^\dagger + \text{SAM 3 Tracker}$</td><td>2.3</td><td>30.1</td><td>8.0</td><td>37.9</td><td>0.3</td><td>18.6</td><td>15.2</td><td>33.3</td><td>31.3</td><td>20.4</td></tr><tr><td>SAM 3 Detector + T-by-D</td><td>25.7</td><td>55.7</td><td>47.6</td><td>68.2</td><td>29.7</td><td>60.0</td><td>35.9</td><td>39.7</td><td>56.5</td><td>55.1</td></tr><tr><td>SAM 3</td><td>30.3</td><td>58.0</td><td>50.8</td><td>69.9</td><td>36.4</td><td>63.6</td><td>36.3</td><td>44.5</td><td>57.4</td><td>60.5</td></tr></table>


_Table 5 Video PCS from a text prompt (open-vocabulary video instance segmentation) on SA-Co/VEval and public benchmarks (see Tab. 39 for more results and analyses). SAM 3 shows strong performance, especially on benchmarks with a large number of NPs. †: GLEE and LLMDet do not perform well zero-shot on SA-Co/VEval._


PVS. We evaluate SAM 3 on a range of visual prompting tasks, including Video Object Segmentation (VOS) and interactive image segmentation. Tab. 6 compares SAM 3 to recent state-of-the-art methods on the VOS task. SAM 3 achieves significant improvements over SAM 2 on most benchmarks, particularly on the challenging MOSEv2 dataset, where SAM 3 outperforms prior work by 6.5 points. For the interactive image segmentation task, we evaluate SAM 3 on the 37 datasets benchmark introduced in Ravi et al. (2024). As shown in Tab. 7, SAM 3 outperforms SAM 2 on average mIoU. See also §F.6 and Fig. 21 for interactive video segmentation.


<table><tr><td rowspan="2">Model</td><td colspan="5">J&amp;F</td><td>G</td><td>J&amp;F</td></tr><tr><td>MOSEv1 val</td><td>DAVIS17 val</td><td>LVOSv2 val</td><td>SA-V val</td><td>SA-V test</td><td>YTVOS19 val</td><td>MOSEv2 val</td></tr><tr><td>SAMURAI</td><td>72.6</td><td>89.9</td><td>84.2</td><td>79.8</td><td>80.0</td><td>88.3</td><td>51.1</td></tr><tr><td>SAM2Long</td><td>75.2</td><td>91.4</td><td>85.9</td><td>81.1</td><td>81.2</td><td>88.7</td><td>51.5</td></tr><tr><td>SeC</td><td>75.3</td><td>91.3</td><td>86.5</td><td>82.7</td><td>81.7</td><td>88.6</td><td>53.8</td></tr><tr><td>SAM 2.1 L</td><td>77.9</td><td>90.7</td><td>79.6</td><td>77.9</td><td>78.4</td><td>89.3</td><td>$47.9^{\dagger}$</td></tr><tr><td>SAM 3</td><td>78.4</td><td>92.2</td><td>88.5</td><td>83.5</td><td>84.4</td><td>89.7</td><td>60.3</td></tr></table>


_Table 6 SAM 3 improves over SAM 2 in VOS. †: Zero-shot._


<table><tr><td rowspan="2">Model</td><td colspan="3">Avg. mIoU</td><td rowspan="2">FPS</td></tr><tr><td>1-click</td><td>3-clicks</td><td>5-clicks</td></tr><tr><td>SAM 1 H</td><td>58.5</td><td>77.0</td><td>82.1</td><td>41.0</td></tr><tr><td>SAM 2.1 L</td><td>66.4</td><td>80.3</td><td>84.3</td><td>93.0</td></tr><tr><td>SAM 3</td><td>66.1</td><td>81.3</td><td>85.1</td><td>43.5</td></tr></table>


_Table 7 Interactive image segmentation on the SA-37 benchmark._


SAM 3 Agent. We experiment with an MLLM that uses SAM 3 as a tool to segment more complex text queries (see Fig. 25). The MLLM proposes noun phrase queries to prompt SAM 3 and analyzes the returned masks, iterating until the masks are satisfactory. Tab. 8 shows that this "SAM 3 Agent" evaluated zero-shot on ReasonSeg and OmniLabel surpasses prior work without training on any referring expression segmentation or reasoning segmentation data. SAM 3 Agent also outperforms previous zero-shot results on RefCOCO+ and RefCOCOg. SAM 3 can be combined with various MLLMs, with the same set of the system prompts for all those MLLMs, showing SAM 3's robustness. See §G for more details.


---


<table><tr><td rowspan="3">Model</td><td rowspan="3">MLLM</td><td colspan="4">ReasonSeg (gIoU)</td><td colspan="4">Omnilabel (AP)</td></tr><tr><td>val</td><td colspan="3">test</td><td colspan="4">val 2023</td></tr><tr><td>All</td><td>All</td><td>Short</td><td>Long</td><td>descr</td><td>descr-S</td><td>descr-M</td><td>descr-L</td></tr><tr><td>X-SAM</td><td>Phi-3-3.8B</td><td>56.6</td><td>57.8</td><td>47.7</td><td>56.0</td><td>12.0*</td><td>17.1*</td><td>11.4*</td><td>8.8*</td></tr><tr><td>SegZero</td><td>Qwen2.5-VL 7B</td><td>62.6</td><td>57.5</td><td>-</td><td>-</td><td>13.5*</td><td>20.7*</td><td>12.4*</td><td>9.1*</td></tr><tr><td>RSVP</td><td>GPT-4o</td><td>64.7</td><td>55.4</td><td>61.9</td><td>60.3</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td colspan="2">Overall state-of-the-art†</td><td>65.0</td><td>61.3</td><td>55.4</td><td>63.2</td><td>36.5</td><td>54.4</td><td>33.2</td><td>25.5</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 7B</td><td>62.2</td><td>63.0</td><td>59.4</td><td>64.1</td><td>36.7</td><td>52.6</td><td>34.3</td><td>26.6</td></tr><tr><td>SAM 3 Agent</td><td>Llama4 Maverick</td><td>68.5</td><td>67.1</td><td>66.8</td><td>67.2</td><td>32.8</td><td>43.7</td><td>30.9</td><td>27.5</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 72B</td><td>74.6</td><td>70.8</td><td>70.3</td><td>71.0</td><td>42.0</td><td>56.0</td><td>40.4</td><td>33.2</td></tr><tr><td>SAM 3 Agent</td><td>Gemini 2.5 Pro</td><td>77.0</td><td>74.0</td><td>75.8</td><td>73.4</td><td>45.3</td><td>53.8</td><td>45.1</td><td>37.7</td></tr></table>


_Table 8 SAM 3 Agent results. Gray indicates fine-tuned results on ReasonSeg (train), * indicates reproduced results, underline indicates the main metric. †: LISA-13B-LLaVA1.5 for ReasonSeg; REAL for OmniLabel._


<table><tr><td></td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>×</td><td>50.7</td><td>0.77</td><td>65.4</td></tr><tr><td>✓</td><td>52.2</td><td>0.82</td><td>63.4</td></tr></table>


_(a) Presence head._


<table><tr><td>#/img</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>0</td><td>28.3</td><td>0.44</td><td>62.4</td></tr><tr><td>5</td><td>39.4</td><td>0.62</td><td>62.9</td></tr><tr><td>15</td><td>41.8</td><td>0.67</td><td>62.4</td></tr><tr><td>30</td><td>43.0</td><td>0.68</td><td>62.8</td></tr></table>


_(b) Hard Negatives._


<table><tr><td colspan="2">EXT SYN</td><td>HQ</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>✓</td><td>×</td><td>×</td><td>23.7</td><td>0.46</td><td>50.4</td></tr><tr><td>✓</td><td>✓</td><td>×</td><td>32.8</td><td>0.57</td><td>56.9</td></tr><tr><td>✓</td><td>×</td><td>✓</td><td>45.5</td><td>0.71</td><td>64.0</td></tr><tr><td>✓</td><td>✓</td><td>✓</td><td>47.4</td><td>0.74</td><td>63.8</td></tr></table>


_(c) Training data._


<table><tr><td>Model</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>Human</td><td>72.8</td><td>0.94</td><td>77.0</td></tr><tr><td>SAM 3</td><td>54.0</td><td>0.82</td><td>65.9</td></tr><tr><td>+ EV AI</td><td>61.2</td><td>0.86</td><td>70.8</td></tr><tr><td>+ MV AI</td><td>62.3</td><td>0.87</td><td>71.1</td></tr></table>


_(d) SAM 3 + AI verifiers._


_Table 9 Selected model and data ablations on SA-Co/Gold. Numbers across tables are not directly comparable._


Selected Ablations. In Tab. 9 we report a subset of the more extensive ablations from $\S A$. Note that the ablated models are from different, shorter training runs than the model evaluated above. The presence head boosts $\mathrm{cgF_1}$ by $+1.5$ (9a), improving image-level recognition measured by IL_MCC by $+0.05$. Tab. 9b shows that adding hard negatives significantly improves the model performance, most notably the image-level IL_MCC from 0.44 to 0.68. Tab. 9c shows that synthetic (SYN) training data improves over the external (EXT) by $+8.8\mathrm{cgF}_1$ and our high-quality (HQ) annotations add $+14.6\mathrm{cgF}_1$ on top of this baseline. We present detailed data scaling laws of both types of data in $\S A.2$, showing their effectiveness on both in-domain and out-of-domain test sets. In Tab. 9d, we show how AI verifiers can improve pseudo-labels. Replacing the presence score from SAM 3 with that score from the exhaustivity verification (EV) AI verifier boosts $\mathrm{cgF_1}$ by $+7.2$. Using the mask verification (MV) AI verifier to remove bad masks adds another 1.1 points. Overall, AI verifiers close half of the gap between SAM 3's and human performance.


Domain adaptation ablation. With domain-specific synthetic data generated by SAM 3 + AI verifiers, we show that one can significantly improve performance on a new domain without any human annotation. We hold out one of the SA-Co domains, "Food&drink", from training SAM 3 and AI verifiers. We then use three variants of training data for the novel "Food&drink" domain: high-quality AI+human annotations as in SA-Co/HQ (referred to as SA-Co/HQ-Food), synthetic annotations as in SA-Co/SYN, using AI but no humans (SA-Co/SYN-Food), and pseudo-labels generated before the AI verification step, i.e. skipping both AI verifiers and humans (PL-Food). Fig. 8 plots performance on the "Food&drink" test set of the SA-Co/Gold benchmark as each type of training data is scaled up. We mix the domain specific data and high-quality general domain data at a 1:1 ratio. PL-Food provides some improvement compared to the baseline SAM 3 (zero-shot), but is far below the other


![图片 page10-13](images/page010_img01.png)


_Figure 8 Domain adaptation via synthetic data. Synthetic (SYN) data generated by SAM 3 + AI verifiers (teacher system) achieves similar scaling behavior as human-annotated (HQ) data._


variants due to its lower quality. HQ-Food and SYN-Food show similar scaling behavior, with SYN-Food slightly lower but eventually catching up, without incurring any human annotation cost. This points to a scalable way to improve performance on new data distributions. More details are in §A.3.


---


# 7 Related Work


Promptable and Interactive Visual Segmentation. SAM (Kirillov et al., 2023) introduces "promptable" image segmentation with interactive refinement. While the original task definition included text prompts, they were not fully developed. SAM 2 (Ravi et al., 2024) extended the promptable visual segmentation task to video, allowing refinement points on any frame. SAM 3 inherits geometry-based segmentation while extending to include text and image exemplar prompts to segment all instances of a concept in images and videos.


Open-Vocabulary Detection and Segmentation in Images exhaustively labels every instance of an open-vocabulary object category with a coarse bounding box (detection) or a fine-grained pixel mask (segmentation). Recent open-vocabulary (OV) detection (Gu et al., 2021; Minderer et al., 2022) and segmentation (Ding et al., 2022; Liang et al., 2023) methods leverage large-scale vision-language encoders such as CLIP (Radford et al., 2021) to handle categories described by arbitrary text, even those never seen during training. While DETR (Carion et al., 2020) is limited to a closed set of categories seen during training, MDETR (Kamath et al., 2021) evolves the approach to condition on raw text queries. Image exemplars used as prompts to specify the desired object category (e.g., DINOv (Li et al., 2023a), T-Rex2 (Jiang et al., 2024)) present a practical alternative to text, but fall short in conveying the abstract concept of objects as effectively as text prompts. We introduce a new benchmark for OV segmentation with $>100\times$ more unique concepts than prior work.


Visual Grounding localizes a language expression referring to a region of the image with a box or mask. (Plummer et al., 2020) introduces phrase detection as both deciding whether the phrase is relevant to an image and localizing it. GLIP (Li et al., 2022b) and GroundingDino (Liu et al., 2023) formulate object detection as phrase grounding, unifying both tasks during training. MQ-GLIP (Xu et al., 2023) adds image exemplars to text as queries. Building on this trend toward models supporting multiple tasks and modalities, GLEE (Wu et al., 2024a) allows text phrases, referring expressions, and visual prompts for category and instance grounding in both images and videos. Unlike SAM 3, GLEE does not support exemplars or interactive refinement. LISA (Lai et al., 2024) allows segmentation that requires reasoning, while OMG-LLaVa (Zhang et al., 2024a) and GLaMM (Rasheed et al., 2024) generate natural language responses interleaved with corresponding segmentation masks, with GLaMM accepting both textual and optional image prompts as input. Some general-purpose MLLMs can output boxes and masks (Gemini2.5 (Comanici et al., 2025)) or points (Molmo (Deitke et al., 2025)). SAM 3 can be used as a “vision tool” in combination with an MLLM (§6).


Multi-Object Tracking and Segmentation methods identify object instances in video and track them, associating each with a unique ID. In tracking-by-detection methods, detection is performed independently on each frame to produce boxes and confidence scores, followed by association of boxes using motion-based and appearance-based matching as in SORT (Bewley et al., 2016; Wojke et al., 2017), Tracktor (Bergmann et al., 2019), ByteTrack (Zhang et al., 2022c), SAM2MOT (Jiang et al., 2025), or OC-SORT (Cao et al., 2023). An alternative is an end-to-end trainable architecture that jointly detects and associates objects, e.g., TrackFormer (Meinhardt et al., 2022), TransTrack (Sun et al., 2020), or MOTR (Zeng et al., 2022). TrackFormer uses a DETR-like encoder-decoder that initializes new tracks from static object queries and auto-regressively follows existing tracks with identity-preserving track queries. A challenge with joint models is the conflict between detection and tracking (Feichtenhofer et al., 2017; Yu et al., 2023a), where one needs to focus on semantics while the other on disentangling identities, even if their spatial locations overlap over time. SAM 3 is a strong image detector tightly integrated into a tracker to segment concepts in videos.


# 8 Conclusion


We present Segment Anything with Concepts, enabling open-vocabulary text and image exemplars as prompts in interactive segmentation. Our principal contributions are: (i) introducing the PCS task and SA-Co benchmark, (ii) an architecture that decouples recognition, localization and tracking and extends SAM 2 to solve concept segmentation while retaining visual segmentation capabilities, (iii) a high-quality, efficient data engine that leverages the complimentary strengths of human and AI annotators. SAM 3 achieves state-of-the-art results, doubling performance over prior systems for PCS on SA-Co in images and videos. That said, our model has several limitations. For example, it struggles to generalize to out-of-domain terms, which could be mitigated by automatic domain expansion but requires extra training. We discuss this and other limitations of our model in §B. We believe SAM 3 and the SA-Co benchmark will be important milestones and pave the way for future research and applications in computer vision.


---


# 9 Acknowledgements


We would like to thank the following people for their contributions to the SAM 3 project: Alex He, Alexander Kirillov, Alyssa Newcomb, Ana Paula Kirschner Mofarrej, Andrea Madotto, Andrew Westbury, Ashley Gabriel, Azita Shokpour, Ben Samples, Bernie Huang, Carleigh Wood, Ching-Feng Yeh, Christian Puhrsch, Claudette Ward, Daniel Bolya, Daniel Li, Facundo Figueroa, Fazila Vhora, George Orlin, Hanzi Mao, Helen Klein, Hu Xu, Ida Cheng, Jake Kinney, Jiale Zhi, Jo Sampaio, Joel Schlosser, Justin Johnson, Kai Brown, Karen Bergan, Karla Martucci, Kenny Lehmann, Maddie Mintz, Mallika Malhotra, Matt Ward, Michelle Chan, Michelle Restrepo, Miranda Hartley, Muhammad Maaz, Nisha Deo, Peter Park, Phillip Thomas, Raghu Nayani, Rene Martinez Doehner, Robbie Adkins, Ross Girshik, Sasha Mitts, Shashank Jain, Spencer Whitehead, Ty Toledano, Valentin Gabeur, Vincent Cho, Vivian Lee, William Ngan, Xuehai He, Yael Yungster, Ziqi Pang, Ziyi Dou, Zoe Quake. We also thank the IDEA team for granting us DINO-X and T-Rex2 access to benchmark them on the SA-Co/Gold dataset.


# Appendix


# A Ablations 13


A.1 Model Ablations. 13


A.2 Image Training Data Ablations 14


A.3 Automatic Domain Adaptation 16


A.4 Image Data Engine Annotation Speed 17


A.5 Video Data Engine Annotation Speed 18


A.6 Video Training Data Ablations 18


# B Limitations 18


# C Model Details. 19


C.1 Model Architecture. 19


C.2 Image Implementation Details. 19


C.3 Video Implementation Details 22


C.4 Model Training 23


# D Data Engine Details. 26


D.1 Media Pool 26


D.2 SA-Co Ontology 26


D.3 Phase 1: Human Verification. 26


D.4 Phase 2: Human + AI Verification 28


D.5 Phase 3: Scaling and Domain Expansion 32


D.6 Phase 4: Video Annotation 32


# E SA-Co Dataset and Metric Details 34


E.1 SA-Co Training Data 34


E.2 SA-Co Evaluation Benchmark 35


E.3 Metrics 37


E.4 Human Performance on SA-Co 37


E.5 Additional Dataset Examples 39


---


F Additional Experiments and Details 45


F.1 PCS with NPs on Images 45


F.2 Visual Exemplars and Interactivity 46


F.3 Few-Shot Fine-tuning 46


F.4 Object Counting 47


F.5 Video PCS Details 48


F.6 PVS Details. 50


G SAM 3 Agent. 55


G.1 Agent Design 55


G.2 Qualitative Analysis 58


G.3 Full Quantitative Results 60


H Model and annotation cards. 64


H.1 Data annotation card 64


H.2 Model card 65


# A Ablations


# A.1 Model Ablations


Presence Token. We first ablate the impact of the presence token and the approach to its training. The presence token is included in the decoder (discussed further in §C.2), together with the object queries, and predicts a concept presence score. The presence score receives gradients only on the PCS task during joint training and is always supervised with the presence (or absence) of the concept in the image using a binary cross-entropy loss. Using a presence token to decouple presence and localization brings significant gains in performance, particularly on IL_MCC, see Tab. 9a.


When used with a presence score, we found that it is better for the box/mask object scores to not receive gradients when a concept is an image-level negative, see Setting (a) in Tab. 10. Note that this is in contrast to the approach in typical DETR variants, where all individual object scores are supervised negatively to reflect the absence of the concept in the image, see Setting (b) in Tab. 10. We find that (b) works worse than (a) when used with the presence score. When a concept is present in the image, individual object queries always receive classification supervision based on Hungarian matching. Setting (a) is consistent with our recognition-localization decoupled design, where the presence score is responsible for recognition (existence in the image) and the object scores are responsible for localization (i.e., rank the best match to the positive ground-truth highest among all the proposals).


During inference, we use the product of the global presence score and the object score as the total object score. In Setting (c), we explored directly supervising the total object scores (instead of the typical object scores) as positive or negative (as determined by matching); this setting can slightly improve the overall $\mathrm{cgF}_1$, but is less flexible as the presence and object scores are jointly calibrated, e.g. such a model is less amenable to conditioning on a concept known to be present in the image. Finally, Setting (d) in Tab. 10 investigates detaching the presence score from the computation graph while supervising the total scores, but this does not improve over (c).


Training with presence can be considered as a form of post-training and occurs in Stage 3 (see §C.4.1) of our training pipeline. By default, ablations do not undergo this stage unless otherwise mentioned.


Vision and Text Encoder. While SAM 2 uses an MAE (He et al., 2022) pre-trained Hiera (Ryali et al., 2023) vision encoder for its strong localization capability and efficiency for the more geometric PVS task, SAM 3 also needs strong semantic and linguistic understanding with broad coverage. We adapted PE (Bolya et al., 2025) for the vision and text encoders of SAM 3, so that a large and diverse set of concepts is seen in Stage 1 of training, while producing aligned image and text encoders. In Tab. 11, we compare performance with


---


<table><tr><td rowspan="2"></td><td rowspan="2">Supervise mask scores only when concept present</td><td rowspan="2">Supervise total score</td><td rowspan="2">Sup. total score, detach presence</td><td colspan="3">SA-Co/Gold</td></tr><tr><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>a.</td><td>✓</td><td>×</td><td>×</td><td>54.0</td><td>0.82</td><td>65.5</td></tr><tr><td>b.</td><td>×</td><td>×</td><td>×</td><td>52.2</td><td>0.81</td><td>64.2</td></tr><tr><td>c.</td><td>✓</td><td>✓</td><td>×</td><td>54.9</td><td>0.83</td><td>66.0</td></tr><tr><td>d.</td><td>✓</td><td>×</td><td>✓</td><td>53.6</td><td>0.83</td><td>64.9</td></tr></table>


_Table 10 Supervision strategy for object/mask scores for a model with a presence token. We find the best supervision strategy is to supervise mask scores only for positive concepts and to supervise the presence and mask scores separately, although their product is used as the total object score during inference._


Hiera and DINOv2 (Oquab et al., 2024); since these vision encoders lack an aligned text encoder, we use DistilRoBERTa-base (Sanh et al., 2019). We find PE to be the best overall choice of vision backbone, and using its own aligned text encoder provides further gains over PE with an unaligned text baseline. Use of PE enables strong robustness in SAM 3 (here measured by AP on COCO-O, demonstrating good object detection across various domain shifts, e.g. "sketch", "cartoon", "painting", etc).


<table><tr><td>Encoder (patch size)</td><td>SA-Co/Gold (cgF1)</td><td>COCO-O (AP)</td></tr><tr><td>PE-L+ (14)</td><td>43.2</td><td>42.5</td></tr><tr><td>PE-L+ (14) w/ DistilRoBERTa</td><td>38.1</td><td>39.6</td></tr><tr><td>DINOv2-L (14)</td><td>35.3</td><td>31.9</td></tr><tr><td>Hiera-L (16)</td><td>32.8</td><td>22.0</td></tr></table>


_Table 11 Choice of encoders. As SAM 3 needs both semantic visual and linguistic understanding, we find PE's aligned image and text encoders work well._


Implementation Details. The image resolution is set to 1008 px, 1008 px, 1152 px for PE, DINOv2, Hiera, respectively, ensuring the same number of tokens in the detector due to their differences in patch size. All vision encoders used global attention in only a subset of the layers, using windowed $(24 \times 24$ tokens) attention otherwise. Since Hiera is a hierarchical multiscale encoder, we set the window size to $24 \times 24$ in stage 3 of the encoder, which has most of the FLOPs. Since PE is capable of using relative positional information via RoPE (Su et al., 2021; Heo et al., 2024), we include relative positional embeddings in global layers for Hiera and DINOv2 following Bolya et al. (2024). All models are trained using SA-Co/HQ viewing 5 million samples over the course of training. Recipe is separately optimized for each choice of encoder. Tokens from the respective vision encoders are downsampled by $2 \times 2$ to 1296 tokens before being passed to the fusion encoder and detector.


# A.2 Image Training Data Ablations


Setup. We adopt a simplified, lighter model and training strategy for ablations in this section. Specifically, we use (i) a stride-28 (instead of 14) variant of SAM 3 using $4 \times$ fewer tokens in the detector, (ii) limit to $45\%$ of the entire SA-Co/SYN dataset and adopt, (iii) shorter training schedules and do not run "presence post-training" (see §A), (iv) evaluations are on an internal version of SA-Co/Gold, which has slightly lower human performance than the public version ($\mathrm{cgF}_1$: internal 70.8 vs public 72.8). This allows running ablations more efficiently (but results in lower absolute accuracy vs. SAM 3). We observed similar trends when training at scale.


SAM 3 Training Data. Tab. 9c analyzes the impact of various SA-Co training data subsets. Training with even with just SA-Co/EXT shows comparable performance with best external models on SA-Co/Gold (see OWLv2's and DINO-X's performance in Tab. 1), indicating a strong base model. Adding synthetic data SA-Co/SYN into the training mix results in significantly improved performance. The performance further increases after adding the high-quality SA-Co/HQ data due to its quality and distributional similarity with SA-Co/Gold. Although SA-Co/HQ is large-scale and in-domain with SA-Co/Gold, SA-Co/SYN shows further gains on SA-Co/Gold when added on top of SA-Co/HQ.


SA-Co/HQ Scaling Law. Tab. 12 investigates scaling behavior of the SA-Co/HQ training data. For this ablation, the data mix is sampled randomly from the entire SA-Co/HQ (collected from the three phases in §4) at a fixed


---


<table><tr><td rowspan="2">Training data</td><td colspan="3">SA-Co/Gold (All) 
(in-domain)</td><td colspan="3">SA-Co/Gold-MetaCLIP 
(in-domain)</td><td colspan="3">SA-Co/Gold-Wiki-Food&amp;Drink 
(in domain)</td></tr><tr><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>SA-Co/EXT</td><td>23.7</td><td>0.46</td><td>50.4</td><td>21.5</td><td>0.45</td><td>47.7</td><td>20.5</td><td>0.45</td><td>45.4</td></tr><tr><td>+ 1% SA-Co/HQ</td><td>34.0</td><td>0.57</td><td>59.6</td><td>30.8</td><td>0.56</td><td>54.6</td><td>33.4</td><td>0.55</td><td>60.7</td></tr><tr><td>+ 4% SA-Co/HQ</td><td>37.3</td><td>0.62</td><td>59.6</td><td>35.4</td><td>0.65</td><td>54.7</td><td>39.2</td><td>0.66</td><td>58.9</td></tr><tr><td>+ 10% SA-Co/HQ</td><td>40.0</td><td>0.65</td><td>60.9</td><td>37.5</td><td>0.67</td><td>55.7</td><td>46.6</td><td>0.71</td><td>65.5</td></tr><tr><td>+ 20% SA-Co/HQ</td><td>42.2</td><td>0.68</td><td>61.8</td><td>38.5</td><td>0.69</td><td>56.1</td><td>50.3</td><td>0.74</td><td>67.6</td></tr><tr><td>+ 100% SA-Co/HQ</td><td>45.5</td><td>0.71</td><td>64.0</td><td>40.3</td><td>0.71</td><td>57.1</td><td>53.3</td><td>0.77</td><td>68.9</td></tr><tr><td>Teacher (Human)</td><td>70.8</td><td>0.944</td><td>71.5</td><td>63.3</td><td>0.936</td><td>67.7</td><td>77.3</td><td>0.964</td><td>80.2</td></tr></table>


_Table 12 SA-Co/HQ scaling. SA-Co/EXT data alone is not enough to solve SA-Co/Gold, training on SA-Co/HQ scales well with increasing amount of data. Human performance given as an estimated range where applicable, see §E.4 for details. Ablations use a lighter model and training setting vs. SAM 3._


<table><tr><td rowspan="2">Training data</td><td colspan="3">SA-Co/Gold (All)</td><td colspan="3">SA-Co/Gold-MetaCLIP 
(in-domain)</td><td colspan="3">SA-Co/Gold-Wiki-Food&amp;Drink 
(out-of-domain)</td></tr><tr><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>SA-Co/EXT</td><td>23.7</td><td>0.46</td><td>50.4</td><td>21.5</td><td>0.45</td><td>47.7</td><td>20.5</td><td>0.45</td><td>45.4</td></tr><tr><td>+ 1% SA-Co/SYN</td><td>30.1</td><td>0.52</td><td>57.3</td><td>31.0</td><td>0.58</td><td>53.7</td><td>26.2</td><td>0.44</td><td>59.1</td></tr><tr><td>+ 4% SA-Co/SYN</td><td>30.7</td><td>0.53</td><td>56.9</td><td>31.8</td><td>0.59</td><td>53.6</td><td>27.8</td><td>0.47</td><td>59.7</td></tr><tr><td>+ 15% SA-Co/SYN</td><td>32.1</td><td>0.56</td><td>56.6</td><td>33.3</td><td>0.62</td><td>53.4</td><td>29.5</td><td>0.50</td><td>59.5</td></tr><tr><td>+ 45% SA-Co/SYN</td><td>32.8</td><td>0.57</td><td>56.9</td><td>34.5</td><td>0.64</td><td>53.6</td><td>30.1</td><td>0.51</td><td>59.6</td></tr><tr><td>Teacher (SAM 3 + AI verifiers)</td><td>55.4</td><td>0.84</td><td>65.3</td><td>48.3</td><td>0.83</td><td>58.5</td><td>59.0</td><td>0.87</td><td>68.1</td></tr></table>


_Table 13 SA-Co/SYN scaling. SAM 3 benefits from increasing SA-Co/SYN data, both on MetaCLIP which is in-domain with the synthetic data, and on Wiki concepts which are out-of-domain of the synthetic data. The teacher that generated the SA-Co/SYN data consists of an older version of SAM 3, and AI verifiers from the SAM 3 data engine. Ablations use a lighter model and training setting vs. SAM 3._


percentage. We also report scaling behavior on two specific subsets of SA-Co/Gold: the MetaCLIP Xu et al. (2024b) subset annotated with generic caption-derived NPs, and Wiki-Food&Drink subset annotated with fine-grained NPs from SA-Co Ontology nodes. SA-Co/HQ improves performance on both subsets as expected, since they are from the same distribution (in-domain). We also report the Teacher (Human) performance in the last row. Due to the simplified setting, the gap between SAM 3 and Human is larger than that of the best SAM 3 model.


SA-Co/SYN Scaling Law. Tab. 13 shows that SAM 3 scales well with SA-Co/SYN data on SA-Co/Gold benchmark as it benefits from the large scale concepts captured from image captions generated by Llama4 and alt-text associated with the images, for both the in-domain MetaCLIP subset and the out-of-domain WikiFood&Drink subset within the SA-Co/Gold benchmark. The last row shows the Teacher performance (an older version of SAM 3 and AI verifiers) is much better than the student, and explains why SA-Co/SYN is useful. When comparing the SA-Co/SYN in Tab. 13 and SA-Co/HQ in Tab. 12, the lower in-domain performance gap on MetaCLIP (42.5 vs. 49.0) comes from the relatively weaker annotation quality of SA-Co/SYN, due to lacking of the human correction step. The gap is larger on the out-of-domain Wiki-Food&Drink set (37.4 vs. 59.9), because SA-Co/SYN only covers the MetaCLIP images and noun phrases from a captioning model; see Table 26. We also show in Fig. 9 that with additional in-domain synthetic data, we can close the performance gap on SA-Co/Gold-Wiki-Food&Drink subset without any human involvement.


Hard Negatives. We ablate the number of hard negative noun phrases in SA-Co/HQ per image in Tab. 9b. We show that increasing the number of negatives improves SAM 3 performance across all metrics, most notably IL_MCC. Hard negatives are phrases that are not present in the image but that (a previous generation of) SAM 3 predicts masks for, i.e., they are adversarial to (a previous generation of) SAM 3. Training on such difficult distractors helps improve the image-level classification performance captured by the IL_MCC metric.


SAM 3 and AI Verifiers. AI verifiers improve performance over the final SAM 3 model alone on the PCS task, as shown in Tab. 9d, with per-domain results in Tab. 14. We first replace the presence score from SAM 3 with a presence score from the Exhaustivity Verification (EV) AI verifier (given the image and noun phrase with no objects as input, the probability of not exhaustive, defined in Tab. 22). This results in a +7.2 point


---


gain in $\mathrm{cgF_1}$, from both IL_MCC and $\mathrm{pmF_1}$. The reason why EV presence score can even improve $\mathrm{pmF_1}$ is because it serves as a better calibration of object scores. Then we apply the Mask Verification (MV) AI verifier to each mask, and remove the rejected masks. This results in a further $+1.1$ point gain in $\mathrm{cgF_1}$. The system closes nearly half the gap between SAM 3 and human performance, which indicates potential further improvements of SAM 3 by scaling up the SA-Co/SYN data and SAM 3 model size.


<table><tr><td></td><td colspan="3">Average</td><td colspan="3">Metaclip</td><td colspan="3">SA-1B</td><td colspan="3">Crowded</td><td colspan="3">Food&amp;Drink</td><td colspan="3">Sports Equip.</td><td colspan="3">Attributes</td><td colspan="3">Wiki-Common</td></tr><tr><td></td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>SAM 3</td><td>54.0</td><td>0.82</td><td>65.9</td><td>46.9</td><td>0.81</td><td>58.8</td><td>53.8</td><td>0.85</td><td>63.4</td><td>60.3</td><td>0.90</td><td>67.0</td><td>56.1</td><td>0.81</td><td>69.2</td><td>63.0</td><td>0.89</td><td>71.0</td><td>54.2</td><td>0.76</td><td>71.0</td><td>42.9</td><td>0.70</td><td>60.9</td></tr><tr><td>SAM 3 +EV</td><td>61.2</td><td>0.86</td><td>70.8</td><td>54.2</td><td>0.85</td><td>64.0</td><td>56.0</td><td>0.89</td><td>62.9</td><td>61.3</td><td>0.88</td><td>69.8</td><td>67.6</td><td>0.86</td><td>78.5</td><td>67.5</td><td>0.89</td><td>75.6</td><td>71.1</td><td>0.91</td><td>77.8</td><td>51.1</td><td>0.76</td><td>67.1</td></tr><tr><td>SAM 3 +EV&amp;MV</td><td>62.3</td><td>0.87</td><td>71.1</td><td>56.5</td><td>0.88</td><td>64.3</td><td>58.0</td><td>0.90</td><td>64.2</td><td>62.7</td><td>0.89</td><td>70.6</td><td>68.0</td><td>0.86</td><td>78.9</td><td>67.2</td><td>0.88</td><td>76.0</td><td>70.9</td><td>0.92</td><td>77.4</td><td>52.3</td><td>0.79</td><td>66.5</td></tr><tr><td>Human</td><td>72.8</td><td>0.94</td><td>77.0</td><td>64.1</td><td>0.94</td><td>68.5</td><td>64.3</td><td>0.97</td><td>66.6</td><td>70.4</td><td>0.94</td><td>75.3</td><td>78.3</td><td>0.96</td><td>81.2</td><td>80.4</td><td>0.97</td><td>83.1</td><td>80.2</td><td>0.95</td><td>84.4</td><td>71.6</td><td>0.89</td><td>80.1</td></tr></table>


_Table 14 Per-domain results of SAM 3 + AI verifiers in Tab. 9d._


# A.3 Automatic Domain Adaptation


With domain-specific synthetic data generated by SAM 3 + AI verifiers, we show that one can significantly improve performance on a new domain without any human annotation. We select "Food & drink" concepts with MetaCLIP images as the new domain. We generated three variants of synthetic training data on this "Food & drink" domain, while ensuring that no data from the new domain was used in training the AI annotators (including SAM 3 and AI verifiers):


- PL-Food: We select "Food&drink" Wiki nodes and mine images from MetaCLIP (refer to Concept Selection, Offline Concept Indexing and Online Mining steps in §D.4 for more details on data mining). For pseudo-annotating fine-grained "Food&drink" concepts, we use Wiki ontology to identify relevant coarse-grained concepts that SAM 3 works well on and prompt SAM 3 with them to generate masks. This data is similar to typical pseudo-labeled data used in prior work for detection self-training (e.g. Minderer et al. (2022)).


- SA-Co/SYN-Food: PL-Food is cleaned by AI verifiers: MV AI verifier to remove bad masks, and EV AI verifier to verify exhaustivity/negativity of (image, noun phrase) pairs, as the AI verification step in Fig. 5.


- SA-Co/HQ-Food: PL-Food is cleaned by human verifiers for both MV and EV tasks. For non-exhaustive datapoints after EV, human annotators further manually correct them, as the "Correct" step in Fig. 5.


We study the data scaling law of these three variants by evaluating their performance on the Wiki-Food&Drink subset of the SA-Co/Gold benchmark.


![图片 page16-10](images/page016_img01.png)


_(a) Data scaling mixing pre-training data at a 1:1 ratio._


![图片 page16-12](images/page016_img02.png)


_(b) Data scaling without mixing pre-training data._


_Figure 9 Domain adaptation via synthetic data. (a) SAM 3 + AI verifiers (teacher system) can annotate synthetic (SYN) data in new domains (e.g., fine-grained food concepts) and achieve similar scaling behavior as with human-annotated (HQ) data. (b) Not mixing in high-quality pre-training data can limit performance gains when fine-tuning on new domains, particularly when using synthetic data._


We train the models in 2 steps to isolate the impact of the data from the new domain from other data as well as to amortize training costs. We first pre-train a base model using "SA-Co/HQ minus SA-Co/HQ-Food" to


---


establish base capability and a common starting point. Next, we fine-tune the same base model with the three data variants in two settings: with or without mixing the pre-training data.


Fig. 9a shows the scaling law when mixing the synthetic data for the new domain with the pre-training data in a 1:1 ratio. We observe some improvement with PL-Food compared to baseline, but there is a large gap between the other variants due to its lower quality. SA-Co/HQ-Food and SA-Co/SYN-Food have similar data scaling behavior, with SA-Co/SYN-Food slightly lower but eventually catching up, without incurring any human annotation cost. The model trained on SA-Co/SYN-Food eventually surpassed the performance of its teacher system, thanks to the high quality pre-training data mixed during the fine-tuning.


Fig. 9b shows the scaling law when fine-tuned with only synthetic data for the new domain. All three data variants result in poorer performance than that in Fig. 9a. In this setting, there is a larger gap between SA-Co/HQ-Food and SA-Co/SYN-Food reflecting the lower quality of SA-Co/SYN-Food (mainly lack of exhaustivity due to no human correction). Comparing Fig. 9a and 9b, it is beneficial to include high-quality general-domain data when fine-tuning SAM 3 on new domains, particularly when using synthetic data.


# A.4 Image Data Engine Annotation Speed


Tab. 15 measures the speedup in the SAM 3 data engine from adding AI verifiers when collecting data on a new domain with fine-grained concepts. We use the same setup as Fig. 9, annotating Wiki-Food&Drink data generated with a data engine where neither SAM 3 nor AI verifiers have been trained on Wiki-Food&Drink data. We annotate the same set of image-NP pairs in four settings:


- Human (NP Input). A human annotator is given a single image noun-phrase pair from SA-Co/HQ-Food, and is required to manually annotate all instance masks. No mask proposals or AI-verifiers are used in the loop.


- Human (Mask Input). The same annotation task as "NP input" but in this setting, the human annotators starts with PL-Food, i.e., image noun-phrase pairs with mask proposals generated by SAM 3.


- Engine (All Human) Similar to Phase 1 in the SAM 3 data engine, humans start with PL-Food, and sequentially perform 3 tasks: Mask Verification, Exhaustivity Verification and Correction. All three tasks are performed by humans.


- Engine (Full) Similar to Phase 3 in the SAM 3 data engine, Mask Verification and Exhaustivity Verification tasks are completed by AI verifiers, and Correction is done by humans i.e human annotators in the manual annotation task start with SA-Co/SYN-Food.


<table><tr><td>Task</td><td>Human from NP</td><td>Human from masks</td><td>Engine - all human</td><td>Engine - full</td></tr><tr><td>Time for datamix (sec)</td><td>90</td><td>86</td><td>50</td><td>23</td></tr><tr><td>Time for positive NP (sec)</td><td>236</td><td>205</td><td>207</td><td>152</td></tr><tr><td>Time for negative NP (sec)</td><td>71</td><td>70</td><td>30</td><td>6</td></tr></table>


_Table 15 Data engine efficiency - Image. AI verifiers significantly increase throughput, allowing humans to focus on challenging cases and the manual correction task. AI verifiers allow for a 5x speed up on negative phrases and a $36\%$ speed up for positive phrases. The time for datamix is calculated based on $88.5\%$ negatives in SA-Co/HQ, see Tab. 24 for dataset composition. Timing is calculated based on the Wiki-Food&Drink domain. Compared to captioner-based domains, fine-grained domains require more research by annotators to understand the concept, leading to much higher annotation times for negative phrases and amortized time per mask._


Tab. 15 shows that a version of the SAM 3 model and AI verifiers that were never trained in this new domain double the throughput of the data engine. AI verifiers also allow verifying generated hard negative NPs at scale with close to no human-annotator involvement. As SAM 3 and AI verifiers are updated with the collected data and improve, human annotators need to manually correct fewer errors. This leads to increasingly higher throughput and the collection of more challenging data for a given amount of human annotation time.


In Tab. 23, we show that AI verifiers achieve a similar even better performance on the MV and EV tasks than human verifiers, so the quality of annotations from these four settings are similar.


---


# A.5 Video Data Engine Annotation Speed


Using the same settings as described in Section A.4, we evaluate annotation speed in the video data engine by comparing Human (NP Input) and Engine (All Human) on positive video-NP pairs from SA-Co/VEval - SA-V. In contrast to the image data engine, we observe that starting with PL increases annotation time, but also improves exhaustivity by providing annotators with more visual cues and candidate masklets.


<table><tr><td>Task</td><td>Human (NP Input)</td><td>Engine (All Human) human</td></tr><tr><td>Time for positive NP (sec)</td><td>2307</td><td>3221</td></tr><tr><td>Number masklets per video-NP pair</td><td>2.52</td><td>2.76</td></tr><tr><td>Total masklets</td><td>1700</td><td>1860</td></tr></table>


_Table 16 Data engine efficiency - Video. While the Engine - all human is $40\%$ slower than Human (NP Input) annotation, it yields $9\%$ more masklets. PL helps annotators focus on regions where masklets potentially exist, but the additional time required for human verification and correction increases the overall annotation time. This experiment was run before the final cleaning of the SA-Co/VEval data, so the number of masklets might differ from the released version._


# A.6 Video Training Data Ablations


We analyze how much the SAM 3 model benefits from the videos and annotations in SA-Co/VIDEO obtained through the video data engine, which are used in Stage 4 (video-level) training (described further in §C.4.1). Specifically, we train the model with a varying amount of masklets from SA-Co/VIDEO as VOS training data, and evaluate the resulting checkpoints on SA-Co/VEval under the VOS task with the $\mathcal{I}\&\mathcal{F}$ metric. The results are shown in Tab. 17, where adding masklets collected with noun phrases through the video data engine (as additional Stage 4 training data) improves the $\mathcal{I}\&\mathcal{F}$ performance on both SA-Co/VEval and public benchmarks such as DAVIS17 (Pont-Tuset et al., 2017a) and SA-V (Ravi et al., 2024).


<table><tr><td>Stage-4 training data</td><td>SA-Co/VEval YT-1B
val J&amp;F</td><td>SA-Co/VEval SA-V
val J&amp;F</td><td>DAVIS17
val J&amp;F</td><td>SA-V
val J&amp;F</td><td>SA-V
test J&amp;F</td></tr><tr><td>using SAM 2 video data only</td><td>80.7</td><td>84.9</td><td>91.6</td><td>77.0</td><td>77.1</td></tr><tr><td>+ 25% SA-Co Train videos</td><td>80.9</td><td>85.3</td><td>91.3</td><td>75.2</td><td>76.9</td></tr><tr><td>+ 50% SA-Co Train videos</td><td>81.2</td><td>85.3</td><td>91.5</td><td>76.7</td><td>77.0</td></tr><tr><td>+ 75% SA-Co Train videos</td><td>81.4</td><td>85.9</td><td>91.7</td><td>76.5</td><td>78.5</td></tr><tr><td>+ 100% SA-Co Train videos</td><td>81.4</td><td>86.5</td><td>91.5</td><td>77.4</td><td>78.0</td></tr></table>


_Table 17 Scaling analysis on SA-Co/VEval under Stage 4 (video-level) training, evaluated on multiple benchmarks through the Video Object Segmentation (VOS) task under the $\mathcal{I}\&\mathcal{F}$ metric. Note that "SA-Co/VEval YT-1B" and "SA-Co/VEval SA-V" refer to the subset of SA-Co/VEval built upon YT-Temporal-1B videos and SA-V videos respectively, while "SA-V" referred to the VOS evaluation dataset released in Ravi et al. (2024)._


# B Limitations


SAM 3 shows strong performance on the PCS task in images and videos but has limitations in many scenarios.


SAM 3 struggles to generalize to fine-grained out-of-domain concepts (e.g., aircraft types, medical terms) in a zero-shot manner, especially in niche visual domains (e.g., thermal imagery). Concept generalization for PCS is inherently more challenging than the class-agnostic generalization to new visual domains for the PVS task, with the latter being the key that enables SAM and SAM 2 to be successfully applied zero-shot in diverse settings. Our experiments show that SAM 3 is able to quickly adapt to new concepts and visual domains when fine-tuned on small quantities of human-annotated data (Tab. 2). Further, we show that we can improve the performance in a new domain without any human involvement (Fig. 9), using domain-specific synthetic data generated using our data engine.


From our formulation of the PCS task, SAM 3 is constrained to simple noun phrase prompts and does not support multi-attribute queries beyond one or two attributes or longer phrases including referring expressions. We show that when combined with an MLLM, SAM 3 is able to handle more complex phrases (§6 and §G).


---


In the video domain, SAM 3 tracks every object with a SAM 2 style masklet, which means the cost of SAM 3 inference scales linearly with the number of objects being tracked. To support real-time inference (30 FPS) on videos in practical applications (e.g., a web demo), we parallelize the inference over multiple GPUs: up to 10 objects on 2 H200s, up to 28 objects on 4 H200s, and up to 64 objects on 8 H200s. Further, under the current architecture, there is no shared object-level contextual information to aid in resolving ambiguities in multi-object tracking scenarios. Future developments could address this through shared global memory across multiple objects, which would also improve inference efficiency.


Supporting concept-level interactivity for PCS, alongside instance-level interactivity for PVS, poses several challenges. To support instance-level modifications without affecting all other instances of the concept, we enforce a hard "mode-switch" within the model from concept to instance mode. Future work could include more seamlessly interleaving concept and instance prompts.


# C Model Details


# C.1 Model Architecture


Our architecture is broadly based on the SAM series (Ravi et al., 2024; Kirillov et al., 2023) and DETR (Carion et al., 2020) and uses a (dual) encoder-decoder transformer architecture, see Fig. 10 for an overview. SAM 3 is a generalization of SAM 2, supporting the new Promptable Concept Segmentation (PCS) task as well as the Promptable Visual Segmentation (PVS) task (Ravi et al., 2024). The design supports multimodal prompts (e.g., text, boxes, points) and interactivity, in images and videos.


SAM 3 has $\sim 850\mathrm{M}$ parameters, distributed as follows: $\sim 450\mathrm{M}$ and $\sim 300\mathrm{M}$ for the vision and text encoders (Bolya et al., 2025), and $\sim 100\mathrm{M}$ for the detector and tracker components. We next discuss the detector architecture for images followed by the tracker components built on top of it for video.


![图片 page19-6](images/page019_img01.png)


_Figure 10 SAM 3 architecture. New components are in yellow, SAM 2 (Ravi et al., 2024) in blue and PE (Bolya et al., 2025) in cyan._


# C.2 Image Implementation Details


The image detector is an encoder-decoder transformer architecture. We describe its details in this section.


Image and Text Encoders. The image and text encoders are Transformers (Vaswani et al., 2017) trained using constrastive vision language training using 5.4 billion image-text pairs following Perception Encoder (PE) (Bolya et al., 2025), see §C.4.1 for training details. As in SAM 2, the vision encoder uses windowed attention (Ryali et al., 2023; Li et al., 2022d) and global attention in only a small subset of layers (4 out


---


of 32), where an image of 1008 pixels is divided into $3 \times 3$ non-overlapping windows of 336 pixels each. The vision encoder uses RoPE (Su et al., 2021; Heo et al., 2024) in each layer and windowed absolute positional embeddings as in Bolya et al. (2024). The text encoder is causal, with a maximum context length of 32.


As in Ravi et al. (2024), we use a streaming approach, ingesting new frames as they become available. We run the PE backbone only once per frame for the entire interaction, which can span multiple forward/backward propagation steps through a video. The backbone provides unconditioned tokens (features/embeddings) representing each frame to the dual-encoder consisting of the fusion encoder described below and memory attention for video.


Geometry and Exemplar Encoder. The geometry and exemplar encoder is primarily used to encode image exemplars (if present) for the PCS task. It is additionally used to encode visual prompts for the PVS task on images as an auxiliary functionality that is primarily used to include pre-training data for the PVS task in stages-2,-3 of training (see §C.4.1), to enable a more modular training approach.


Each individual image exemplar is encoded using positional embedding, label embedding (positive or negative) and ROI-pooled visual features that are concatenated (comprising "exemplar tokens") and processed by a small transformer. Visual prompts (points, boxes) for auxiliary training are encoded in a similar manner, comprising "geometry tokens". It is possible for neither "geometry tokens" nor "exemplar tokens" to be present (e.g. when only a text prompt is used). The geometry or exemplar tokens attend to each other via self-attention and also cross-attend to the frame-embeddings of the corresponding (unconditioned) frame from the image encoder.


Fusion Encoder. The text and geometry/exemplar tokens together constitute the prompt tokens. The fusion encoder accepts the unconditioned frame-embeddings and conditions on prompt tokens using a stack of 6 transformer blocks with self- and cross-attention (to prompt tokens) layers followed by an MLP. We use vanilla self-attention operations. The output of the fusion encoder are the conditioned frame-embeddings.


Decoder. The decoder architecture follows Carion et al. (2020); Kamath et al. (2021) as a starting point and is a stack of 6 transformer blocks. $Q$ learned object queries (not to be confused with prompts) self-attend to each other and cross attend to the prompts tokens (made up of text and geometry/exemplar tokens) and conditioned frame-embeddings, followed by an MLP. We use box-to-pixel relative position bias (Lin et al., 2023) in the cross-attention layers attending to the conditioned frame-embeddings.


Following standard practice in stronger DETR variants, we use iterative box refinement (Zhu et al., 2020), look-forward-twice (Zhang et al., 2022a) and hybrid matching (Jia et al., 2022) and Divide-And-Conquer (DAC) DETR (Hu et al., 2023). By default, we use $ Q = 200 $ object queries. Bounding boxes and scores are predicted using dedicated MLPs and accept the object queries as input.


Presence Head. Classifying each object in isolation is often difficult, due to insufficient information, and may require contextual information from the rest of the image. Forcing each object query to acquire such global awareness is however detrimental, and can conflict with the localization objectives that are by nature very local. To address this, we propose decomposing the classification problem into two complementary components: a global-level classification that determines object presence within the entire image, and a local-level localization that functions as foreground-background segmentation while preventing duplicate detections. Formally, we add the following structure: instead of predicting $ p(\text{query}_i $ matches NP) directly, we break it down as


$p(\mathrm{query}_i$ matches NP) $= p(\mathrm{query}_i$ matches NP | NP appears in image) $\cdot p(\mathrm{NP}$ appears in image).


To compute $ p(\mathrm{NP}) $ appears in image), we use a presence token, which is added to our decoder and then fed through an MLP classification head. Crucially, the presence score is shared by all object queries. The per-query classification loss is kept as usual, but to account for the decomposition, we only compute it when the NP is present in the image (see §A.1 for ablations on supervision strategy). The same decomposition is applied to the semantic segmentation head, where we reuse the same presence score, and train the binary mask head only on the positive examples.


Besides being more robust to false positives, decomposing the prediction in this manner is also more flexible, e.g. in typical counting tasks, we already know the NP is present in the image and instead want to know how many instances are present - in this case we can simply set $ p(\mathrm{NP}) $ is present in frame) = 1. The presence token is concatenated with the object queries in all operations, but is excluded from DAC.


---


We also learn 4 geometric queries. Their function is similar to the 4 geometric queries in SAM 1 and 2 (where they were called "output tokens") and are used to perform the PVS on individual image or video frames during the stags-2,-3 of training, see §C.4.1. The prompts are provided by the "geometry tokens" in the form of visual prompts. The presence score is set to 1 when performing the PVS task on a single frame, as the target is known to be present in the frame.


Segmentation Head. The segmentation head is adapted from MaskFormer (Cheng et al., 2021). Semantic segmentation and instance segmentation share the same segmentation head. The conditioned features from the fusion encoder are used to produce semantic segmentation masks, while instance segmentation additionally uses the decoder's output object queries. "Multi-scale" features are provided to the segmentation head using SimpleFPN (Li et al., 2022d), since the vision encoder is a (single-scale) ViT.


Handling Ambiguity. Experimentally, if we train a SAM 3 model without handling ambiguities as described in §2 in any way, we observe that the model tends to predict several valid but conflicting interpretations of the phrase. This is expected; if in our training dataset a given phrase has two distinct interpretations, and roughly half the data is annotated assuming the first one, while the other half follows the second one, then the solution that minimizes the training loss is to output both interpretations with $50\%$ confidence. However, this behavior is undesirable for end-users, because it produces conflicting, sometimes overlapping masks.


To address this issue, we add an ambiguity head to our model. Similar to SAM 1 and 2, this head is a mixture of experts, where we train in parallel $ K $ experts, and only supervise the expert that gets the lowest loss (winner-takes-all). We find that $ K = 2 $ performs the best and that it is more difficult to train $ K > 3 $ experts due to mode collapse.


For a mixture of $K$ experts, each producing an output $y_{k}$ with loss $\mathcal{L}_k$, the mixture loss is a weighted average:


$$
\mathrm {L o s s :} \quad \mathcal {L} _ {\mathrm {M o E}} = \sum_ {k = 1} ^ {K} p _ {k} \mathcal {L} _ {k} \qquad \mathrm {G r a d i e n t :} \quad \frac {\partial \mathcal {L} _ {\mathrm {M o E}}}{\partial \theta_ {j}} = p _ {j} \frac {\partial \mathcal {L} _ {j}}{\partial \theta_ {j}}.
$$


In our winner-takes-all variant, only the expert with the lowest loss receives gradient:


$$
\mathrm {L o s s :} k ^ {\star} = \underset {k} {\arg \min} \mathcal {L} _ {k}, \qquad \mathcal {L} _ {\mathrm {W T A}} = \mathcal {L} _ {k ^ {\star}}
$$


$$
\begin{array}{r l} \mathrm {G r a d i e n t :} & \frac {\partial \mathcal {L} _ {\mathrm {W T A}}}{\partial \theta_ {j}} = \left\{ \begin{array}{l l} \frac {\partial \mathcal {L} _ {k ^ {\star}}}{\partial \theta_ {j}}, & \mathrm {i f} j = k ^ {\star}, \\ 0, & \mathrm {o t h e r w i s e .} \end{array} \right. \end{array}
$$


Backpropagating the loss only through the expert which received the minimal loss allows each expert to specialize to one kind of interpretation. This behavior is illustrated in Fig. 11.


![图片 page21-10](images/page021_img01.png)


_(a) Original image_


![图片 page21-12](images/page021_img02.png)


_(b) Prediction by Expert 1_


![图片 page21-14](images/page021_img03.png)


_(c) Prediction by Expert 2_


_Figure 11 Two interpretations of the noun phrase "large circular shape" learned by two Experts (SA-1B image)._


While this strategy allows experts to specialize, it does not explicitly select which expert should be used at inference time. To resolve this, we train a classification head that predicts the expert that has the highest


---


probability of being correct. The classification head is trained in a supervised fashion with a cross entropy loss, by predicting which expert obtained the minimal loss during training. The Ambiguity head adjusts only the classification logits, leaving masks, boxes, and presence scores unchanged. We train it on top of a frozen SAM 3 model.


Finally, to detect overlapping instances, we compute the Intersection-over-Minimum (IoM) between masks. IoM is more effective than Intersection-over-Union (IoU) for identifying nested instances. With the ambiguity head, we obtain a $15\%$ reduction in overlapping instances.


# C.3 Video Implementation Details


The tracker architecture follows Ravi et al. (2024), which we briefly describe for convenience followed by a discussion of the disambiguation strategies we introduce.


Tracker. The image encoder is the shared PE (Bolya et al., 2025) with the detector and provides unconditioned tokens to the memory attention using a separate neck. The memory attention receives these unconditioned PE tokens stacks self- and cross- attention layers that condition the current frame's tokens on spatial memories and corresponding object pointers in the memory bank. Memories are encoded by fusing a frame's mask prediction with the unconditioned PE tokens from the image encoder and placed in the memory bank.


As in Ravi et al. (2024), the decoder includes an occlusion head to indicate the likelihood of the object of interest being visible in the current frame. During inference, the occlusion score may also be used to select frames to place in the memory bank adaptively.


SAM introduced the ability to output multiple valid masks when faced with ambiguity about the object being segmented in an image. For example, when a person clicks on the tire of a bike, the model can interpret this click as referring to only the tire or the entire bike and output multiple predictions. In videos, this ambiguity can extend across video frames. For example, if in one frame only the tire is visible, a click on the tire might relate to just the tire, or as more of the bike becomes visible in subsequent frames, this click could have been intended for the entire bike. To handle this ambiguity, SAM 2 predicts multiple masks at each step of the video. If further prompts do not resolve the ambiguity, the model selects the mask with the highest predicted IoU for the current frame for further propagation in the video although other strategies are possible.


Disambiguation Strategy. As outlined in §3, tracking in videos can suffer from ambiguities in mask propagation, false predictions from the detector, or limitations of IoU-based matching in crowded scenes with highly overlapping objects. In this section, we present the details of the temporal disambiguation strategies used to address these challenges. We begin by introducing the notation used throughout this section.


Let $\mathcal{D}_{\tau}$ and $\hat{\mathcal{M}}_{\tau}$ denote the set of detector outputs and the set of tracker's predicted masks on frame $\tau$ respectively. We define a frame-wise matching function $\Delta_i(\tau)$ for a masklet $i$ on frame $\tau$ as


$$
\Delta_ {i} (\tau) = \left\{ \begin{array}{l l} + 1, & \mathrm {i f} \exists d \in \mathcal {D} _ {\tau} \mathrm {s . t .} \mathrm {I o U} (d, \hat {\mathcal {M}} _ {\tau} ^ {i}) > \mathrm {i o u \_ t h r e s h o l d} \\ - 1, & \mathrm {o t h e r w i s e ,} \end{array} \right.
$$


where $\hat{\mathcal{M}}_{\tau}^{i}$ is the predicted output mask of object $i$ on frame $\tau$. In addition, we define a Masklet Detection Score (MDS) over an interval $[t,t^{\prime}]$ as $S_{i}(t,t^{\prime}) = \sum_{\tau = t}^{t^{\prime}}\Delta_{i}(\tau)$. This score measures how a masklet is consistently matched to a detection within a temporal window. The first frame in which object $i$ appears is denoted $t_{\mathrm{first}}^{i}$.


Track Confirmation Delay. To reduce spurious and duplicate masklets, we delay the output of the model slightly. Specifically, the output at frame $\tau$ is shown only after observing frame $\tau + T$. This delay provides temporal context for validating candidate masklets before outputting their masks. By default, we use $T = 15$ which achieves good accuracy at a slight delay cost of around half a second for 30 frames per second videos. During the delay, we apply the following two criteria to remove unconfirmed or duplicate masklets as follows.


Removal of Unconfirmed Masklets. A candidate masklet is considered unconfirmed within the confirmation window $[t, t + T]$ if their MDS is below a threshold, $S_{i}(t, t + T) < V$, and the masklet first appears within the window $t_{\mathrm{first}}^{i} \geq t$. If both conditions are satisfied within the confirmation delay, we remove the masklet from the tracker's state. We choose $V = 0$, requiring that the masklet has to be matched to a detection for at least


---


half of the frames within the confirmation delay period to be confirmed. This strategy helps reject some false positive detections and not track them.


Removal of Duplicate Masklets. If the tracker temporarily fails to predict a mask for an object in some frames, but the detector continues to detect the object during those frames, this can lead to the creation of a new masklet for the same object. As a result, two masklets may end up tracking the same object: the original (older) masklet, and a new masklet that is initiated during the period when the tracker missed the object. To resolve this issue, during the confirmation delay period, if two masklets consistently overlap with the same detection, we remove the one that started later. Specifically, two masklets $ i $, $ j $ are considered duplicates on frame $ \tau $ if there exists a detection $ d \in \mathcal{D}_{\tau} $ such that $ \mathrm{IoU}(\hat{\mathcal{M}}_{\tau}^{i},d) \geq \mathrm{iou\_threshold} $ and $ \mathrm{IoU}(\hat{\mathcal{M}}_{\tau}^{j},d) \geq \mathrm{iou\_threshold} $. If the two masklets $ i $ and $ j $ are found to be duplicates for at least $ \lceil T/2 \rceil $ frames, we remove the one with the latest first appearance $ t_{\mathrm{first}} $ only if it first appeared within the confirmation window $ [t,t + T] $. Empirically, we find that using iou_threshold $ = 0.1 $ gives the best results.


Masklet Suppression. For confirmed masklets that were not removed during the confirmation delay, we apply an additional suppression step: if a masklet's MDS over its entire lifetime falls below zero at any frame $\tau$ (i.e. $S_{i}(t_{\mathrm{first}}^{i},\tau) < 0$), we suppress its output by zeroing out its mask. However, we retain the masklet in the tracker's state, allowing for the possibility that the object may be confirmed in future frames. This strategy primarily addresses ambiguous detections, such as objects entering the scene near the boundary. For example, if only a person's hands are visible as they enter the frame, the detector may be unable to determine whether the object matches the text prompt (e.g., impossible to distinguish between a man and a woman). In such cases, if the detector subsequently fails to detect the object after it fully enters the scene, the masklet suppression criterion ensures that these masklets are suppressed, unless they are consistently matched with new detections.


Periodic Re-Prompting. In challenging scenarios involving occlusions or visually similar distractor objects, the tracker may lose track of the target object. To address such tracking failures, we periodically re-prompt the tracker using the latest detection outputs. Specifically, on every $N$-th frame $\tau$, we compare each detection $d \in \mathcal{D}_{\tau}$ with the tracker's current predictions $\hat{\mathcal{M}}_{\tau}$. If a detection $d$ has a high overlap with the tracker's prediction (i.e., $\mathrm{IoU}(d, \hat{\mathcal{M}}_{\tau}^{i}) \geq 0.8$) and both the detection score and the masklet prediction score exceed a confidence threshold of 0.8, we re-initialize the tracker for that object using the detection output mask. We observed that re-prompting is most effective on frames where objects are not occluded and are fully visible, which motivates our choice of high confidence thresholds. In our experiments, we set $N = 16$ by default. This periodic re-prompting helps the tracker recover from temporary failures and maintain accurate object tracking throughout the video.


Detection-Guided Re-Prompting. In cases where the tracker's predictions may drift and its predicted masks become leaky, we employ the detectors' outputs. For each frame $\tau$, we compare every detection $d \in \mathcal{D}_{\tau}$ with the tracker's current predictions $\hat{\mathcal{M}}_{\tau}$. If the highest-matching detection $d$ has a low bounding box IoU (i.e., $\mathrm{IoU}_{\mathrm{bbox}}(d, \hat{\mathcal{M}}_{\tau}^{i}) < 0.85$) with the corresponding tracker prediction $\hat{\mathcal{M}}_{\tau}^{i}$, we recondition the tracker for that object using the latest detector output. This approach ensures that the tracker remains synchronized with reliable detection results.


The impact of these strategies is ablated in Tab. 39, and they show quantitative improvements across our evaluation sets.


# C.4 Model Training


# C.4.1 Training Stages


SAM 3 is trained in 4 stages, with each stage introducing new capabilities or refining existing capabilities. Tab. 18 lists the data used in each stage.


Stage 1. Perception Encoder (PE) pre-training (Bolya et al., 2025), which pre-trains the image and text encoders with 5.4 billion image-text pairs. In addition to broad concept coverage, this stage is key for robustness (see §A.1). Since the vision encoder has to support multiple tasks (while also not being too large) we opt for an “L+” size; The vision and text encoders are transformers with 450M and 300M parameters


---


respectively. We largely follow Bolya et al. (2025), but do not use distillation and do not perform video fine-tuning in this stage.


Stage 2. This stage is for detector pre-training and trains the (image-level) detector as well as the vision and text encoders with large-scale image segmentation data (including video frames as images). This stage uses both psuedo-labelled and human-annotated data, see Tab. 18. The main goal of this stage is broad concept coverage of (image, noun phrase, masks) tuples. At the end of this stage, the model is able to do open-vocabulary object detection, instance and semantic segmentation across many domains fairly well.


An additional goal of this stage is to prepare the base model for tasks in subsequent stages. To prepare for the PCS task, (image, noun phrase) pairs are randomly ($p = 0.2$) converted into visual queries (i.e. noun phrase is dropped) or augmented with input bounding boxes ($p = 0.2$).


Besides training for the PCS task, in this stage, the model is also pre-trained on the visually prompted PVS task. This is done by adding 4 decoder queries specific to this task following the design of SAM 1 & 2. Training data includes images (e.g., SA-1B) and videos frames (e.g., SA-V), see Tab. 18; the number of interactivity steps is restricted to 4 for efficiency. We largely follow the settings from Ravi et al. (2024), but use the Align loss (Cai et al., 2024) in lieu of the IoU prediction loss, co-opting the classification head for object queries for this task.


We train for $\sim 95\mathrm{k}$ iterations with a batch size of 896 with $5\mathrm{k}$ warm up and cooldown steps using AdamW (Loshchilov & Hutter, 2019). We apply layer-wise learning rate decay (Clark et al., 2020) of 0.9 to the vision encoder. We use a reciprocal square-root schedule (Zhai et al., 2022) and weight decay of 0.1. We use an initial learning rate of $5e - 4$, $1e - 4$ for vision and text encoder and $1e - 3$ for all other components. For boxes, we use $L_{1}$ and gIoU losses with weights of 5 and 2. Classification loss uses a weight of 100 and focal and dice losses use weights of 200 and 10 respectively. The encoder and decoder use a dropout of 0.1.


Stage 3. This stage further trains the model with the highest-quality human annotated image segmentation data, expands the interactivity capabilities and introduces post-training to improve detection performance.


Specifically, in terms of interactivity, (a) in the PVS task, the number of interactivity steps is increased to 7 and (b) interactivity is introduced into the PCS task, where positive or negative exemplars are provided based on model error, as described next. We iteratively sample box prompts to mimic the real user policy. Positive boxes are sampled from false negative errors, and we prompt their corresponding ground-truth boxes. Negative boxes are sampled from high-confidence false positive predictions that do not have significant overlap with ground truths. At each iteration, the box inputs are added on top of the previous ones. If both a valid positive and negative box exist, we randomly select one; if no valid candidates are available, no additional prompt is given. The process is repeated for 5 iterations.


The expanded interactivity in the PCS and PVS in this stage significantly slows down training, but the extensive pretraining with limited interactivity for the PVS and no interactivity for PCS (but using image exemplars together with text prompts) prepares the model well to ensure that a short stage 3 is sufficient.


This stage retains only the highest quality, exhaustivity verified data (e.g., SA-Co/SYN is dropped) and introduces a presence token (and presence loss) to better model presence of target segments and their location location greatly increasing the precision of the model. The presence loss is a binary cross-entropy loss with weight of 20. All learning rates are lowered by a factor of 0.025. We train for $\sim 5\mathrm{k}$ iterations with a batch size of 512, with other settings identical to stage 2.


Stage 4. For video, the tracker decoder is trained on top of the frozen backbone. Freezing the backbone at this stage is made possible by pre-training on VOS data in previous stages at the video frame level. This stage retains the strong spatial grounding of the previous stage and focuses on spatial-temporal tracking without degrading other capabilities. We use a batch size of 512, train for $\sim 190\mathrm{k}$ iterations using a cosine schedule with a peak learning rate of $5.0e^{-4}$ and a linear warmup of 1k iterations. We supervise the model's outputs using a weighted sum of losses: a linear combination of focal and dice losses for mask prediction, mean absolute error (MAE) loss for IoU prediction, and cross-entropy loss for object occlusion prediction, with respective weights of 20:1:1:1. For multi-mask predictions, we only apply supervision to the mask with the lowest segmentation loss. If a frame's ground truth does not include a mask, we do not supervise any mask outputs for that frame; however, we always supervise the occlusion prediction head, which determines


---


whether a mask should be present. As in Ravi et al. (2024), we further fine-tune the tracker with a longer temporal context using 16-frame and 32-frame videos for 60k iterations, while scaling the learning rate by a factor of 0.1.


<table><tr><td>Dataset</td><td>Ingested As</td><td>Train</td><td>Test</td><td>Stage 1</td><td>Stage 2</td><td>Stage 3</td><td>Stage 4</td></tr><tr><td colspan="8">Promptable Visual Segmentation-In Images</td></tr><tr><td>SA-1B</td><td>Image</td><td>✓</td><td></td><td></td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>SA-Co/VIDEO</td><td>Frames</td><td>✓</td><td></td><td></td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>SA-Co/VIDEO-EXT</td><td>Frames</td><td>✓</td><td></td><td></td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>SA-37</td><td>Image</td><td></td><td>✓</td><td></td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td colspan="8">Promptable Visual Segmentation-In Videos</td></tr><tr><td>SA-Co/VIDEO</td><td>Video</td><td>✓</td><td></td><td></td><td></td><td></td><td>✓</td></tr><tr><td>SA-Co/VIDEO-EXT</td><td>Video</td><td>✓</td><td></td><td></td><td></td><td></td><td>✓</td></tr><tr><td>SA-V val</td><td>Video</td><td></td><td>✓</td><td></td><td></td><td></td><td>✓</td></tr><tr><td>LVOSv2</td><td>Video</td><td></td><td>✓</td><td></td><td></td><td></td><td>✓</td></tr><tr><td colspan="8">Promptable Concept Segmentation-In Images</td></tr><tr><td>SA-Co/SYN</td><td>Image</td><td>✓</td><td></td><td></td><td>✓</td><td></td><td></td></tr><tr><td>SA-Co/HQ</td><td>Image</td><td>✓</td><td></td><td></td><td>✓</td><td>✓</td><td></td></tr><tr><td>SA-Co/EXT</td><td>Image</td><td>✓</td><td></td><td></td><td>✓</td><td>✓</td><td></td></tr><tr><td>SA-Co/VIDEO</td><td>Frames</td><td>✓</td><td></td><td></td><td>✓</td><td>✓</td><td></td></tr><tr><td>SA-Co/Gold</td><td>Image</td><td></td><td>✓</td><td></td><td>✓</td><td>✓</td><td></td></tr><tr><td>SA-Co/Silver</td><td>Image</td><td></td><td>✓</td><td></td><td>✓</td><td>✓</td><td></td></tr><tr><td>SA-Co/Bronze, SA-Co/Bio</td><td>Image</td><td></td><td>✓</td><td></td><td>✓</td><td>✓</td><td></td></tr><tr><td colspan="8">Promptable Concept Segmentation-In Videos</td></tr><tr><td>SA-Co/VEval</td><td>Video</td><td></td><td>✓</td><td></td><td></td><td></td><td>✓</td></tr></table>


_Table 18 Dataset usage across different tasks and training stages._


# C.4.2 Additional Training Settings


Data augmentation. For the PCS task, we apply the following transformations:


- Geometric: We use some cropping and resizing to vary the aspect ratios and help with small objects. The input resolution of our model is always a fixed square (usually $1008 \times 1008$). During evaluation, the images are resized to this size, without respecting their aspect ratio. During training, we apply our augmentations, and pad if the resulting size is smaller than $1008 \times 1008$. We found it important to randomly distribute the padding on all sides, to avoid creating biases towards one particular region of the image. If the dataset does not contain notions of left and right, we also apply random horizontal flips.


- Semantic: When training on datasets with a closed vocabulary, we leverage our mapping toWikidata to further enhance the training. There are three main ways we can leverage the ontology: (i) to sample synonyms, which expand the vocabulary of the model; (ii) to sample negatives (typically, if the dataset is exhaustively annotated, we can sample any node in the graph that corresponds to a category and is not present in the image); and (iii) to ensure the hierarchy closure of the concepts (for example, if we have some annotations for "canoe" and "boat" in the same image, we need to make sure that all the "canoe" objects are also labeled as "boat" since a canoe is a type of boat).


- **Safety:** To prevent the model from randomly making predictions for unsafe concepts, we randomly sample some of them at train time and add them as negatives. These concepts mainly include slurs of all kinds. We also try to prevent the model from making predictions for subjective and non-visual adjectives, especially when applied to a person. This includes flattering ones (such as "a smart person") as well as derogatory ones (such as "a dull person").


- Mosaics: On some datasets, we further increase the complexity of the images by doing mosaics (Bochkovskiy et al., 2020). The maximal grid size of our mosaics is $3 \times 3$, and we sample any configuration that is at most that, including irregular ones, as long as the constituents are still square. For example, in a $3 \times 3$ regular grid, we can have a large image that effectively covers a $2 \times 2$ area, and use $1 \times 1$ for the remaining 5 slots. Unifying different images can be tricky in an open vocabulary setting, since there is no guarantee that concepts are exhaustively annotated. For example, if one image has a car annotated, but the second


---


does not (neither as positive nor negative), then we do not know if the second image has a car or not, and thus could create some labeling noise. To avoid this, we only mosaic datasets that have low chance of such missing annotations (either closed vocabulary ones, or some created with specific mining patterns). To merge annotations, we again rely on the wikidata mapping if available, otherwise rely on plain-text queries to merge appropriately.


# D Data Engine Details


The overview of the SAM 3 data engine's components is shown in Fig. 5. In this section we provide further details of how each component is implemented in the image (phases 1-3) and video (phase 4) versions of the engine. The datasets collected in each phase and the performance improvements are in Tab. 19.


# D.1 Media Pool


The media (image and video) pool consists of many sources with varying visual domains, from web-scraped data to datasets collected for specialized domains such as art, food, or driving. Tab. 26 lists the datasets used to mine media for each subset of the SA-Co training data. The web-scraped images and alt captions are sourced from MetaCLIP (Xu et al., 2024b), a curated version of CommonCrawl. We further expand coverage by mining media from a large pool with the help of a curated ontology. Compared to previous works such as OWLv2 (Minderer et al. (2024)) which mainly rely on uncurated web-scraped data, our target mining strategy resulted in coverage of 12 media domains.


# D.2 SA-Co Ontology


To track and improve the coverage and overall distribution of concepts in our data, we build a custom SA-Co ontology of visual concepts from Wikidata (Vrandecic & Krötzsch, 2014), which covers a comprehensive set of entities and offers hierarchical information with its graph data structure. We manually select high-level Wikidata nodes (e.g., Human, Mammals) and recursively include all of their descendants. The resulting 22.4 million nodes are classified into 17 top-level categories (e.g. animal, furnishing & home) and 72 sub-categories (e.g., birds, home appliance). The full list of categories and Wikidata node counts are shown in Tab. 20. We further develop a mapping process that can map an arbitrary NP to a node in the SA-Co ontology by leveraging a retrieval model (Sentence-BERT) to source candidate nodes and an AI annotator as judge (Llama 3.2) to select the closest match. This mapping is used to track the distribution of nodes in the dataset (see Fig. 12) as well as to create negative phrases (see below for details).


# D.3 Phase 1: Human Verification


Data Mining. During this phase, we randomly sample images from MetaCLIP.


Proposing NPs. We generate image-level captions using the BLIP-2 captioner (Li et al., 2023b) followed by the spaCy parser (Honnibal et al., 2020) to parse the caption into NPs.


Proposing Masks. We prompt an off-the-shelf open-vocabulary detector, FIBER (Dou et al., 2022) or OWLv2 (Minderer et al., 2024) with the noun phrase and use the resulting boxes to prompt SAM 2 to generate mask proposals.


<table><tr><td rowspan="2"></td><td colspan="3">SA-Co/HQ</td><td colspan="2">SA-Co/SYN</td><td colspan="2">SA-Co/EXT</td><td colspan="2">SA-Co/VIDEO</td><td colspan="3">SAM 3 performance</td></tr><tr><td>#img</td><td>#img-NP</td><td>#annotation domains</td><td>#img</td><td>#img-NP</td><td>#img</td><td>#img-NP</td><td>#vid</td><td>#vid-NP</td><td>SA-Co/Gold (cgF1)</td><td>SA-Co/Silver (gCF1)</td><td>SA-Co/VEval (test pHOTA)</td></tr><tr><td>Phase 1</td><td>1.2M</td><td>4.3M</td><td>1</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Phase 2</td><td>2.4M</td><td>122.2M</td><td>5</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>21.4</td><td>18.9</td><td>-</td></tr><tr><td>Phase 3</td><td>1.6M</td><td>19.5M</td><td>15</td><td>39.4M</td><td>1.7B</td><td>9.3M</td><td>136.6M</td><td>-</td><td>-</td><td>54.4</td><td>50.5</td><td>-</td></tr><tr><td>Phase 4</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>52.5K</td><td>134.3K</td><td>54.5</td><td>50.1</td><td>63.9</td></tr></table>


_Table 19 Data engine phases and SAM 3 progress._


---


<table><tr><td>1. animals</td><td>2.3M</td></tr><tr><td>insects &amp; crustaceans</td><td>1.7M</td></tr><tr><td>molluscs</td><td>188.4K</td></tr><tr><td>other animals</td><td>166.5K</td></tr><tr><td>fish &amp; other chordates</td><td>85.7K</td></tr><tr><td>birds</td><td>52.4K</td></tr><tr><td>mammals</td><td>38.3K</td></tr><tr><td>reptiles</td><td>28.2K</td></tr><tr><td>echinoderms</td><td>23.0K</td></tr><tr><td>amphibians</td><td>14.2K</td></tr><tr><td>2. art, history &amp; religion</td><td>3.1M</td></tr><tr><td>artworks</td><td>3.1M</td></tr><tr><td>collectibles</td><td>10.2K</td></tr><tr><td>religious objects</td><td>9.1K</td></tr><tr><td>flags</td><td>8.3K</td></tr><tr><td>musical instruments</td><td>4.9K</td></tr><tr><td>gemstones</td><td>526</td></tr><tr><td>art material</td><td>438</td></tr><tr><td>3. buildings &amp; locations</td><td>2.7M</td></tr><tr><td>places</td><td>2.4M</td></tr><tr><td>geographical features</td><td>343.2K</td></tr><tr><td>4. celestial</td><td>9.2K</td></tr><tr><td>meteorological phenomena</td><td>5.3K</td></tr><tr><td>space related</td><td>3.1K</td></tr><tr><td>light related</td><td>734</td></tr><tr><td>5. documents &amp;OCR</td><td>201.3K</td></tr><tr><td>glyphs</td><td>173.4K</td></tr><tr><td>logos</td><td>21.0K</td></tr><tr><td>documents</td><td>6.0K</td></tr><tr><td>cards</td><td>435</td></tr><tr><td>infographics</td><td>324</td></tr><tr><td>GUI &amp; layout elements</td><td>135</td></tr><tr><td>maps</td><td>23</td></tr></table>


<table><tr><td>6. electronics</td><td>10.2K</td></tr><tr><td>electronics</td><td>6.9K</td></tr><tr><td>cameras</td><td>3.3K</td></tr><tr><td>7. equipments</td><td>14.9K</td></tr><tr><td>military equipments</td><td>10.2K</td></tr><tr><td>sport equipments</td><td>2.0K</td></tr><tr><td>safety equipments</td><td>1.2K</td></tr><tr><td>medical equipments</td><td>1.1K</td></tr><tr><td>agricultural machinery</td><td>458</td></tr><tr><td>8. fashion &amp; beauty</td><td>7.5K</td></tr><tr><td>fashion</td><td>3.9K</td></tr><tr><td>beauty &amp; healthcare products</td><td>3.7K</td></tr><tr><td>9. food &amp; drinks</td><td>33.1K</td></tr><tr><td>dishes</td><td>12.9K</td></tr><tr><td>other food</td><td>6.7K</td></tr><tr><td>fruits</td><td>6.6K</td></tr><tr><td>drinks</td><td>6.3K</td></tr><tr><td>vegetables</td><td>621</td></tr><tr><td>10. furnishing &amp; home</td><td>2.7K</td></tr><tr><td>furnishing</td><td>1.3K</td></tr><tr><td>home appliances</td><td>486</td></tr><tr><td>stationery</td><td>472</td></tr><tr><td>household supplies</td><td>417</td></tr><tr><td>11. human</td><td>11.3M</td></tr><tr><td>humans</td><td>11.0M</td></tr><tr><td>occupations</td><td>140.8K</td></tr><tr><td>fictional characters</td><td>87.8K</td></tr><tr><td>gestures &amp; expressions</td><td>158</td></tr></table>


<table><tr><td>12. object parts</td><td>101.9K</td></tr><tr><td>body parts</td><td>75.8K</td></tr><tr><td>other object parts</td><td>26.1K</td></tr><tr><td>13. other products</td><td>3.5K</td></tr><tr><td>other products</td><td>2.7K</td></tr><tr><td>celebration supplies</td><td>384</td></tr><tr><td>animal-related products</td><td>359</td></tr><tr><td>tobacco products</td><td>51</td></tr><tr><td>14. patterns &amp; material</td><td>896.6K</td></tr><tr><td>material</td><td>885.6K</td></tr><tr><td>patterns &amp; shapes</td><td>10.9K</td></tr><tr><td>15. plants &amp; fungi</td><td>1.5M</td></tr><tr><td>plants</td><td>1.1M</td></tr><tr><td>fungi</td><td>376.4K</td></tr><tr><td>16. tools &amp; appliances</td><td>14.5K</td></tr><tr><td>other appliances</td><td>6.5K</td></tr><tr><td>toys</td><td>3.0K</td></tr><tr><td>tools</td><td>1.8K</td></tr><tr><td>kitchenware</td><td>1.6K</td></tr><tr><td>containers</td><td>945</td></tr><tr><td>light sources</td><td>672</td></tr><tr><td>17. transportation</td><td>258.1K</td></tr><tr><td>watercraft</td><td>178.9K</td></tr><tr><td>land vehicles</td><td>41.6K</td></tr><tr><td>aircraft</td><td>27.4K</td></tr><tr><td>other vehicles</td><td>6.4K</td></tr><tr><td>transport infrastructures</td><td>3.7K</td></tr><tr><td>construction machines</td><td>100</td></tr></table>


_Table 20 SA-Co ontology top-level categories and sub-categories with corresponding node counts in Wikidata._


Verification (Human). Verification of mask proposals consists of two tasks which can be performed by human or AI annotators: mask quality verification and mask exhaustivity verification. In Phase 1, verification is done by humans only. Each human verifier works exclusively on one task type.


- Mask Verification (MV). Given a triplet of an image, a noun phrase and a set of candidate masks for that phrase, the task is to accept or reject each of the masks. A mask is accepted if it matches the given noun phrase and is high quality (no holes, coverage issues, etc.) If the mask is unrelated to the phrase, or low quality, it is rejected.


- Exhaustivity Verification (EV). All accepted masks from the verification task are sent to an exhaustivity check. Given an image, noun phrase, and any accepted masks that passed the previous mask verification for that phrase, the task is to decide whether or not the accepted masks (if any) exhaustively cover all instances of the phrase in the image. If there are unmasked instances of the phrase, annotators decide whether or not at least one of the remaining instances is separable, or if the remaining instances are too crowded together to separate. Phrases that are annotated as non-exhaustive from this step are sent to the correction task. Phrases that are annotated as exhaustive are directly sent to final annotations.


Correction. Human annotators are given the same input as the exhaustivity task: an image, noun phrase, and any (0 or more) accepted masks from the mask verification task. Annotators manually add individual masks for the unmasked instances of the noun phrase by prompting SAM 1 with clicks in a browser based tool. If there are non-separable occurrences of the phrase, annotators use special group masks to indicate that the mask covers more than a single instance. The output of the task is a complete set of instance and/or group masks covering all pixels in the image corresponding to the noun phrase. Noun phrases that are not present are submitted with no masks. If it is not possible to reach a complete set of masks due to mask complexity, the annotator rejects the job.


---


In each task, annotators are given the ability to reject the image-NP pairing if they decide the phrase is un-maskable as a set of objects (e.g. "it", "blue") or if after research they are still unsure if it is present (e.g., fine-grained species of animals). Filtering out vague phrases and allowing annotators to be unsure increases the consistency and agreement in the resulting annotations.


# D.4 Phase 2: Human + AI Verification


Data Mining. We use a retrieval model (including Perception Encoder, DINoV2, and MetaCLIPv2) for mining concepts that are challenging and not prevalent in the caption NPs from Phase 1. We leverage our SA-Co ontology to determine the list of candidate concepts, followed by offline concept indexing and online mining from MetaCLIP.


- Concept Selection. We use a taxonomy-guided mining strategy to balance the overall ontological distribution, expand concept coverage and enhance performance on long-tail and fine-grained phrases. Two groups of concepts are selected from the SA-Co Ontology for targeted mining: Wiki-Common are nodes judged by an LLM to be common concepts, Wiki-FG are all nodes from the "sports equipment" and "food and drink" sub-graphs, chosen to test the model's ability to generalize to very fine-grained concepts like "kefir", "pastille", "kettlebell".


- Offline Concept Indexing. For every new concept, we collect reference images from Wikipedia and compute their K-dimensional embedding offline. We aggregate the embeddings from all reference images resulting in a single embedding per concept. We repeat the process across all N concepts resulting in an $\mathbf{N}^{*}\mathbf{K}$ dimensional offline index.


- Online Mining. Relevant images for each concept are retrieved using both image and text based mining. With image-based retrieval, we compute the embedding on every image, run KNN on the offline concept index followed by top-k sampling, and apply a threshold before mapping it to a specific concept. With text-based retrieval, we compute CLIP based similarity scores between the text embedding from input concepts and image embeddings from the corpus and apply a threshold before mapping the image to a specific concept.


The following additional mining strategies are used to further refine the selection.


- Image-Type Balancing. Web datasets are usually dominated by a few types of images such as ads or product photos. To avoid over-representation of certain image types, we use a MLLM (Llama 3.2) and prompt it zero-shot to classify an image into image types (such as ads, product photos, indoor and outdoor scenes, infographics), and sample based on a type-agnostic probability.


Proposing NPs. We improve this step to generate higher-quality and more diverse noun phrases.


- Image-Level Captioner and Parser. We use an image captioning model (Llama 3.2) to generates image-level captions and a phrase parser (Llama 3.1) that proposes noun phrases given the caption. The Llama 3.2 captioning model improved concept recall compared to BLIP-2 from Phase 1. The phrase parser is fine-tuned for this task and significantly outperforms its zero-shot model variant and spaCy parser.


- Removing Non-Groundable Phrases. The parser can generate non-specific phrases such as "it", "them" or hard to segment phrases such as "middle". To address this, we use another AI verifier (MLLM) that is fine-tuned to classify such cases and remove them from the rest of the pipeline.


- NP Balancing. We employ heuristics to avoid collecting too many frequent or easy objects. We remove NPs if the data engine has already annotated enough instances, if the SAM 3 has high accuracy when prompted with the NP, and based on a fixed list (e.g. that occur frequently, are harmful). From Phase 3 we rely on AI verifiers to remove easy cases.


- Cleaning NPs. We singularize noun phrases, deduplicate nearly-identical ones, and remove possessives.


- Hard Negative Proposal. A hard negative phrase generator proposes image-level negative phrases, i.e. those that do not exist in the image and are adversarial to SAM 3. Given verified positive NPs (i.e. that exist in the image), negative NPs are proposed and then checked for adversariality.


---


- Proposal. The proposal of hard negatives is done in two ways. The first approach maps every positive NP to a node in the SA-Co ontology, then navigates the ontology graph to find sibling, cousin, or uncle nodes corresponding to different but related concepts. For example, the noun phrase "gray Siamese cat" maps to the node "Siamese cat", which could result in negative candidates like "tabby cat" (sibling), "dog" (uncle), or "Chihuahua" (cousin). The second approach relies on an MLLM (Llama 4), which proposes visually similar negatives for every positive NP.


- Check for Adversariality. Once the negative NPs are proposed, they are filtered to retain only those adversarial to the current SAM 3 version. For each negative NP candidate, predictions from SAM 3 are obtained. If the set of predictions is empty, the candidate is discarded. If the model predicts one or more objects, these predictions are compared to the original segmentation masks of the corresponding positive NP. If the overlap between the negative NP predictions and the positive NP annotations exceeds a certain threshold, the negative NP is retained as a hard negative. This final check is necessary because initial proposals may not be true negatives and instead may be only negatives relative to the existing positive NPs (i.e. the object could still be present somewhere else in the image).


Proposing Masks. We prompt SAM 3 with the set of positive and negative phrases to produce candidate instance and semantic masks for the image. For pseudo-annotating domains with fine-grained concepts that SAM 3 fails on (e.g., Zanclus cornutus), we identify the relevant coarse-grained concept that SAM 3 works well on (e.g., frog), and use this as the prompts to generate masks. We deduplicate masks generated per NP based on a IoU metric. These noisy pseudo-labels undergo further cleaning by both human and AI annotators.


Verification (Human+AI). We train "AI verifiers" to perform the mask verification (MV) and exhaustivity verification (EV) tasks. More specifically, we fine-tune Llama 3.2 Dubey et al. (2024) on human annotated data collected during Phase 1 of the data engine for both tasks.


- Task Formulation. Tab. 21 provides an example data point of the mask verification task: given an (image, phrase, mask) triplet, we render the mask on top of the image as the image prompt, provide the task guidance as text prompt, and use the human annotation (1 out of 5 choices) as output. Each mask's quality is evaluated independently from other masks for the same image-phrase pair. Rendering tricks are used to better visualize small objects, and to avoid color confusion from mask overlay. Tab. 22 provides an example data point of the exhaustivity verification task: given the (image, phrase, masks) triplet, we render the bounding boxes of the masks on top of the image and use this as the image prompt, provide the task guidance as the text prompt, and use the human annotation (1 out of 6 choices) as the output.


- Evaluation. We construct test sets for "AI verifiers" from jobs that were reviewed by multiple human annotators for all SA-Co test sets. We leave one human annotation as human prediction, and use the majority vote of the remaining human annotations as ground truth. This allows us to compare human and AI verifiers' accuracy.


- Training. The training data of each task comes from not only the task itself, but also from the Correction task. For example, each manually added mask is a good data point in the mask verification task. Each exhaustively finished job in the Correction task results in a good data point in exhaustivity verification task. We merge all training data for these two tasks together (over 200M image-text pairs) to pre-train a foundational AI verifier, and then only use high quality human annotated data from the task itself (around 10M scale) to fine-tune two AI verifiers, one for each task.


- Result. Thanks to the simplicity of these two tasks (MCQ tasks on image-text pairs) and the large volume of training data from Phase 1, AI verifiers reach and even surpass human performance on these two tasks, as shown in Tab. 23. We also evaluate the system of SAM 3 and AI verifiers end-to-end on the PCS task, and the system always performs better than the single SAM 3 model, as shown in Tab. 9d.


- Generalization to new domains. We also study the generalization ability of AI verifiers. For a given new domain, the MV AI verifier is typically on par with human verifiers without any domain specific data; the EV AI annotator is typically worse than human in a zero-shot evaluation, but can reach human performance with only thousands of domain specific data points.


As discussed in §A.4, using AI verifiers is effective and allows human annotators to focus on the most challenging data points, i.e. those that have poor mask quality or missing masks. This approach more than


---


<table><tr><td>Input</td><td>Image input for AI verifier (left) and UI for human annotation (right)</td></tr><tr><td>AI Verifier Instructions</td><td>You are an expert annotator of object segmentation masks. For an image and a pre-defined label, you are given a mask and asked to evaluate the quality of the mask. Follow the following rules when rating the mask.
1. Rate the mask as “Accept” when the label accurately describes the masked object and the mask covers the object with good boundaries. We do not need masks to be pixel-perfect for this task. However, the mask should cover all important parts of the object.
2. Rate the mask as “Accept as text” when the mask covers text and the label exactly matches the masked text. The mask should cover all important parts of the text (all specified letters/punctuation/etc.)
3. Rate the mask as “Flag label” when the label corresponds to Race, Ethnicity, Sexual orientation, Religion, Socio-economic status, Medical conditions, Disabilities, Derogatory terms/profanity and Animal phrases for a person.
4. Rate the mask as “Whole image” when the label corresponds to the entire image. Description that refers to the whole image may describe setting (e.g., inside, outside, day, night), type of media (e.g., flier, screenshot, photo), type of image (e.g., close up, an aerial view) and location (e.g., an airport, the woods, a bedroom).
5. Otherwise, rate the mask as “Reject”.
Please give your rating directly without any explanation. Now let’s start. In the given figure, the left half shows a fuchsia box highlighting the region of interest in the original image, and the right half shows a fuchsia mask overlaid on a zoom-in view of that region.
Rate the fuchsia mask for the label “a computer monitor”: (A). Accept as text. (B). Flag label. (C). Reject. (D). Accept. (E). Whole image.</td></tr><tr><td>Mask Verification Result</td><td>(D). Accept.</td></tr></table>


_Table 21 An example data point of Mask Verification (either human or AI verifier). The AI-verifier is given two crops of the image, a zoomed-out view where the object is highlighted via a box and a zoomed-in view where the mask is highlighted. This allows better visualization of small objects, and avoids color confusion from mask overlay. The AI-verifier instructions are a condensed version of the annotation guidelines given to human annotators. Human annotators are also given options to reject the phrase at the image-level as vague. For AI verifier, we use the model output logits of the choice tokens (e.g., A/B/C/D/E) as the logits for the corresponding labels, and soft-max over the logits to get their probabilities._


---


<table><tr><td>Input</td><td>Image input for AI verifier (left) and UI for human annotation (right)</td></tr><tr><td>AI Verifier Instructions</td><td>You are an expert annotator of object detection. For an image and a pre-defined label, you are given some boxes and asked to evaluate if the boxes are exhaustive. Follow the following rules when rating the boxes.
1. Rate the boxes as “Accept” when there are no undetected objects that match the label or the label is vague (i.e., “rent”, “it”) or malformed (heavily misspelled).
2. Rate the boxes as “Reject” when there are undetected objects and at least some undetected instances can be separated.
3. Rate the boxes as “Reject but unseparated” when there are undetected objects but cannot be separated. This is common with groups where it is not possible to tell the undetected instances apart or distinguish them from others.
4. Rate the boxes as “Flag label” when the label corresponds to Race, Ethnicity, Sexual orientation, Religion, Socio-economic status, Medical conditions, Disabilities, Derogatory terms/profanity and Animal phrases for a person.
5. Rate the boxes as “Whole image” when the label corresponds to the entire image. Description that refers to the whole image may describe setting (e.g., inside, outside, day, night), type of media (e.g., flier, screenshot, photo), type of image (e.g., close up, an aerial view) and location (e.g., an airport, the woods, a bedroom).
6. Rate the boxes as “Ungroundable / Vague / Unsure” when the label is an ungroundable or vague concept (e.g., “a corner”, “gap”) or when you are unsure whether or not the phrase is in the image.
Now let’s start. The given figure shows one or multiple red boxes highlighting the region of interest in the original image, evaluate the red box for the label “a computer monitor” and select your answer from the following options: (A). Reject but unseparated. (B). Whole image. (C). Reject. (D). Ungroundable / Vague / Unsure. (E). Accept. (F). Flag label.</td></tr><tr><td>Exhaustivity Verification Result</td><td>(E). Accept.</td></tr></table>


_Table 22 An example data point of Exhaustivity verification (either human or AI verifier). For AI verifier, objects to are highlighted via boxes in the image. If there are no candidate objects, the original image is used and "one or multiple red boxes" is replaced by "zero red box" in the text prompt. For AI verifier, we use the model output logits of the choice tokens (e.g., A/B/C/D/E/F) as the logits for the corresponding labels, and soft-max over the logits to get their probabilities. The presence score from the EV AI verifier is defined as 1 - Prob(Accept|no boxes as input). Prob(Accept|no masks as input) is the probability of Accept (no missing objects) given zero detections as input, which is equivalent to the probability of NO presence._


<table><tr><td rowspan="2"></td><td colspan="2">Attributes</td><td colspan="2">Crowded</td><td colspan="2">Food&amp;Drinks</td><td colspan="2">Sports Equip.</td><td colspan="2">MetaCLIP</td><td colspan="2">SA-1B</td><td colspan="2">Wiki-Common</td><td colspan="2">Average</td></tr><tr><td>MV</td><td>EV</td><td>MV</td><td>EV</td><td>MV</td><td>EV</td><td>MV</td><td>EV</td><td>MV</td><td>EV</td><td>MV</td><td>EV</td><td>MV</td><td>EV</td><td>MV</td><td>EV</td></tr><tr><td>Human</td><td>72.3</td><td>81.2</td><td>72.9</td><td>82.4</td><td>76.8</td><td>76.7</td><td>79.2</td><td>87.3</td><td>72.4</td><td>79.5</td><td>72.3</td><td>73.8</td><td>79</td><td>91.5</td><td>75</td><td>81.8</td></tr><tr><td>AI verifier</td><td>77.1</td><td>82.6</td><td>74.6</td><td>81.3</td><td>79.4</td><td>75.1</td><td>80.1</td><td>84.7</td><td>75.3</td><td>78.8</td><td>75.9</td><td>76.8</td><td>81.3</td><td>88.4</td><td>77.7</td><td>81.1</td></tr></table>


_Table 23 Human/AI verifier accuracy on mask verification (MV) and exhaustivity verification (EV) tasks_


---


doubles the throughput of the SAM 3 data engine. As both SAM 3 and AI verifier models improve, more data can be exhaustively annotated using only SAM 3 and AI verifiers. This leads to increasingly higher throughput and ensures that human annotators only work on SAM 3 failure cases.


Correction. We perform manual correction wherever needed as described in phase 1.


# D.5 Phase 3: Scaling and Domain Expansion


Data Mining. We continue the data mining approaches from Phase 2 and scale to more novel domains. In addition, we target cases that are rare in web datasets and challenging for the model: crowded scenes with high object counts and images with very small objects. To mine such images, we rely on the SA-1B dataset with mask annotations and compute the "crowdedness" metric i.e. calculate IoU between pair of masks and then aggregate it over all pairs of masks. We also use statistics of the number of masks and mask area to identify images with high object counts and very small objects.


Proposing NPs. We continue leveraging the approach from phase 2. We also expand concept coverage to long-tail, fine-grained concepts by extracting NPs from each image's alt-text where available and by mining concepts from the SA-Co ontology.


Proposing Masks. Unchanged from Phase 2.


Verification (Human+AI). We continue to use both human and AI verifiers as described in Phases 1 and 2 respectively, but primarily rely on AI verifiers to increase the data engine throughput.


Correction (Human). We perform manual correction wherever needed, as described in Phase 1. Annotators are asked to correctly mask all occurrences of the given concept in the image.


# D.6 Phase 4: Video Annotation


In Phase 4, we extend the data engine to video. We use the same high-level stages as the image version, but with video-specific implementation details which are described next.


Media Pool. We curate a pool of O(1M) hours of video from SA-V, SA-V internal, YouTube-1B and SA-FARI (wildlife cameras) datasets that covers diverse domains and a range of video quality.


Data Mining. To efficiently utilize human annotation resources, we developed aggressive data mining filters and selected only videos that presented the most challenging object tracking scenarios. The mining pipeline finds challenging single-shot video clips that are 5-30s long. Focusing on single-shot clips largely reduces annotation time and ambiguity originating from attempting to track objects across camera shots in edited videos. The mining pipeline consists of the following steps:


- Scene and Motion Filters. First, we leverage scene boundary detection and VMAF motion scores from FFmpeg (FFmpeg developers) to identify non-static single-shot camera clips from the video pool. To further improve the precision of single-shot clip selection, we also use Shot Boundary Detection from the PySceneDetect (PySceneDetect Developers) library;


- Content Balancing. We use a video-specific ontology to balance content distribution. We build the taxonomy by combining 1) frequent NPs annotated in the image data engine that tend to be associated with higher motion scores, and 2) a taxonomy that emphasizes human activities, animals and transportation. We then generate a set of text queries based on the video ontology and leverage PE Bolya et al. (2025) embeddings to retrieve video candidates for each text query. We propose text queries that elicit grouped objects and crowded scenes, for example "group of dogs" is a text query based on the concept "dog";


- Challenging Track Filter. We use an MLLM (PLM (Cho et al., 2025)) as a judge to select videos with challenging tracking scenarios. This is achieved by performing video-QA on a set of questions regarding the existence of various difficult scenarios, and selecting videos that receive more affirmative responses to these questions;


---


- Targeted Semantic Search. Lastly, we enhance the search for challenging scenarios by performing a video similarity search (using PE embeddings) using known challenging videos identified in human annotation as seeds.


Proposing NPs. We obtain candidate noun phrases for objects in the video.


- Frame-level captioner and parser. We apply the Phase 3 captioner and parser on each video frame, as opposed to video level, to maximize the diversity and volume of candidate noun phrases.


- NP Filtering. To keep only relevant phrases, we implement a series of filters. First, we filter out noun phrases that correspond to the overall scene, such as room, using a fine-tuned Llama 3.1 model. Similarly, we filter out noun phrases that are too ambiguous to be masked, using the previously trained EV AI Verifier which has been trained to classify such cases. Next, we remove noun phrases if they are present in a given list of restricted noun phrases. This list contains 1) phrases that have been annotated as non-maskable in previous annotation rounds, 2) phrases for which we already have a lot of annotations, and 3) phrases that correspond to "background" concepts, as our focus is on challenging moving objects. Next, we optionally filter out phrases that do not belong to certain pre-specified super-categories, such as "animal" or "vehicle" to further focus on moving objects. We determine the super-category of a given noun phrase using a Llama 3.1 model.


- NP Cleaning. The same cleaning is applied as in previous phases.


Proposing Masklets. We use the latest iteration of SAM 3 to generate instance masklets by prompting it with the proposed noun phrases.


- Masklet Generation. Initially, we use SAM 3 at the image level to process frames independently, and then propagate the masks using SAM 2. If masks detected in non-propagated frames are not encompassed by the propagated masklets, they are used as starting points for new SAM 2 masklet propagations. Once SAM 3 video performance surpassed the decoupled system, the pipeline was updated to use SAM 3 alone.


- Masklet Dedduplication. After the masklets are obtained, we deduplicate them based on their IoU.


- Masklet Filtering. We filter out the noun phrases that result in masklets containing the whole scene.


- Filtering Out Easy Cases. We target challenging multi-object scenarios, namely videos that are relatively crowded and contain multiple objects of the same category. The last step of the pseudo-labeling pipeline filters out all noun phrases with fewer than $ \mathrm{N} = 3 $ objects, and videos that contain fewer than $ \mathrm{M} = 2 $ such noun phrases.


# Verification and Correction (Human).


- Verification. Human annotators check if the video is well pre-processed, e.g., no scene cuts, split screen, or explicit content. Then they check if the noun phrase is groundable throughout the video, e.g., there are no comparison or size attributes that might be unclear, and no action attributes which might change across the timeline. Finally, they check that the masklet is challenging to track yet possible to annotate i.e. focus on fast motion and highly occluded objects but which are still identifiable by human annotators and not too blurry to annotate properly.


- Correction. Another annotator reviews the proposed masklets, removing those that are incorrect (improving precision), and using online SAM 2 in the loop to correct those that can be improved. Next, they check for any missing masklets, and use SAM 2 to add them if needed (improving recall). This annotation task results in two types of data: fully exhaustive tracking data where every object that matches the noun phrase is annotated, or partially exhaustive tracking data, where some masklets might be missing because they are impossible to annotate (e.g., inseparable background objects that match the noun phrase).


- Exhaustivity Confirmation. To ensure data quality, a final round of exhaustivity checking is performed. If there are any remaining missing masklets, they are added as necessary.


Sampling Frame Annotations. To increase the diversity and volume of the annotated video data, we also sample video frames and annotate them using the image data engine (Phase 3), where they are treated the same way as other images. The sampling follows two separate strategies. The first one is just random sampling of a


---


frame within a video. This guarantees we cover the distribution of frames. The second strategy consists of first running the video data engine pipeline, and using the results to determine frames that contain many objects.


# E SA-Co Dataset and Metric Details


# E.1 SA-Co Training Data


SAM 3 training data includes images and videos from many diverse sources, including existing datasets with box or mask annotations. The training data consists of three image datasets and one video dataset. Fig. 12 visualizes statistics on these subsets in comparison with existing open-source image and video detection and instance segmentation datasets as well as the distribution of top-level SA-Co ontology categories on image datasets. More detailed statistics for each subset and comparison with open-source datasets are shown in Tab. 24 and Tab. 25. The original dataset sources by subset are listed in Tab. 26.


SA-Co/HQ: High quality. This image dataset is generated by the data engine in Phases 1-3 with high quality annotations verified either by human annotators or by AI verifiers that have accuracy on par with humans.


SA-Co/SYN: Synthetic. We generate this synthetic dataset via the data engine in Phase 3, relying only on AI annotators. We use MetaCLIP images as the media pool and extract NPs from two sources: 1) alt-text captions associated with the images, 2) captions generated by Llama4. We prompt SAM 3 using the extracted NPs to generate mask proposals. The image-NP-mask proposals are then verified by MV and EV AI verifiers resulting in high-quality synthetic data. We also generate hard negatives proposals (§D.4) and verify them using the EV AI verifier resulting in exhaustive image-level negatives. This scalable system enabled large-scale synthetic data generation, resulting in 39M images, 1.7B image-NPs and 1.4B masks.


SA-Co/EXT: External. This dataset includes eighteen external datasets with existing instance mask or bounding boxes annotations. For datasets with only bounding boxes, we generate instance masks with SAM 2. We further enrich these external datasets by mapping the original label to SA-Co ontology and propose additional negative labels using Wikidata hierarchy.


SA-Co/VIDEO: Video. The video dataset is collected via the data engine in phase 4 with high quality annotations. All the data in SA-Co/VIDEO is verified by human annotators.


![图片 page34-8](images/page034_img01.png)


_(a)_


![图片 page34-10](images/page034_img02.png)


_(b)_


![图片 page34-12](images/page034_img03.png)


_(c)_


_Figure 12 (a) SAM 3 image training data statistics and comparison with existing open-source image detection and instance segmentation datasets. Bubble size denotes total number of NP-mask/bbox pairs. Bolded datasets are annotated with masks, others are bboxes only. Datasets with * are exhaustively annotated, datasets with † are partially exhaustively annotated. (b) SAM 3 video training data statistics and comparison with existing open-source video instance segmentation datasets. Bubble size denotes total number of NP-masklet pairs. Datasets with * are exhaustively annotated. (c) Instance masks distribution among SA-Co ontology top-level categories in SA-Co/HQ and SA-Co/EXT. SA-Co/EXT incorporates several medical and microscopy image datasets, we categorize them under medical imaging in addition to categories in Tab. 20._


---


<table><tr><td>Dataset</td><td># NPs</td><td># Images</td><td># Image-NP</td><td>% Negatives</td><td># NP-bbox</td><td># NP-mask</td><td># masks per pair</td></tr><tr><td>Flickr 30k</td><td>86.4K</td><td>30.1K</td><td>193.0K</td><td>-</td><td>312.2K</td><td>-</td><td>-</td></tr><tr><td>LVIS*</td><td>1.2K</td><td>120.0K</td><td>1.6M</td><td>72.7%</td><td>1.5M</td><td>1.5M</td><td>3.51</td></tr><tr><td>V3Det*</td><td>13.2K</td><td>213.2K</td><td>737.7K</td><td>-</td><td>1.6M</td><td>-</td><td>-</td></tr><tr><td>Visual Genome</td><td>542.6K</td><td>108.1K</td><td>4.3M</td><td>-</td><td>6.3M</td><td>-</td><td>-</td></tr><tr><td>Open Images</td><td>600</td><td>1.7M</td><td>4.1M</td><td>-</td><td>13.3M</td><td>2.7M</td><td>2.79</td></tr><tr><td>Object365*</td><td>365</td><td>1.7M</td><td>10.1M</td><td>-</td><td>22.9M</td><td>-</td><td>-</td></tr><tr><td>SA-Co/HQ*</td><td>4.0M</td><td>5.2M</td><td>146.1M</td><td>88.5%</td><td>52.3M</td><td>52.3M</td><td>3.10</td></tr><tr><td>SA-Co/EXT†</td><td>497.4K</td><td>9.3M</td><td>136.6M</td><td>71.8%</td><td>70.5M</td><td>70.5M</td><td>1.83</td></tr><tr><td>SA-Co/SYN*</td><td>38.0M</td><td>39.4M</td><td>1.7B</td><td>74.0%</td><td>1.4B</td><td>1.4B</td><td>3.17</td></tr></table>


_Table 24 Detailed statistics for image training datasets and comparison with existing open-source image detection and instance segmentation datasets. Datasets with * are exhaustively annotated, datasets with † are partially exhaustively annotated. % Negatives denotes percentage of Image-NPs that are negatives._


<table><tr><td>Dataset</td><td># NPs</td><td># Videos</td><td># Video-NP</td><td>% Negatives</td><td># NP-masklet</td><td># masklets per pair</td></tr><tr><td>OVIS*</td><td>25</td><td>607</td><td>886</td><td>-</td><td>3.6K</td><td>4.04</td></tr><tr><td>YTVIS 2021*</td><td>40</td><td>3.0K</td><td>3.9K</td><td>-</td><td>6.3K</td><td>1.61</td></tr><tr><td>BURST</td><td>482</td><td>2.9K</td><td>6.9K</td><td>-</td><td>16.0K</td><td>2.33</td></tr><tr><td>LVVIS</td><td>1.2K</td><td>3.9K</td><td>13.8K</td><td>-</td><td>19.7K</td><td>1.40</td></tr><tr><td>SA-Co/VIDEO*</td><td>24.8K</td><td>52.5K</td><td>134.3K</td><td>26.7%</td><td>467.1K</td><td>4.75</td></tr></table>


_Table 25 Detailed statistics for video training dataset and comparison with existing open-source video instance segmentation datasets. Datasets with * are exhaustively annotated. % Negatives denotes percentage of Video-NPs that are negatives._


# E.2 SA-Co Evaluation Benchmark


We create the Segment Anything with Concepts (SA-Co) Benchmark for evaluating promptable concept segmentation (PCS) in images and videos. Our benchmark contains images and videos paired with text labels, each annotated exhaustively with masks on all object instances that match the label. The dataset is federated, meaning that not all labels are annotated for all images, but only a handful of positive and negative labels are verified as ground-truth per image. We add a large volume of challenging hard negative label annotations to test models' ability to handle large, open vocabularies. In particular, the SA-Co/Gold benchmark has $\sim 50\times$ more unique phrases compared to existing exhaustively annotated mask dataset LVIS-test. The SA-Co benchmark covers a diverse array of sub-domains including common objects, fine-grained concepts, food, art, robotics, etc. See Tab. 28 for detailed benchmark statistics and Tab. 27 for the list of sub-domains and their original sources.


In particular, the SA-Co/Gold benchmark has seven sub-domains as shown in Fig. 14 and Tab. 27, created to test different aspects of the concept and image distributions:


<table><tr><td>SA-Co/HQ</td><td>SA-Co/SYN</td><td>SA-Co/EXT</td><td>SA-Co/VIDEO</td><td>SA-Co/VIDEO-EXT</td></tr><tr><td>MetaCLIP (Xu et al., 2024b,a)</td><td>MetaCLIP</td><td>Objects365 (Shao et al., 2019)</td><td>SA-V (Ravi et al., 2024)</td><td>DAVIS2017 (Pont-Tuset et al., 2017b)</td></tr><tr><td>SA-1B (Kirillov et al., 2023)</td><td></td><td>OpenImages (Kuznetsova et al., 2020)</td><td>SA-V internal (Ravi et al., 2024)</td><td>MOSEv2 (Ding et al., 2025)</td></tr><tr><td>Armbench (Mitash et al., 2023)</td><td></td><td>ImageNet (Russakovsky et al., 2015)</td><td>YT-Temporal-1B (Zellers et al., 2022)</td><td>YTVOS2019 (Xu et al., 2018)</td></tr><tr><td>National Gallery of Art (National Gallery of Art)</td><td></td><td>VisualGenome (Krishna et al., 2017)</td><td>SA-FARI (SA-FARI Dataset)</td><td></td></tr><tr><td>Ego4d (Grauman et al., 2022)</td><td></td><td>Sapiens Body-Parts (Khirodkar et al., 2024)</td><td></td><td></td></tr><tr><td>MyFoodRepo-273 (Mohanty et al., 2021)</td><td></td><td>EDEN (Le et al., 2021)</td><td></td><td></td></tr><tr><td>GeoDE (Ramaswamy et al., 2023)</td><td></td><td>Fashionpedia (Jia et al., 2020)</td><td></td><td></td></tr><tr><td>DROID (Khazatsky et al., 2024)</td><td></td><td>Fathomnet (Katija et al., 2021)</td><td></td><td></td></tr><tr><td>BDD100k (Yu et al., 2020)</td><td></td><td>iNaturalist-2017 (Horn et al., 2017)</td><td></td><td></td></tr><tr><td>SA-V (Ravi et al., 2024)</td><td></td><td>BDD100k (Yu et al., 2020)</td><td></td><td></td></tr><tr><td>SA-V internal (Ravi et al., 2024)</td><td></td><td>Livecell (Edlund et al., 2021)</td><td></td><td></td></tr><tr><td>YT-Temporal-1B (Zellers et al., 2022)</td><td></td><td>PanNuke (Gamper et al., 2019, 2020)</td><td></td><td></td></tr><tr><td></td><td></td><td>MedSAM2 (Ma et al., 2025)</td><td></td><td></td></tr><tr><td></td><td></td><td>SNOW (Ding et al., 2023)</td><td></td><td></td></tr><tr><td></td><td></td><td>Visdrone (Zhu et al., 2021)</td><td></td><td></td></tr><tr><td></td><td></td><td>WCS Camera Traps (LILA BC)</td><td></td><td></td></tr><tr><td></td><td></td><td>HierText (Long et al., 2023, 2022)</td><td></td><td></td></tr><tr><td></td><td></td><td>FSC-147 (Ranjan et al., 2021)</td><td></td><td></td></tr></table>


_Table 26 Media pool used to construct each SA-Co train subset. See Figs. 13 and 15 to 18 for examples of each domain and annotations._


---


MetaCLIP MetaCLIP images (web-scraped) annotated with captioner-proposed noun phrases.


- SA-1B SA-1B images (stock photos, more objects per image than MetaCLIP) annotated with captioner-proposed noun phrases.


- Attributes MetaCLIP images annotated with attribute phrases. To better test attribute understanding, we also annotate phrases with swapped nouns, e.g., “pink rose” $\rightarrow$ “pink flamingo”, and adjectives, e.g., “pink rose” $\rightarrow$ “red rose.”


- **Crowded Scenes** SA-1B images filtered to select very crowded scenes, annotated with noun phrases proposed by MLLM.


- Wiki-Common MetaCLIP images annotated with labels corresponding to 1K nodes from the SA-Co ontology judged to be common by an LLM. These concepts are meant to expand the vocabulary beyond frequent terms like "car", but still be recognizable to non-experts, e.g., "Jeep", "bunk bed", "ballot box."


- Wiki-Food&Drink MetaCLIP images annotated with labels corresponding to nodes from the Food&Drink branch of the SA-Co ontology. Many are very fine-grained concepts like “kefir”, “pastille”.


- Wiki-Sports Equipment MetaCLIP images annotated with labels corresponding to nodes from the Sports Equipment branch of the SA-Co ontology, with many fine-grained concepts like "kettlebell."


All of the above sub-domains also have a high number of hard negative annotations, see Tab. 28.


<table><tr><td>SA-Co/Gold</td><td>SA-Co/Silver</td><td>SA-Co/Bronze</td><td>SA-Co/Bio</td><td>SA-Co/VEval</td></tr><tr><td>MetaCLIP</td><td>BDD100k</td><td>BDD100k</td><td>Livecell</td><td>SA-V</td></tr><tr><td>SA-1B</td><td>DROID</td><td>EDEN</td><td>PanNuke</td><td>YT-Temporal-1B</td></tr><tr><td>Attributes (MetaCLIP)</td><td>Ego4D</td><td>Fashionpedia</td><td>MedSAM2</td><td>SmartGlasses</td></tr><tr><td>Crowded Scenes (SA-1B)</td><td>MyFoodRepo-273</td><td>Visdrone</td><td>SNOW</td><td></td></tr><tr><td>Wiki-Common (MetaCLIP)</td><td>GeoDE</td><td>WCS Camera Traps</td><td></td><td></td></tr><tr><td>Wiki-Food&amp;Drink (MetaCLIP)</td><td>iNaturalist-2017</td><td></td><td></td><td></td></tr><tr><td>Wiki-Sports Equipment (MetaCLIP)</td><td>National Gallery of Art SA-V</td><td></td><td></td><td></td></tr><tr><td></td><td>YT-Temporal-1B</td><td></td><td></td><td></td></tr><tr><td></td><td>Fathomnet</td><td></td><td></td><td></td></tr></table>


_Table 27 Domains in each SA-Co evaluation benchmark subset. See Figs. 13, 14 and 16 to 18 for examples of each domain and annotations._


<table><tr><td>Dataset</td><td># NPs</td><td># Images</td><td># Image-NP</td><td>% Negatives</td><td># NP-masks</td><td>% 0-shot NPs</td></tr><tr><td>LVIS test</td><td>1.2K</td><td>19.8K</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>COCO test2017</td><td>80</td><td>40.7K</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>ODinW-35 test</td><td>290</td><td>15.6K</td><td>26.1K</td><td>-</td><td>131.1K</td><td>-</td></tr><tr><td>SA-Co/Gold</td><td>51.8K</td><td>15.8K</td><td>168.9K</td><td>84.4%</td><td>126.9K</td><td>6.98%</td></tr><tr><td>SA-Co/Silver</td><td>54.6K</td><td>66.1K</td><td>1.8M</td><td>94.0%</td><td>219.8K</td><td>8.00%</td></tr><tr><td>SA-Co/Bronze</td><td>105.3K</td><td>32.5K</td><td>1.0M</td><td>84.9%</td><td>261.5K</td><td>57.25%</td></tr><tr><td>SA-Co/Bio</td><td>166</td><td>5.4K</td><td>35.9K</td><td>71.8%</td><td>264.6K</td><td>-</td></tr><tr><td colspan="7">(a)</td></tr><tr><td>Dataset</td><td># NPs</td><td># Videos</td><td># Video-NP</td><td>% Negatives</td><td># NP-masklets</td><td>% 0-shot NPs</td></tr><tr><td>LVVIS test</td><td>1.2K</td><td>908</td><td>-</td><td>-</td><td>5.7K</td><td>-</td></tr><tr><td>BURST test</td><td>482</td><td>1.4K</td><td>3.4K</td><td>-</td><td>8.0K</td><td>-</td></tr><tr><td>SA-Co/VEval</td><td>5.2K</td><td>1.7K</td><td>10.3K</td><td>75.4%</td><td>11.2K</td><td>6.37%</td></tr></table>


_(b)_


_Table 28 (a) Summary statistics of SA-Co image evaluation benchmark by subsets and comparison with existing image instance segmentation benchmarks. (b) Summary statistics of SA-Co/VEval benchmark and comparison video instance segmentation benchmarks. % Negatives denotes percentage of Image-NPs or Video-NPs that are negative. Percentages of zero-shot NPs in each subset are reported. A zero-shot NP is defined as a phrase that has not been seen in the combined set of SA-Co/HQ, SA-Co/EXT and SA-Co/VIDEO._


---


# E.3 Metrics


We introduce the classification-gated $ F1 $ ($ \mathrm{cgF}_1 $) to evaluate the PCS task on images. The traditional AP (Average Precision) metric designed for closed-vocabulary detection tasks (e.g., COCO), breaks down when applied to open-vocabulary detection with very large label spaces. While averaging AP over 80 classes is feasible, with tens of thousands most appear just once in the test set and the average is dominated by noise. Computing full precision-recall curves for all labels is also computationally infeasible and unnecessary for practical use cases. AP also does not account for the model calibration, which means that high-scoring models can be difficult to use in practice. F1 at a fixed confidence threshold presents a good alternative, however it is sensitive to high ratios of negative annotations: no extra credit is given for correctly predicting nothing for a negative, but the score is lowered by predicting false positives.


To remedy these issues we design new metrics for the PCS task. Given datapoints consisting of predicted and ground truth (media, phrase, masks) triplets we compute the following metrics to measure localization and classification separately:


- Localization. We measure this only on positive datapoints with at least one ground-truth mask. For one sample, assume we have $N$ predicted masks $m_{1}, \dots, m_{N}$ and $M$ ground-truth masks $\hat{m}_{1}, \dots, \hat{m}_{M}$. We compute the IoU matrix $\mathrm{iou}_{i,j} = \mathrm{iou}(m_{i}, \hat{m}_{j})$, then deduce the optimal bipartite matching $\hat{\sigma} = \operatorname{argmax}_{\sigma} \sum_{i} \mathrm{iou}_{i,\sigma(i)}$. We fix an IoU threshold $\tau$, then for every prediction $i$, if it is matched and $\mathrm{iou}_{i,\sigma(i)} \geq \tau$, then it is counted as TP (true positive), otherwise FP (false positive). Unmatched ground truths are counted as FN (false negative). We compute $\mathrm{F}_1^\tau = \frac{2\mathrm{TP}^\tau}{2\mathrm{TP}^\tau + \mathrm{FP}^\tau + \mathrm{FN}^\tau}$ for each datapoint, known as the local F1 score. We accumulate the counts of TP, FP and FN over all data points with at least one groundtruth mask, and compute "positive micro F1" score $\mathrm{pmF}_1^\tau$. We compute $\mathrm{pmF}_1^\tau$ for all $\tau \in [0.5, 0.95]$ with increments of 0.05, then average to obtain the final $\mathrm{pmF}_1$:


$$
\mathrm {p m F} _ {1} ^ {\tau} = \frac {2 \mathrm {T P} _ {t o t a l} ^ {\tau}}{2 \mathrm {T P} _ {t o t a l} ^ {\tau} + \mathrm {F P} _ {t o t a l} ^ {\tau} + \mathrm {F N} _ {t o t a l} ^ {\tau}}, \quad p m F _ {1} = \frac {1}{1 0} \sum_ {\tau \in \mathrm {n p . l i n s p a c e (0 . 5 , 0 . 9 5 , 1 0)}} \mathrm {p m F} _ {1} ^ {\tau}. \qquad (1)
$$


We also compute the average of the local F1 scores over all data points with at least one groundtruth mask, and obtain the "positive macro F1" score. We report both the positive micro and macro F1 scores in our score chart, and choose the positive micro score $\mathrm{pmF}_1$ as the main metric for localization.


- Classification. This metric between $[-1, 1]$ computes the ability of the model to predict one or several masks, if and only if the datapoint is positive. This can be seen as a binary prediction task at the image level ("is the object present or not?"), and crucially, in this metric we do not care about the quality of the predicted masks. If the datapoint is positive, and if the model has predicted any mask (with confidence greater than 0.5), then it is an IL_TP (image level TP), otherwise an IL_FN. If the datapoint is negative, and if the model has predicted any mask, then it is an IL_BP, otherwise an IL_TN. We summarize this confusion matrix into a single metric, and measure potential imbalances with the Matthews Correlation Coefficient (MCC) as:


$$
\mathrm {I L \_ M C C} = \frac {\mathrm {I L \_ T P} \cdot \mathrm {I L \_ T N} - \mathrm {I L \_ F P} \cdot \mathrm {I L \_ F N}}{\sqrt {(\mathrm {I L \_ T P} + \mathrm {I L \_ F P}) \cdot (\mathrm {I L \_ T P} + \mathrm {I L \_ F N}) \cdot (\mathrm {I L \_ T N} + \mathrm {I L \_ F P}) \cdot (\mathrm {I L \_ T N} + \mathrm {I L \_ F N})}}. (2)
$$


As our main metric, we combine these two metrics to compute $\mathrm{cgF}_1$ ("classification-gated F1"), defined as


$$
\mathrm {c g F} _ {1} = 1 0 0 \cdot \mathrm {p m F} _ {1} \cdot \mathrm {I L \_ M C C}. \tag {3}
$$


The PCS task is quite ambiguous in many cases, and to alleviate this issue our SA-Co/Gold subset contains three independent ground-truth annotations for each datapoint. To adapt our metric, we use an oracle setting, where we compare the model's predictions to each ground-truth for each datapoint, and select the one that yields the best local F1 score.


# E.4 Human Performance on SA-Co


As described in §2, the PCS task is intrinsically ambiguous. Given an image-NP or video-NP pair, even trained annotators can have different interpretations that are all valid. When the phrase is vague, annotators


---


<table><tr><td rowspan="2"></td><td colspan="3">SA-Co/Gold</td><td colspan="8">SA-Co/VEval</td></tr><tr><td colspan="3">3-Reviewed Subset</td><td colspan="4">YT-Temporal-1B test</td><td colspan="4">SmartGlasses test</td></tr><tr><td>Evaluation Protocol</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>cgF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td></tr><tr><td>Oracle</td><td>76.2</td><td>0.99</td><td>77.0</td><td>85.1</td><td>88.4</td><td>84.7</td><td>92.6</td><td>73.5</td><td>83.1</td><td>75.4</td><td>92.0</td></tr><tr><td>Random Pair</td><td>55.5</td><td>0.87</td><td>63.4</td><td>75.9</td><td>82.0</td><td>73.8</td><td>91.3</td><td>53.4</td><td>70.2</td><td>54.9</td><td>90.2</td></tr></table>


_Table 29 Human instance segmentation performance comparison between the Oracle and Random Pair protocols on SA-Co benchmark. The comparison is shown on subsets of the benchmark with three human annotations._


can even disagree on the presence of the NP. Hence when evaluating on the SA-Co benchmark, disagreement with ground truth does not necessarily mean the prediction is wrong. To this end, it is important to study the human-level performance (i.e. the agreement among skilled annotators) on the PCS task to facilitate interpreting model performance.


Human Performance on SA-Co/Gold. On the image benchmark, we provide three sets of annotations by different annotators. Fig. 14 shows examples of the three independent annotations per (phrase, image) pair for each domain in the benchmark. These annotations are done from scratch, meaning that the annotators create masks (using SAM 1) without seeing any SAM 3 model interpretations. We define the "oracle" metric as follows to measure the upper bound of human performance. For each image-NP, the best pair (out of all three pairs of annotations) is selected by maximizing the local F1 score or minimizing the sum of false negatives (FN) and false positives (FP) when there is a tie in local F1 scores. We then report the $\mathrm{cgF}_1$ metric based on these selected best pairs using one annotation as ground truth and the other as prediction. To make the model performance comparable, the "oracle" model performance is calculated by comparing model predictions to all three annotations and selecting the best pairs.


Alternative to the "Oracle" protocol, human performance can also be measured on randomly selected pairs. Specifically, we adopt the following protocol to compute "Random Pair" human performance on SA-Co benchmark with three sets of annotations: 1) randomly choosing a pair of annotations for each image/video-NP, then aggregate over all image/video-NPs to get the metric values, 2) repeating the process a thousand times and reporting the 0.5 quantile for each metric. As shown in Tab. 29, there is a noticeable gap between Oracle and Random Pair performance on both image and video benchmarks, suggesting that the PCS task is inherently ambiguous.


The image benchmark has a large portion of hard negatives. These phrases go through human verification, but as it is costly to collect three sets of human annotations on the entire dataset due to the large volume, the negative noun phrases only have one ground-truth label. The human performance on these phrases is estimated by collecting additional human annotations on a subsample of phrases and comparing them with the initial annotation (i.e., the ground truth). Specifically, we collect additional human annotations on about one thousand image-NPs for each domain in SA-Co/Gold. Since the ground truths are all negatives, these phrases only contribute to the IL_MCC metric. We compute counts of IL_TN and IL_FP on these samples, and then extrapolate these results to estimate the corresponding counts for the entire set of hard negatives. These estimated counts are then combined with image-level counts from the rest of the benchmark where NPs have three annotations to get the final IL_MCC.


Typically, our annotation protocol allows annotators to mark NPs as ambiguous if they are unsure. In this additional human review for the hard negatives, we remove the unsure option and prompt them to make a choice between positive and negative, thus reduce uncertainty and potential bias that could arise from ambiguous data.


Human Performance on SA-Co/VEval. Annotating videos is much more expensive than static images, so we collect only one set of annotations per NP on the video benchmark. To guarantee annotation quality, these ground-truth annotations undergo multiple rounds of human refinement. To measure human performance in a way that is directly comparable to model evaluation in video PCS, we collect one additional from-scratch human annotation for every NP in the test set across all sub-domains. Human performance on video PCS task is then reported by comparing this additional annotation to the ground truth, using the same metrics as for model evaluation ($\mathrm{cgF}_1$ and pHOTA).


---


Additionally, to study the gap between the Random Pair and the Oracle protocols, we collect two further human annotations (for a total of three) on the YT-Temporal-1B and SmartGlasses test splits of the SA-Co/VEval dataset. This allows us to verify that the gap observed in the image domain also exists in the video domain (see Tab. 29).


# E.5 Additional Dataset Examples


Figs. 13 and 15 to 18 show examples of each visual domain in our image and video datasets. Fig. 14 illustrates the domains in our SA-Co/Gold evaluation benchmark and the three independent annotations per sample. Fig. 19 show an example image from our synthetic data set SA-Co/SYN, with its positive noun phrases in the figure and negative noun phrases in the caption.


![图片 page39-3](images/page039_img01.png)


_Farrea Crinoidea Farrea occa Asbestopluma_


![图片 page39-5](images/page039_img02.png)


![图片 page39-6](images/page039_img03.png)


_Calidris alpina_


![图片 page39-8](images/page039_img04.png)


![图片 page39-9](images/page039_img05.png)


_Cell_


![图片 page39-11](images/page039_img06.png)


![图片 page39-12](images/page039_img07.png)


_Cell_


_Livecell (Edlund et al., 2021)_


![图片 page39-15](images/page039_img08.png)


![图片 page39-16](images/page039_img09.png)


_Neoplastic Cell Connective Cell Inflammatory Cell_


![图片 page39-18](images/page039_img10.png)


_Figure 13 Per-domain examples in SA-Co/EXT. Shown with annotated phrases and masks overlaid._


---


![图片 page40-0](images/page040_img01.png)


_Figure 14 Annotations in our SA-Co/Gold benchmark. Each row shows an example (image, noun phrase) pair from one of the seven domains, with masks from three independent annotators overlayed. No mask means the annotator decided that the phrase is not in the image. Dashed borders indicate special group masks that cover more than a single instance, used when separating into instances is deemed difficult. Annotators sometimes disagree on precise mask borders, the number of instances (e.g., house, broccoli, swimfins) or on whether the phrase applies (Jeep). We use the 3 GT annotations to measure the human performance on the task, to serve as an upper bound for model performance._


---


![图片 page41-0](images/page041_img01.png)


![图片 page41-1](images/page041_img02.png)


Waistband


Halloween Pumpkin ...


![图片 page41-4](images/page041_img03.png)


![图片 page41-5](images/page041_img04.png)


trundle bed


![图片 page41-7](images/page041_img05.png)


![图片 page41-8](images/page041_img06.png)


anana bread


![图片 page41-10](images/page041_img07.png)


![图片 page41-11](images/page041_img08.png)


pool table


![图片 page41-13](images/page041_img09.png)


the red body


tape measure


a red pencil


the white band


black and yellow pla...


MetaCLIP alt-text


(Xu et al., 2024b)


MetaCLIP Common WikiNodes


MetaCLIP Food and Drink


MetaCLIP Sports Equipment


SA-1B


(Kirillov et al., 2023)


![图片 page41-27](images/page041_img10.png)


sign


hum


#


wind


cour


the I


a pill


a shi


the p


the animal


the carving


the backpack


light fixture


vending machine


![图片 page41-42](images/page041_img11.png)


table


an egg


plastic


a tomato


ingredient


a halloumi


salad green


the ceramic


a yellow bowl


a tomato, raw


a pumpkin seeds


a cherry tomato


the halloumi cheese


variety of ingredients


a mix of green lettuce


light-colored woode...


![图片 page41-59](images/page041_img12.png)


atwig


black spot


black rabbit


a grey rabbit


the white rabbit


the rusty metal trough


a white rabbit with a …


![图片 page41-67](images/page041_img13.png)


abush


a drawing


a gentleman


the left hand


a formal attire


the man's outfit


the white cravat


![图片 page41-76](images/page041_img14.png)


a road


a person a shadow


black pant


the ground


the pigeon


a blue tarp


a dirt road


a dirt ground


SA-1B Crowded Scenes


Food Recognition


2022 Challenge


(Mohanty et al., 2021)


SA-V internal


(Ravi et al., 2024)


National Gallery of Art


(National Gallery of Art)


SA-V


(Ravi et al., 2024)


![图片 page41-96](images/page041_img15.png)


a bush


a garden


black light


a brick pathway


a red brick house


dense, green shrub


![图片 page41-103](images/page041_img16.png)


tire


a building


the headlight


![图片 page41-107](images/page041_img17.png)


wire


a clip


machine


the screw


the package


![图片 page41-113](images/page041_img18.png)


the window


the grey wall


the robotic arm


the white paper bag


![图片 page41-118](images/page041_img19.png)


![图片 page41-119](images/page041_img20.png)


stove


Ego4d


(Grauman et al., 2022)


BDD100K


(Yu et al., 2020)


Armbench


(Mitash et al., 2023)


DROID


(Khazatsky et al., 2024)


GeoDE


(Ramaswamy et al., 2023)


_Figure 15 Per-domain examples in SA-Co/HQ. Shown with annotated phrases and masks overlaid._


---


![图片 page42-0](images/page042_img01.png)


_Figure 16 Example annotations from the SA-V media in the SA-Co/VIDEO and SA-Co/VEval datasets._


---


![图片 page43-0](images/page043_img01.png)


![图片 page43-1](images/page043_img02.png)


_a silver lid_


![图片 page43-3](images/page043_img03.png)


_the black knob_


![图片 page43-5](images/page043_img04.png)


_black burner_


![图片 page43-7](images/page043_img05.png)


_white piece of paper_


![图片 page43-9](images/page043_img06.png)


![图片 page43-10](images/page043_img07.png)


_a blue-handled knife_


![图片 page43-12](images/page043_img08.png)


![图片 page43-13](images/page043_img09.png)


![图片 page43-14](images/page043_img10.png)


_a person_


![图片 page43-16](images/page043_img11.png)


![图片 page43-17](images/page043_img12.png)


![图片 page43-19](images/page043_img13.png)


![图片 page43-20](images/page043_img14.png)


![图片 page43-22](images/page043_img15.png)


_Figure 17 Example annotations from the SmartGlasses media in the SA-Co/VEval dataset._


![图片 page43-24](images/page043_img16.png)


![图片 page43-25](images/page043_img17.png)


_white-nosed coati_


![图片 page43-27](images/page043_img18.png)


![图片 page43-28](images/page043_img19.png)


![图片 page43-29](images/page043_img20.png)


![图片 page43-30](images/page043_img21.png)


![图片 page43-31](images/page043_img22.png)


![图片 page43-32](images/page043_img23.png)


![图片 page43-33](images/page043_img24.png)


![图片 page43-34](images/page043_img25.png)


![图片 page43-35](images/page043_img26.png)


![图片 page43-36](images/page043_img27.png)


![图片 page43-37](images/page043_img28.png)


![图片 page43-38](images/page043_img29.png)


![图片 page43-39](images/page043_img30.png)


![图片 page43-40](images/page043_img31.png)


![图片 page43-41](images/page043_img32.png)


![图片 page43-42](images/page043_img33.png)


![图片 page43-43](images/page043_img34.png)


![图片 page43-44](images/page043_img35.png)


![图片 page43-45](images/page043_img36.png)


![图片 page43-46](images/page043_img37.png)


![图片 page43-47](images/page043_img38.png)


![图片 page43-48](images/page043_img39.png)


![图片 page43-49](images/page043_img40.png)


![图片 page43-50](images/page043_img41.png)


![图片 page43-51](images/page043_img42.png)


![图片 page43-52](images/page043_img43.png)


![图片 page43-53](images/page043_img44.png)


![图片 page43-54](images/page043_img45.png)


![图片 page43-55](images/page043_img46.png)


![图片 page43-56](images/page043_img47.png)


![图片 page43-57](images/page043_img48.png)


![图片 page43-58](images/page043_img49.png)


![图片 page43-59](images/page043_img50.png)


![图片 page43-60](images/page043_img51.png)


_squirrel monkey_


![图片 page43-62](images/page043_img52.png)


![图片 page43-63](images/page043_img53.png)


![图片 page43-64](images/page043_img54.png)


![图片 page43-65](images/page043_img55.png)


![图片 page43-66](images/page043_img56.png)


![图片 page43-67](images/page043_img57.png)


![图片 page43-68](images/page043_img58.png)


![图片 page43-69](images/page043_img59.png)


![图片 page43-70](images/page043_img60.png)


![图片 page43-71](images/page043_img61.png)


![图片 page43-72](images/page043_img62.png)


![图片 page43-73](images/page043_img63.png)


![图片 page43-74](images/page043_img64.png)


![图片 page43-75](images/page043_img65.png)


_Figure 18 Example annotations from the SA-FARI dataset._


---


![图片 page44-0](images/page044_img01.png)


![图片 page44-1](images/page044_img02.png)


![图片 page44-2](images/page044_img03.png)


![图片 page44-3](images/page044_img04.png)


![图片 page44-4](images/page044_img05.png)


![图片 page44-5](images/page044_img06.png)


![图片 page44-6](images/page044_img07.png)


![图片 page44-7](images/page044_img08.png)


_Figure 19 Example annotations on one image from the SA-Co/SYN dataset. There are 19 positive noun phrases for this image, which we visualize in 4 groups for better visualization quality. AI verifier assigned an exhaustivity label for each noun phrase: noun phrases in green box means the masks for that noun phrase is exhaustive, noun phrases in pink box means not exhaustive. There are 20 negative noun phrases that are verified and confirmed by AI verifier for this image: a black and white post, a blue and yellow post, a large red and white truck, an orange and black post, large orange and white boat, the green and red post, the large orange and white motorcycle, the orange and white bollard, the purple and yellow post, yellow sash, the Malaysian Air Boeing 737-800, display vehicle, Zamboni ice resurfacer, boot, the small folded paper package, long yellow vehicle, the small gift-wrapped package, small trailer, bunker gear, two-seat model._


---


# F Additional Experiments and Details


# F.1 PCS with NPs on Images


This section describes the experiments in Tab. 1 in detail. We compare to OWLv2 (Minderer et al., 2024), GroundingDino (Liu et al., 2024a) and LLMDet (Fu et al., 2025). Since they produce only bounding boxes, we convert them to masks using SAM 1 to evaluate segmentation. We also compare to APE (Shen et al., 2024) and DINO-X (Ren et al., 2025), two state-of-the-art segmentation models, and finally Gemini 2.5 Flash (Comanici et al., 2025), a generalist LLM.


We report performance on LVIS (Gupta et al., 2019), COCO (Lin et al., 2014), COCO-O (Mao et al., 2023), Cityscapes (Cordts et al., 2016), ADE (Zhou et al., 2019), and Pascal Context (Mottaghi et al., 2014), reporting their official metrics. For LVIS, we report AP-fixed (Dave et al., 2022). On our new SA-Co benchmark, we report the average across every split. We report $\mathrm{cgF}_1$, except for SA-Co/Bio where we report $\mathrm{pmF}_1$ (this split does not have negatives, so only localization is meaningful). On SA-Co/Gold we have three ground-truth annotations per datapoint, so we report the oracle metric and estimated human performance (human performance measurement detailed in §E.4). To evaluate semantic segmentation using SAM 3, we predict instance masks for each semantic category and filtering the predictions using the presence scores, mask scores, and mask areas to create the semantic mask per image. In Tab. 34, we include additional semantic segmentation evaluation on ADE-150 (Zhou et al., 2019) and PC-459 (Mottaghi et al., 2014).


We employ the following Hugging Face model checkpoints: “google/owlv2-large-patch14” for OWLv2, “google/owlv2-large-patch14-ensemble” for OWLv2\*, “IDEA-Research/grounding-dino-tiny” for gDino-T, and “iSEE-Laboratory/llmdet_large” for LLMDet-L. OWLv2\* utilizes an ensemble of checkpoint weights after self-training and after fine-tuning the model on LVIS base, demonstrating improved open-world generalization compared to fine-tuning alone (Minderer et al., 2024). We provide per-domain performance of instance segmentation for all baselines, SAM 3, and human on SA-Co/Gold in Tab. 30 and on SA-Co/Silver in Tab. 31 and Tab. 32. We also include per-domain performance for the AI verifier ablation study in Tab. 9d. In Tab. 33, we compare with additional baselines using “IDEA-Research/grounding-dino-base” for gDino-B and “iSEE-Laboratory/llmdet_base” for LLMDet-B.


For OWLv2, GroundingDino, and LLMDet, we swept over the detection threshold at 0.1 intervals and determined the best threshold using the LVIS $\mathrm{cgF}_1$ metric for the box detection task. Then, we applied this threshold to compute $\mathrm{cgF}_1$ on the remaining datasets for the box detection and instance segmentation tasks. The detection threshold is set to 0.4 for LLMDet-L, LLMDet-B, and gDino-T; 0.3 for OWLv2\*; and 0.2 for OWLv2 and gDino-B. For DINO-X, we find the detection threshold 0.5 gives the best $\mathrm{cgF}_1$ metric. Additionally, we found that prompting multiple noun phrases at once for a given image greatly improved performance for GroundingDino and LLMDet, compared to prompting one noun phrase at a time. For example, we prompted GroundingDino and LLMDet with 30 prompts for SA-Co/Gold and 20 prompts for SA-Co/Silver, SA-Co/Bronze, and SA-Co/Bio.


For Gemini 2.5 Flash, we run inference via the Gemini API. For each (image, text query) pair, we prompt Gemini 2.5 using the same prompt template that is used in Gemini Flash 2.5 image segmentation demo (Paul Voigtlaender, Valentin Gabeur and Rohan Doshi, 2025) with the same generation settings. In addition, we prompt the model multiple times if there are any errors in generation, or parsing the result into a set of masks and bounding boxes.


<table><tr><td></td><td colspan="3">Average</td><td colspan="3">Metaclip</td><td colspan="3">SA-1B</td><td colspan="3">Crowded</td><td colspan="3">Food&amp;Drink</td><td colspan="3">Sports Equip.</td><td colspan="3">Attributes</td><td colspan="3">Wiki-Common</td></tr><tr><td></td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>gDino-T</td><td>3.3</td><td>0.15</td><td>16.2</td><td>2.9</td><td>0.21</td><td>13.9</td><td>3.1</td><td>0.20</td><td>15.4</td><td>0.28</td><td>0.08</td><td>3.4</td><td>0.96</td><td>0.10</td><td>9.8</td><td>1.1</td><td>0.10</td><td>11.2</td><td>13.8</td><td>0.29</td><td>47.3</td><td>0.70</td><td>0.06</td><td>12.1</td></tr><tr><td>OWLv2*</td><td>24.6</td><td>0.57</td><td>42.0</td><td>17.7</td><td>0.52</td><td>34.3</td><td>13.3</td><td>0.50</td><td>26.8</td><td>15.8</td><td>0.51</td><td>30.7</td><td>32.0</td><td>0.65</td><td>49.4</td><td>36.0</td><td>0.64</td><td>56.2</td><td>35.6</td><td>0.63</td><td>56.2</td><td>21.7</td><td>0.54</td><td>40.3</td></tr><tr><td>OWLv2</td><td>17.3</td><td>0.46</td><td>36.8</td><td>12.2</td><td>0.39</td><td>31.3</td><td>9.8</td><td>0.45</td><td>21.7</td><td>8.9</td><td>0.36</td><td>24.8</td><td>24.4</td><td>0.51</td><td>47.9</td><td>24.4</td><td>0.52</td><td>47.0</td><td>25.9</td><td>0.54</td><td>48.2</td><td>15.4</td><td>0.42</td><td>36.6</td></tr><tr><td>LLMDet-L</td><td>6.5</td><td>0.21</td><td>27.3</td><td>4.5</td><td>0.23</td><td>19.4</td><td>5.3</td><td>0.23</td><td>22.8</td><td>2.4</td><td>0.18</td><td>13.7</td><td>5.5</td><td>0.19</td><td>29.1</td><td>4.4</td><td>0.17</td><td>25.3</td><td>22.2</td><td>0.39</td><td>57.1</td><td>1.2</td><td>0.05</td><td>23.3</td></tr><tr><td>APE-D</td><td>16.4</td><td>0.40</td><td>36.9</td><td>12.6</td><td>0.42</td><td>30.1</td><td>2.2</td><td>0.22</td><td>10.0</td><td>7.2</td><td>0.35</td><td>20.3</td><td>22.7</td><td>0.51</td><td>45.0</td><td>31.8</td><td>0.56</td><td>56.5</td><td>26.7</td><td>0.47</td><td>57.3</td><td>11.6</td><td>0.29</td><td>39.5</td></tr><tr><td>DINO-X</td><td>21.3</td><td>0.38</td><td>55.2</td><td>17.2</td><td>0.35</td><td>49.2</td><td>19.7</td><td>0.48</td><td>40.9</td><td>12.9</td><td>0.34</td><td>37.5</td><td>30.1</td><td>0.49</td><td>61.7</td><td>28.4</td><td>0.41</td><td>69.4</td><td>31.0</td><td>0.42</td><td>74.0</td><td>9.7</td><td>0.18</td><td>53.5</td></tr><tr><td>Gemini 2.5</td><td>13.0</td><td>0.29</td><td>46.1</td><td>9.9</td><td>0.29</td><td>33.8</td><td>13.1</td><td>0.41</td><td>32.1</td><td>8.2</td><td>0.27</td><td>30.3</td><td>19.6</td><td>0.33</td><td>59.5</td><td>15.1</td><td>0.28</td><td>53.5</td><td>18.8</td><td>0.30</td><td>63.1</td><td>6.5</td><td>0.13</td><td>50.3</td></tr><tr><td>SAM 3</td><td>54.1</td><td>0.82</td><td>66.1</td><td>47.3</td><td>0.81</td><td>58.6</td><td>53.7</td><td>0.86</td><td>62.6</td><td>61.1</td><td>0.9</td><td>67.7</td><td>53.4</td><td>0.79</td><td>67.3</td><td>65.5</td><td>0.89</td><td>73.8</td><td>54.9</td><td>0.76</td><td>72.0</td><td>42.5</td><td>0.70</td><td>60.9</td></tr><tr><td>Human</td><td>72.8</td><td>0.94</td><td>77.0</td><td>64.1</td><td>0.94</td><td>68.5</td><td>64.3</td><td>0.97</td><td>66.6</td><td>70.4</td><td>0.94</td><td>75.3</td><td>78.3</td><td>0.96</td><td>81.2</td><td>80.4</td><td>0.97</td><td>83.1</td><td>80.2</td><td>0.95</td><td>84.4</td><td>71.6</td><td>0.89</td><td>80.1</td></tr></table>


_Table 30 Per-domain results of instance segmentation on SA-Co/Gold. $\star$: partially trained on LVIS._


---


<table><tr><td></td><td colspan="3">Average</td><td colspan="3">BDD100k</td><td colspan="3">DROID</td><td colspan="3">Ego4D</td><td colspan="3">MyFoodRepo-273</td><td colspan="3">GeoDE</td></tr><tr><td></td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>gDino-T</td><td>2.7</td><td>0.12</td><td>16.6</td><td>2.2</td><td>0.17</td><td>12.7</td><td>3.6</td><td>0.15</td><td>23.8</td><td>2.4</td><td>0.10</td><td>23.6</td><td>0.52</td><td>0.05</td><td>10.3</td><td>10.5</td><td>0.24</td><td>43.8</td></tr><tr><td>OWLv2*</td><td>11.5</td><td>0.32</td><td>33.0</td><td>15.0</td><td>0.46</td><td>32.6</td><td>11.1</td><td>0.36</td><td>30.7</td><td>7.6</td><td>0.23</td><td>33.0</td><td>20.0</td><td>0.44</td><td>45.4</td><td>27.6</td><td>0.50</td><td>55.2</td></tr><tr><td>OWLv2</td><td>7.6</td><td>0.23</td><td>31.1</td><td>7.6</td><td>0.31</td><td>24.7</td><td>7.3</td><td>0.25</td><td>29.0</td><td>5.5</td><td>0.18</td><td>30.7</td><td>13.4</td><td>0.32</td><td>41.8</td><td>12.9</td><td>0.28</td><td>46.2</td></tr><tr><td>LLMDet-L</td><td>7.1</td><td>0.17</td><td>28.7</td><td>1.5</td><td>0.08</td><td>17.1</td><td>2.3</td><td>0.10</td><td>23.2</td><td>2.1</td><td>0.08</td><td>26.6</td><td>0.90</td><td>0.06</td><td>15.0</td><td>21.0</td><td>0.37</td><td>56.8</td></tr><tr><td>APE-D</td><td>7.3</td><td>0.24</td><td>24.5</td><td>6.7</td><td>0.32</td><td>20.9</td><td>8.9</td><td>0.30</td><td>29.6</td><td>6.3</td><td>0.23</td><td>27.9</td><td>7.0</td><td>0.26</td><td>26.5</td><td>26.9</td><td>0.47</td><td>57.5</td></tr><tr><td>Gemini 2.5</td><td>8.3</td><td>0.19</td><td>38.1</td><td>5.1</td><td>0.19</td><td>26.9</td><td>4.5</td><td>0.14</td><td>31.8</td><td>0.32</td><td>0.01</td><td>32.4</td><td>8.6</td><td>0.24</td><td>35.9</td><td>16.4</td><td>0.26</td><td>62.9</td></tr><tr><td>SAM 3</td><td>49.6</td><td>0.76</td><td>65.2</td><td>46.6</td><td>0.78</td><td>60.1</td><td>45.6</td><td>0.76</td><td>60.4</td><td>38.6</td><td>0.62</td><td>62.6</td><td>53.0</td><td>0.79</td><td>67.2</td><td>70.1</td><td>0.89</td><td>78.7</td></tr></table>


_Table 31 Per-domain results of instance segmentation on SA-Co/Silver. $\star$ : partially trained on LVIS._


<table><tr><td></td><td colspan="3">iNaturalist-2017</td><td colspan="3">National Gallery of Art</td><td colspan="3">SA-V</td><td colspan="3">YT-Temporal-1B</td><td colspan="3">Fathomnet</td></tr><tr><td></td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td><td>cgF1</td><td>IL_MCC</td><td>pmF1</td></tr><tr><td>gDino-T</td><td>0.00</td><td>0.00</td><td>3.7</td><td>0.88</td><td>0.09</td><td>9.8</td><td>4.2</td><td>0.19</td><td>22.0</td><td>2.5</td><td>0.16</td><td>15.6</td><td>0.00</td><td>0.00</td><td>0.74</td></tr><tr><td>OWLv2*</td><td>5.6</td><td>0.14</td><td>40.3</td><td>6.7</td><td>0.31</td><td>21.7</td><td>11.5</td><td>0.32</td><td>35.8</td><td>9.9</td><td>0.38</td><td>26.2</td><td>0.07</td><td>0.01</td><td>9.3</td></tr><tr><td>OWLv2</td><td>3.3</td><td>0.10</td><td>33.2</td><td>5.8</td><td>0.25</td><td>23.1</td><td>10.8</td><td>0.32</td><td>33.6</td><td>9.9</td><td>0.35</td><td>28.3</td><td>-0.20</td><td>-0.01</td><td>20.7</td></tr><tr><td>LLMDet-L</td><td>32.7</td><td>0.46</td><td>71.2</td><td>1.8</td><td>0.13</td><td>13.7</td><td>5.0</td><td>0.19</td><td>26.3</td><td>3.2</td><td>0.16</td><td>20.0</td><td>0.65</td><td>0.04</td><td>16.9</td></tr><tr><td>APE-D</td><td>1.1</td><td>0.10</td><td>11.2</td><td>3.1</td><td>0.21</td><td>14.7</td><td>7.6</td><td>0.26</td><td>28.8</td><td>5.8</td><td>0.28</td><td>20.8</td><td>0.1</td><td>0.01</td><td>7.2</td></tr><tr><td>Gemini 2.5</td><td>26.6</td><td>0.36</td><td>74.0</td><td>5.6</td><td>0.20</td><td>27.8</td><td>7.4</td><td>0.22</td><td>33.8</td><td>6.9</td><td>0.23</td><td>29.9</td><td>2.1</td><td>0.08</td><td>25.6</td></tr><tr><td>SAM 3</td><td>65.8</td><td>0.82</td><td>80.7</td><td>38.1</td><td>0.66</td><td>57.6</td><td>44.4</td><td>0.67</td><td>66.1</td><td>42.1</td><td>0.72</td><td>58.4</td><td>51.5</td><td>0.86</td><td>60.0</td></tr></table>


_Table 32 Per-domain results of instance segmentation on SA-Co/Silver continued. $\star$ : partially trained on LVIS._


# F.2 Visual Exemplars and Interactivity


In Tab. 35, visual exemplar experiments, we report performance in 3 settings: (1) text prompt only, (2) visual prompt only, and (3) both text and visual prompt. We note that (2) is quite ambiguous. For example, given a visual example of a dog, one could want to detect all dogs, or only dogs of the same color or breed. As a result, SAM 3 performs worse on SA-Co/Gold in setting (2) compared to (1). Therefore, setting (3) is better suited, as the text lifts most of the ambiguity, and the additional input box gives a hint for unfamiliar concepts.


# F.3 Few-Shot Fine-tuning


We evaluate SAM 3's object detection capabilities on real-world data through comprehensive zero-shot and few-shot experiments using two established benchmarks: OdinW13 Li et al. (2022a) and Roboflow-100VL Robicheaux et al. (2025). These benchmarks encompass 13 and 100 diverse object detection datasets, respectively, capturing a wide range of real-world scenarios with standardized train and test splits that enable fair comparison with existing methods.


Few-shot training and evaluation. For OdinW13 few-shot experiments, we train on all three official few-shot training splits and report mean performance with standard deviation on the test split. For Roboflow-100VL, we utilize the official FSOD training splits provided by the benchmark and report numbers on the test split. We treat few-shot fine-tuning runs similarly to traditional training runs, but with some differences. We train for 40 epochs a reduced learning rate that is one-tenth of the standard value on a batch size of 2. Since these benchmarks focus exclusively on object detection without mask annotations, we disable all mask-specific components and losses during training.


OdinW13 results. Fig. 20a presents our few-shot performance on OdinW13, comparing SAM 3 against previous state-of-the-art methods Ren et al. (2024a); Wu et al. (2024b); Xu et al. (2023); Zhang et al. (2022b). We report mean BoxAP averaged across all 13 datasets, with SAM 3 consistently achieving superior performance and establishing new state-of-the-art results. Complete dataset-specific results for each OdinW13 dataset are provided in Fig. 20b.


Roboflow-100VL results. Tab. 36 summarizes our comprehensive evaluation across zero-shot, few-shot, and full fine-tuning settings on Roboflow-100VL, with results averaged across all 100 datasets. While SAM 3 underperforms the current state-of-the-art Liu et al. (2023) in zero-shot evaluation, it surpasses leading methods Liu et al. (2023); Chen et al. (2024a) in both few-shot and full fine-tuning scenarios. This demonstrates SAM 3's strong visual generalization capabilities when provided with task-specific training data. We attribute


---


<table><tr><td rowspan="3">Model</td><td colspan="8">Box Detection</td></tr><tr><td colspan="2">LVIS</td><td colspan="2">COCO</td><td colspan="4">SA-Co</td></tr><tr><td>cgF1</td><td>AP</td><td>AP</td><td>APo</td><td>Gold cgF1</td><td>Silver cgF1</td><td>Bronze cgF1</td><td>Bio pmF1</td></tr><tr><td>Human</td><td>-</td><td>-</td><td>-</td><td>-</td><td>74.0</td><td>-</td><td>-</td><td>-</td></tr><tr><td>OWLv2 (Minderer et al., 2024)</td><td>19.9</td><td>35.2</td><td>38.2</td><td>42.4</td><td>16.9</td><td>7.1</td><td>4.1</td><td>0.95</td></tr><tr><td>OWLv2* (Minderer et al., 2024)</td><td>30.2</td><td>45.5</td><td>46.1</td><td>23.9</td><td>24.5</td><td>11.0</td><td>12.0</td><td>0.08</td></tr><tr><td>gDino-B (Liu et al., 2024a)</td><td>15.8</td><td>25.7</td><td>52.5</td><td>45.5</td><td>6.0</td><td>4.2</td><td>12.2</td><td>0.90</td></tr><tr><td>gDino-T (Liu et al., 2024a)</td><td>15.1</td><td>20.5</td><td>45.7</td><td>35.3</td><td>3.4</td><td>2.5</td><td>7.6</td><td>0.35</td></tr><tr><td>LLMDet-L (Fu et al., 2025)</td><td>39.3</td><td>42.0</td><td>55.6</td><td>49.8</td><td>6.8</td><td>6.7</td><td>14.0</td><td>0.17</td></tr><tr><td>LLMDet-B (Fu et al., 2025)</td><td>36.6</td><td>37.8</td><td>54.2</td><td>39.4</td><td>5.0</td><td>7.3</td><td>14.5</td><td>0.27</td></tr><tr><td>APE-D (Shen et al., 2024)</td><td>-</td><td>59.6†</td><td>58.3†</td><td>-</td><td>17.3</td><td>7.7</td><td>14.3</td><td>0.00</td></tr><tr><td>DINO-X (Ren et al., 2025)</td><td>-</td><td>52.4†</td><td>56.0†</td><td>-</td><td>22.5δ</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Gemini 2.5 (Comanici et al., 2025)</td><td>16.1</td><td>-</td><td>-</td><td>-</td><td>14.4</td><td>9.4</td><td>8.2</td><td>12.4</td></tr><tr><td>SAM 3</td><td>41.0</td><td>53.7</td><td>56.4</td><td>55.7</td><td>55.7</td><td>50.0</td><td>47.1</td><td>56.3</td></tr></table>


_Table 33 Additional evaluation on box detection. $\mathrm{AP}_{\circ}$ corresponds to COCO-O accuracy, $\star$: partially trained on LVIS, †: from original papers, $\delta$: from DINO-X API. Gray numbers indicate usage of respective closed set training data (LVIS/COCO). See §E.4 for details on human performance._


<table><tr><td rowspan="2">Model</td><td colspan="5">Semantic Segmentation</td></tr><tr><td>ADE-847 mIoU</td><td>ADE-150 mIoU</td><td>PC-59 mIoU</td><td>PC-459 mIoU</td><td>Cityscapes mIoU</td></tr><tr><td>APE-D</td><td>9.2</td><td>30.0</td><td>58.5</td><td>21.8</td><td>44.2</td></tr><tr><td>SAM 3</td><td>13.8</td><td>39.0</td><td>60.8</td><td>18.8</td><td>65.2</td></tr></table>


_Table 34 Additional evaluation on semantic segmentation._


the zero-shot performance gap to the use of specialized, dataset-specific prompts that may lack broad generalizability in Roboflow-100VL. However, even minimal fine-tuning closes this gap and enables substantial performance improvements. Roboflow-100VL also categorizes its 100 datasets into seven dataset types; we report averages per each such dataset type in Tab. 37.


# F.4 Object Counting


We evaluate an internal SAM 3 checkpoint on object counting benchmarks CountBench (Paiss et al., 2023) and PixMo-Count (Deitke et al., 2025) to compare with MLLMs (Wang et al., 2024; Deitke et al., 2025; Comanici et al., 2025) and detection expert models (Ren et al., 2025). See Tab. 38 for results. The metrics include Accuracy(\%) and Mean Absolute Error (MAE). CountBench (Paiss et al., 2023) contains 540 images and their captions, with 2-10 objects in each image. By removing images with unavailable links, we test on 487 images. PixMo-Count (Deitke et al., 2025) contains 540 images and their text descriptions in the form of simple noun phrases, with 2 to 10 objects in each image. By removing images with unavailable links, we test on 529 images.


To evaluate MLLMs on CountBench, we apply the same question set as Molmo (Deitke et al., 2025), which is inherited from PaliGemma (Beyer et al., 2024). When evaluating SAM 3 on CountBench, we modify the question sentence to the simple noun phrase. To evaluate MLLMs on PixMo-Count, we construct the question as “How many {} are there in this image” or “Count the {}.”, where {}. is the simple noun phrase provided by PixMo-Count annotations.


We find that presence token does not help SAM 3 on counting tasks, so we do not use it. For a group of


<table><tr><td rowspan="2">Model</td><td colspan="3">SA-Co/Gold</td></tr><tr><td>pmF1T</td><td>pmF1I</td><td>pmF1T+I</td></tr><tr><td>T-Rex2 (Jiang et al., 2024)</td><td>-</td><td>57.6</td><td>-</td></tr><tr><td>SAM 3</td><td>66.4</td><td>69.5</td><td>74.6</td></tr></table>


_Table 35 Visual prompting on SA-Co/Gold. We report $\mathrm{pmF_1}$ metric in different prompt types: T (text-only), I (image-only), and T+I (combined text and image)._


---


<table><tr><td>Model</td><td>Zero-shot</td><td>1-Shot</td><td>3-Shot</td><td>5-Shot</td><td>10-Shot</td><td>All</td></tr><tr><td></td><td colspan="6">AP</td></tr><tr><td>GLIPv2-H</td><td>55.5</td><td>61.7 ± 0.5</td><td>64.1 ± 0.8</td><td>64.4 ± 0.6</td><td>65.9 ± 0.3</td><td>70.4</td></tr><tr><td>GLEE-Pro</td><td>53.4</td><td>59.4 ± 1.5</td><td>61.7 ± 0.5</td><td>64.3 ± 1.3</td><td>65.6 ± 0.4</td><td>69.0</td></tr><tr><td>MQ-GLIP-L</td><td>54.1</td><td>62.4</td><td>64.2</td><td>65.4</td><td>66.6</td><td>71.3</td></tr><tr><td>Grounding DINO 1.5 Pro</td><td>58.7</td><td>62.4 ± 1.1</td><td>66.3 ± 1.0</td><td>66.9 ± 0.2</td><td>67.9 ± 0.3</td><td>72.4</td></tr><tr><td>SAM 3</td><td>61.0</td><td>66.1 ± 0.8</td><td>69.1 ± 0.0</td><td>70.2 ± 0.3</td><td>71.8 ± 0.4</td><td>75.8</td></tr></table>


_(a) Comparison of different models under few-shot settings on ODinW13._


<table><tr><td></td><td>Aerialmaritimedrone(I)</td><td>Aquarium</td><td>Rabbits</td><td>Egohands(g)</td><td>NAMushrooms</td><td>Packages</td><td>PascalVOC</td></tr><tr><td></td><td colspan="7">AP</td></tr><tr><td>0-Shot</td><td>20.9</td><td>44.5</td><td>80.9</td><td>70.0</td><td>89.6</td><td>74.6</td><td>65.2</td></tr><tr><td>1-Shot</td><td>31.5 ± 2.6</td><td>46.5 ± 0.4</td><td>82.3 ± 1.6</td><td>73.4 ± 0.3</td><td>94.8 ± 0.6</td><td>82.9 ± 1.6</td><td>69.0 ± 0.9</td></tr><tr><td>3-Shot</td><td>35.9 ± 1.0</td><td>49.5 ± 0.8</td><td>84.8 ± 0.2</td><td>72.3 ± 1.0</td><td>97.7 ± 1.0</td><td>88.4 ± 1.7</td><td>70.1 ± 0.3</td></tr><tr><td>5-Shot</td><td>37.7 ± 0.9</td><td>53.4 ± 1.4</td><td>83.8 ± 0.2</td><td>73.1 ± 1.0</td><td>98.1 ± 1.4</td><td>87.9 ± 1.3</td><td>70.6 ± 0.3</td></tr><tr><td>10-Shot</td><td>39.1 ± 0.8</td><td>54.1 ± 0.7</td><td>84.6 ± 0.7</td><td>73.5 ± 0.6</td><td>99.7 ± 0.4</td><td>92.7 ± 1.3</td><td>70.7 ± 0.1</td></tr><tr><td>All</td><td>40.1</td><td>61.5</td><td>84.7</td><td>79.6</td><td>100.0</td><td>98.0</td><td>77.4</td></tr><tr><td></td><td>Raccoon</td><td>Shellfish</td><td>Vehicles</td><td>Pistols</td><td>Pothole</td><td>ThermalDP</td><td>13-Average</td></tr><tr><td></td><td colspan="7">AP</td></tr><tr><td>0-Shot</td><td>65.9</td><td>63.7</td><td>65.0</td><td>62.2</td><td>29.3</td><td>61.2</td><td>61.0</td></tr><tr><td>1-Shot</td><td>69.8 ± 3.1</td><td>64.8 ± 1.5</td><td>67.0 ± 1.9</td><td>70.5 ± 1.0</td><td>36.5 ± 1.0</td><td>70.2 ± 2.6</td><td>66.1 ± 0.8</td></tr><tr><td>3-Shot</td><td>77.9 ± 2.5</td><td>64.0 ± 2.1</td><td>68.0 ± 1.0</td><td>71.5 ± 0.7</td><td>40.5 ± 0.9</td><td>77.9 ± 2.6</td><td>69.1 ± 0.0</td></tr><tr><td>5-Shot</td><td>79.9 ± 1.6</td><td>64.8 ± 2.3</td><td>67.3 ± 0.8</td><td>73.1 ± 0.9</td><td>42.7 ± 1.1</td><td>79.9 ± 3.0</td><td>70.2 ± 0.3</td></tr><tr><td>10-Shot</td><td>84.3 ± 0.3</td><td>66.1 ± 0.8</td><td>68.0 ± 1.5</td><td>73.1 ± 0.5</td><td>45.4 ± 0.9</td><td>82.2 ± 1.9</td><td>71.8 ± 0.4</td></tr><tr><td>All</td><td>86.4</td><td>58.8</td><td>72.1</td><td>78.8</td><td>58.6</td><td>89.7</td><td>75.8</td></tr></table>


_(b) ODimW13 per-dataset results for SAM 3._


_Figure 20 Zero-shot and few-shot results on ODinW13._


objects, we find that SAM 3 outputs predictions for both each individual and the group as a whole, which contradicts the counting task.


As a post-processing step, we perform Non-Maximal Suppression (NMS) to remove duplicate detections. Instead of the usual Intersection-over-Union (IoU) criterion, we use Interaction over Minimum (IoM), where the area of overlap is divided by the area of the smaller mask, rather than by the area of the union. By doing this, we can detect whole vs. part situations: if a mask is fully covered by another, the IoM will be high, even if the covering mask is much bigger (which would lead to low IoU). We set the IoM threshold to 0.5 in our NMS process. Finally, we select the predictions with confidence higher than 0.5 as the final predictions and count the number of predictions as the counting result.


# F.5 Video PCS Details


In this section, we provide additional details for the video PCS evaluation (in §6 and Tab. 5).


Benchmarks. We evaluate the video PCS capabilities of the SAM 3 model based on an input text prompt (similar to the open-vocabulary video instance segmentation task (Wang et al., 2023)) on both our collected video benchmark SA-Co/VEval and public benchmarks. For SA-Co/VEval, we evaluate separately on each subset (SA-V, YT-Temporal-1B, and SmartGlasses) based on their data sources, and report classification-gated F1 ($cgF_1$), phrase-based HOTA ($pHOTA$), and Track Every Thing Accuracy (TETA). The SA-Co/VEval benchmarks contain a large number of noun phrases (5.1K in SA-V and YT-Temporal-1B subsets and 4.9K in


<table><tr><td>Model</td><td>Zero-Shot</td><td>10-Shot AP</td><td>All</td></tr><tr><td>Grounding Dino</td><td>15.7</td><td>33.7</td><td>—</td></tr><tr><td>LW-DETRm</td><td>—</td><td>—</td><td>59.8</td></tr><tr><td>SAM 3</td><td>15.2</td><td>36.5</td><td>61.6</td></tr></table>


_Table 36 Comparison on Roboflow100-VL._


---


<table><tr><td></td><td>Aerial</td><td>Document</td><td>Flora-Fauna</td><td>Industrial</td><td>Medical</td><td>Other</td><td>Sports</td><td>Average</td></tr><tr><td></td><td colspan="8">AP</td></tr><tr><td>0-Shot</td><td>24.0</td><td>12.9</td><td>23.9</td><td>9.0</td><td>2.6</td><td>17.0</td><td>15.6</td><td>15.2</td></tr><tr><td>10-Shot</td><td>35.4</td><td>35.8</td><td>39.6</td><td>40.4</td><td>25.7</td><td>35.0</td><td>40.3</td><td>36.5</td></tr><tr><td>All</td><td>56.9</td><td>64.2</td><td>60.1</td><td>68.4</td><td>52.5</td><td>63.8</td><td>61.5</td><td>61.6</td></tr></table>


_Table 37 SAM 3 Roboflow100-VL results by dataset type._


<table><tr><td rowspan="2">Model</td><td rowspan="2">Presence</td><td rowspan="2">IoM</td><td colspan="2">CountBench</td><td colspan="2">PixMo-Count</td></tr><tr><td>MAE ↓</td><td>Acc (%) ↑</td><td>MAE ↓</td><td>Acc (%) ↑</td></tr><tr><td>SAM 3</td><td>×</td><td>×</td><td>0.34</td><td>84.8</td><td>0.34</td><td>76.9</td></tr><tr><td>SAM 3</td><td>✓</td><td>×</td><td>0.50</td><td>83.7</td><td>0.41</td><td>75.8</td></tr><tr><td>SAM 3</td><td>×</td><td>✓</td><td>0.11</td><td>94.0</td><td>0.21</td><td>86.3</td></tr><tr><td>SAM 3</td><td>✓</td><td>✓</td><td>0.38</td><td>89.3</td><td>0.29</td><td>85.2</td></tr></table>


_Table 38 Ablation on counting results on an internal SAM 3 checkpoint._


SmartGlasses), and provides each video with a list of noun phrases as text prompts. During evaluation, for each evaluation video, we prompt SAM 3 with the list of noun phrases provided for that video, as shown in Tab. 39 (a, b, c).


For public benchmarks, we evaluate on LVVIS (Wang et al., 2023), BURST (Athar et al., 2023), YTVIS (Ke et al., 2022), OVIS (Qi et al., 2022), BDD100K (Yu et al., 2020), GMOT40 (Bai et al., 2021), and DeepSeaMOT (Barnard et al., 2025), and report the official metrics on each dataset (for DeepSeaMOT, we report the average performance over its 4 subsets). These public benchmarks are often based on a set of categories, with a relatively large vocabulary size in LVVIS and BURST (1196 categories in LVVIS and 482 categories in BURST) and much smaller numbers of categories in other datasets. We use the category name as the text prompt, and prompt SAM 3 with all category names in the dataset on every evaluation video, as shown in Tab. 39 (d).


Video PCS Metrics. Similar to its definition in the image domain in §E.3, we define the classification-gated F1 ($cgF_1$) metric on videos as the multiplication between the video-level Matthews correlation coefficient (VL_MCC) on whether the noun phrase exists in the video and the localization positive macro F1 ($pmF_1$) on positive noun phrases. To decide whether a predicted masklet matches a ground-truth masklet, we measure their volume intersection-over-union (IoU), defined by their total intersection volume divided by their total union volume over the video. When computing $pmF_1$, we averaged the results over multiple volume IoU thresholds from 0.5 to 0.95 with increments of 0.05, similar to how it is computed on images.


We also evaluate the phrase-based HOTA (pHOTA) metric, where we compute the Higher Order Tracking Accuracy (HOTA) metric (Luiten et al., 2021) over all video-NP pairs along with their breakdown into phrase-based detection accuracy (pDetA) and phrase-based association accuracy (pAssA). As the HOTA metric was originally designed for category-based evaluation, to get its phrase-based variant pHOTA for open-vocabulary prompts, we remap each video-NP pair in the evaluation benchmark as a new unique video ID and then set all ground-truth annotations and predictions to have the same category ID (i.e., the total number of video IDs after remapping equals the total number of video-NP pairs in the evaluation benchmark). That is, each video-NP pair in the benchmark is treated as an isolated sample for prediction and evaluation, and the results are aggregated over all video-NP pairs in a class-agnostic manner. More specifically, we save the remapped ground-truth annotations and the predictions as JSON files under the YTVIS format, and use the TrackEval package (Jonathon Luiten, 2020) to obtain the mask HOTA statistics on this remapped dataset (using the YTVIS dataset wrapper in TrackEval along with its default parameters), and report their results as pHOTA, pDetA, and pAssA. Similarly, we also evaluate the Track Every Thing Accuracy (TETA) metric (Li et al., 2022c) over the masklet predictions on these datasets.


Baselines. We compare the SAM 3 model with several baselines, including GLEE (Wu et al., 2024a) (a previous work on open-vocabulary image/video segmentation), "LLMDet as detector + SAM 3 Tracker", by replacing the Detector component in SAM 3 with a recent open-vocabulary detector LLMDet (Fu et al., 2025), and "SAM 3 Detector + T-by-D as tracker", by replacing the Tracker in SAM 3 with an association module similar as in tracking-by-detection approaches (Wojke et al., 2017; Zhang et al., 2022c).


---


For GLEE (Wu et al., 2024a), we follow its official implementation. Since GLEE supports taking as inputs multiple text prompt simultaneously, we evaluate it in two ways: a) prompting it with all the noun phrases from an evaluation video at once, denoted as “GLEE (prompted w/ all NPs at once)” in Tab. 39, and b) looping over each noun phrase in the evaluation video and prompting GLEE with one noun phrase at a time, denoted as “GLEE (prompted w/ one NP at a time)". We find that for open-vocabulary segmentation on videos, it is usually better to prompt GLEE with one noun phrase at a time instead of prompting it with all noun phrases at once.


For "LLMDet as Detector + SAM 3 Tracker", we replace the detection outputs from the SAM 3 detector with LLMDet (Fu et al., 2025) bounding box outputs, and obtain the mask output by prompting it with the SAM 3 component. Then we apply the SAM 3 Tracker similar to how it is applied over the SAM 3 Detector output. We also note that GLEE and LLMDet have not been trained on the noun phrases in the SA-Co dataset, so their results should be seen as zero-shot on the SA-Co/VEval benchmark.


For "SAM 3 Detector + T-by-D as tracker", we replace the SAM 3 Tracker with a detection-to-masklet association module as commonly used in the tracking-by-detection paradigm, e.g. Wojke et al. (2017); Zhang et al. (2022c). The detection-to-masklet association module tries to match the masklets already tracked in previous frames with detected objects in the current frame, based on a dot product between the visual features of each detected object and the visual features of the past 16 frames of a masklet. If a high-confidence detection isn't matched to any existing masklet, we add it as a new object and start a new masklet for it. The association module is trained on the SA-Co dataset.


Results. As shown in Tab. 39, SAM 3 largely outperforms these baselines across the benchmarks. On SA-Co/VEval with a very large number of noun phrases, SAM 3 excels in both frame-level detection (pDetA) and cross-frame association (pAssA). Comparisons with "LLMDet as Detector + SAM 3 Tracker" and "SAM 3 Detector + T-by-D as tracker" demonstrate that both the Detector module and the Tracker module in SAM 3 play a critical role in the final video performance. In public benchmarks, SAM 3 also achieves a strong performance, including new state-of-the-art results on LVVIS and OVIS. We also note that GLEE and LLMDet have not been trained on the SA-Co dataset, so their results should be seen as zero-shot on SA-Co/VEval. In addition, the SmartGlasses subset in SA-Co/VEval contains many egocentric videos, which might be out of the training distribution for GLEE and LLMDet.


Strategies on Temporal Disambiguation. As described in §C.3, SAM 3 adopts several strategies to address ambiguities in videos. In Tab. 39, we also report two other settings where we turn off all these temporal disambiguation strategies ("SAM 3 w/o any temporal disambiguation"). The results show that the disambiguation strategies boost the video PCS performance (especially under the pHOTA metric). We also find that the disambiguation strategies notably improve the qualitative outputs on videos.


# F.6 PVS Details


We evaluate SAM 3 on a range of Promptable Video Segmentation (PVS) tasks as in Ravi et al. (2024).


Video Object Segmentation (VOS). The VOS task requires tracking an object throughout a video given an input segmentation mask. As shown in Tab. 6, we compare SAM 3 with recent state-of-the-art models on the VOS task, including SAMURAI (Yang et al., 2024), SAM2Long (Ding et al., 2024), and SeC (Zhang et al., 2025). SAM 3 brings gains on all datasets, including the challenging MOSEv2 benchmark (Ding et al., 2025) and datasets with long videos such as LVOSv2 (Hong et al., 2024).


Interactive Image Segmentation. We evaluate SAM 3 on the 37-dataset benchmark introduced in Ravi et al. (2024) for the interactive image segmentation task. As shown in Tab. 7, SAM 3 outperforms SAM 1 and SAM 2 on average mIoU, producing more accurate segmentation masks when prompted with 1 or 5 clicks.


Interactive Video Segmentation. We follow interactive offline and online evaluation protocol Ravi et al. (2024) and compare SAM 3 with baseline methods, including SAM 2, SAM + XMem++, and SAM + Cutie. The interactive offline evaluation involves multiple passes over the entire video, while the interactive online evaluation involves only one pass over the entire video. We use the same 9 zero-shot datasets and 3 clicks per interacted frame as in Ravi et al. (2024) (see Sec. F.1.2 of Ravi et al. (2024) for details). The results are in Fig. 21, where SAM 3 achieves better overall performance in both interactive offline and online evaluation.


---


<table><tr><td rowspan="2">Model</td><td colspan="7">SA-Co/VEval SA-V val (1.5K NPs)</td><td colspan="7">SA-Co/VEval SA-V test (1.5K NPs)</td></tr><tr><td>cgF1</td><td>VL_MCC</td><td>pmF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>TETA</td><td>cgF1</td><td>VL_MCC</td><td>pmF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>TETA</td></tr><tr><td>Human</td><td>53.9</td><td>0.86</td><td>62.4</td><td>71.7</td><td>56.9</td><td>90.8</td><td>68.9</td><td>53.1</td><td>0.87</td><td>60.8</td><td>70.5</td><td>55.5</td><td>90.2</td><td>69.1</td></tr><tr><td>GLEET( prompted w/ all NPs at once)</td><td>0.1</td><td>0.17</td><td>0.4</td><td>7.7</td><td>2.6</td><td>28.5</td><td>6.5</td><td>0.1</td><td>0.22</td><td>0.6</td><td>8.7</td><td>2.8</td><td>30.9</td><td>7.1</td></tr><tr><td>GLEET( prompted w/ one NP at a time)</td><td>0.2</td><td>0.06</td><td>2.5</td><td>12.4</td><td>3.9</td><td>41.7</td><td>15.7</td><td>0.1</td><td>0.05</td><td>2.2</td><td>11.8</td><td>3.4</td><td>43.1</td><td>14.9</td></tr><tr><td>LLMDet† as detector + SAM 3 Tracker</td><td>1.6</td><td>0.11</td><td>14.1</td><td>30.5</td><td>13.0</td><td>72.1</td><td>29.3</td><td>2.3</td><td>0.15</td><td>14.7</td><td>30.1</td><td>12.0</td><td>76.0</td><td>28.5</td></tr><tr><td>SAM 3 Detector + T-by-D as tracker</td><td>23.7</td><td>0.68</td><td>34.7</td><td>57.6</td><td>44.5</td><td>75.3</td><td>53.4</td><td>25.7</td><td>0.72</td><td>35.8</td><td>55.7</td><td>40.2</td><td>78.3</td><td>53.2</td></tr><tr><td>SAM 3 w/o any temporal disambiguation</td><td>24.4</td><td>0.56</td><td>43.4</td><td>57.4</td><td>40.1</td><td>83.1</td><td>57.4</td><td>27.1</td><td>0.63</td><td>42.9</td><td>55.9</td><td>37.9</td><td>83.4</td><td>57.1</td></tr><tr><td>SAM 3</td><td>29.3</td><td>0.66</td><td>44.5</td><td>60.7</td><td>44.7</td><td>83.2</td><td>57.3</td><td>30.3</td><td>0.69</td><td>43.7</td><td>58.0</td><td>40.9</td><td>83.4</td><td>56.8</td></tr></table>


_(a) Results on SA-Co/VEval SA-V val and test_


<table><tr><td rowspan="2">Model</td><td colspan="7">SA-Co/VEval YT-Temporal-1B val (1.4K NPs)</td><td colspan="7">SA-Co/VEval YT-Temporal-1B test (1.5K NPs)</td></tr><tr><td>cgF1</td><td>VL_MCC</td><td>pmF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>TETA</td><td>cgF1</td><td>VL_MCC</td><td>pmF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>TETA</td></tr><tr><td>Human</td><td>71.3</td><td>0.98</td><td>73.1</td><td>78.3</td><td>68.6</td><td>89.6</td><td>83.4</td><td>71.2</td><td>0.97</td><td>73.2</td><td>78.4</td><td>68.8</td><td>89.7</td><td>84.3</td></tr><tr><td>GLEET( prompted w/ all NPs at once)</td><td>1.8</td><td>0.26</td><td>7.0</td><td>17.1</td><td>7.3</td><td>42.0</td><td>16.9</td><td>1.6</td><td>0.24</td><td>6.7</td><td>16.7</td><td>6.8</td><td>42.6</td><td>16.5</td></tr><tr><td>GLEET( prompted w/ one NP at a time)</td><td>2.3</td><td>0.23</td><td>10.3</td><td>19.5</td><td>9.2</td><td>42.0</td><td>23.2</td><td>2.2</td><td>0.22</td><td>9.9</td><td>18.9</td><td>8.5</td><td>42.9</td><td>22.5</td></tr><tr><td>LLMDet† as detector + SAM 3 Tracker</td><td>10.5</td><td>0.39</td><td>26.8</td><td>39.6</td><td>20.1</td><td>78.3</td><td>37.0</td><td>8.0</td><td>0.33</td><td>24.1</td><td>37.9</td><td>18.7</td><td>77.0</td><td>33.4</td></tr><tr><td>SAM 3 Detector + T-by-D as tracker</td><td>46.0</td><td>0.90</td><td>51.3</td><td>68.5</td><td>61.0</td><td>77.6</td><td>70.4</td><td>47.6</td><td>0.93</td><td>51.3</td><td>68.2</td><td>60.8</td><td>77.0</td><td>70.4</td></tr><tr><td>SAM 3 w/o any temporal disambiguation</td><td>47.0</td><td>0.84</td><td>55.8</td><td>68.6</td><td>57.8</td><td>82.0</td><td>70.1</td><td>47.3</td><td>0.84</td><td>56.3</td><td>67.8</td><td>57.1</td><td>81.2</td><td>69.8</td></tr><tr><td>SAM 3</td><td>50.2</td><td>0.88</td><td>57.2</td><td>70.5</td><td>60.5</td><td>82.7</td><td>70.8</td><td>50.8</td><td>0.89</td><td>57.2</td><td>69.9</td><td>60.2</td><td>81.7</td><td>70.5</td></tr></table>


_(b) Results on SA-Co/VEval YT-Temporal-1B val and test_


<table><tr><td rowspan="2">Model</td><td colspan="7">SA-Co/VEval SmartGlasses val (2.2K NPs)</td><td colspan="7">SA-Co/VEval SmartGlasses test (2.2K NPs)</td></tr><tr><td>cgF1</td><td>VL_MCC</td><td>pmF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>TETA</td><td>cgF1</td><td>VL_MCC</td><td>pmF1</td><td>pHOTA</td><td>pDetA</td><td>pAssA</td><td>TETA</td></tr><tr><td>Human</td><td>53.9</td><td>0.91</td><td>59.6</td><td>68.1</td><td>53.0</td><td>87.9</td><td>71.0</td><td>58.5</td><td>0.92</td><td>63.5</td><td>72.3</td><td>58.8</td><td>89.4</td><td>77.1</td></tr><tr><td>GLEET( prompted w/ all NPs at once)</td><td>0.0</td><td>0.15</td><td>0.2</td><td>4.1</td><td>1.4</td><td>13.1</td><td>8.4</td><td>0.0</td><td>0.16</td><td>0.3</td><td>4.7</td><td>1.6</td><td>15.3</td><td>9.3</td></tr><tr><td>GLEET( prompted w/ one NP at a time)</td><td>0.1</td><td>0.14</td><td>0.5</td><td>4.7</td><td>1.4</td><td>16.0</td><td>13.3</td><td>0.1</td><td>0.14</td><td>0.4</td><td>5.6</td><td>1.8</td><td>18.2</td><td>14.7</td></tr><tr><td>LLMDet† as detector + SAM 3 Tracker</td><td>0.3</td><td>0.02</td><td>14.1</td><td>16.4</td><td>3.8</td><td>71.4</td><td>36.9</td><td>0.3</td><td>0.02</td><td>16.8</td><td>18.6</td><td>4.7</td><td>74.1</td><td>39.0</td></tr><tr><td>SAM 3 Detector + T-by-D as tracker</td><td>27.2</td><td>0.84</td><td>32.4</td><td>56.2</td><td>46.6</td><td>68.4</td><td>60.6</td><td>29.7</td><td>0.85</td><td>35.1</td><td>60.0</td><td>50.6</td><td>71.7</td><td>61.2</td></tr><tr><td>SAM 3 w/o any temporal disambiguation</td><td>30.9</td><td>0.71</td><td>43.5</td><td>58.1</td><td>43.2</td><td>78.7</td><td>65.1</td><td>34.4</td><td>0.75</td><td>45.8</td><td>61.6</td><td>47.0</td><td>81.2</td><td>66.0</td></tr><tr><td>SAM 3</td><td>33.5</td><td>0.76</td><td>44.4</td><td>60.2</td><td>46.2</td><td>79.3</td><td>65.0</td><td>36.4</td><td>0.78</td><td>46.6</td><td>63.6</td><td>50.0</td><td>81.5</td><td>65.9</td></tr></table>


_(c) Results on SA-Co/VEval SmartGlasses val and test_


<table><tr><td rowspan="2">Model</td><td colspan="3">LVVIS test (1.2K NPs)</td><td colspan="3">BURST test (482 NPs)</td><td>YTVIS21 val (40 NPs)</td><td>OVIS val (25 NPs)</td><td>BDD100K val (8 NPs)</td><td>GMOT40 (10 NPs)</td><td>DeepSeaMOT (29 NPs)</td></tr><tr><td>mAP</td><td>$ \mathrm{mAP}_{\text{base}} $</td><td>$ \mathrm{mAP}_{\text{novel}} $</td><td>HOTA</td><td>$ \mathrm{HOTA}_{\text{common}} $</td><td>$ \mathrm{HOTA}_{\text{uncommon}} $</td><td>mAP</td><td>mAP</td><td>TETA</td><td>HOTA</td><td>HOTA</td></tr><tr><td>GLEE (prompted w/ all NPs at once)</td><td>20.8</td><td>24.0</td><td>18.4</td><td>28.4</td><td>46.4</td><td>24.8</td><td>62.2</td><td>38.7</td><td>18.0</td><td>36.8</td><td>6.8</td></tr><tr><td>GLEE (prompted w/ one NP at a time)</td><td>9.3</td><td>13.9</td><td>5.9</td><td>20.2</td><td>42.9</td><td>15.7</td><td>56.5</td><td>32.4</td><td>14.9</td><td>29.9</td><td>22.9</td></tr><tr><td>LLMDet as detector + SAM 3 Tracker</td><td>15.2</td><td>15.1</td><td>15.3</td><td>33.3</td><td>45.8</td><td>30.8</td><td>31.3</td><td>20.4</td><td>28.9</td><td>24.9</td><td>36.4</td></tr><tr><td>SAM 3 Detector + T-by-D as tracker</td><td>35.9</td><td>32.9</td><td>38.2</td><td>39.7</td><td>59.7</td><td>35.8</td><td>56.5</td><td>55.1</td><td>51.0</td><td>60.5</td><td>35.5</td></tr><tr><td>SAM 3</td><td>36.3</td><td>33.3</td><td>38.5</td><td>44.5</td><td>63.5</td><td>40.7</td><td>57.4</td><td>60.5</td><td>47.2</td><td>60.3</td><td>39.9</td></tr></table>


_(d) Results on public benchmarks_


Table 39 More details on video PCS from a text prompt (open-vocabulary video instance segmentation) on SA-Co/VEval and public benchmarks (# NPs of each benchmark in parentheses). SAM 3 excels in both frame-level detection (pDetA) and cross-frame association (pAssA) and largely outperforms the baselines, especially on benchmarks with a very large number of noun phrases. †: GLEE and LLMDet have not been trained on the SA-Co dataset, so their results should be seen as zero-shot on SA-Co/VEval.


---


![图片 page52-0](images/page052_img01.png)


_(a) offline $\mathcal{I}\& \mathcal{F}$ averaged across 9 datasets_


![图片 page52-2](images/page052_img02.png)


_(b) online $\mathcal{J}\& \mathcal{F}$ averaged across 9 datasets_


<table><tr><td>Method</td><td>EndoVis 2018</td><td>ESD</td><td>LVOSv2</td><td>LV-VIS</td><td>PUMaVOS</td><td>UVO</td><td>VIPSeg</td><td>Virtual KITTI 2</td><td>VOST</td><td>(average)</td></tr><tr><td>SAM + XMem++</td><td>68.9</td><td>88.2</td><td>72.1</td><td>86.4</td><td>60.2</td><td>74.5</td><td>84.2</td><td>63.8</td><td>46.6</td><td>71.7</td></tr><tr><td>SAM + Cutie</td><td>71.8</td><td>87.6</td><td>82.1</td><td>87.1</td><td>59.4</td><td>75.2</td><td>84.4</td><td>70.3</td><td>54.3</td><td>74.7</td></tr><tr><td>SAM 2</td><td>77.0</td><td>90.2</td><td>87.9</td><td>90.3</td><td>68.5</td><td>79.2</td><td>88.3</td><td>74.1</td><td>67.5</td><td>80.3</td></tr><tr><td>SAM 3</td><td>79.1</td><td>91.0</td><td>89.7</td><td>88.8</td><td>68.9</td><td>77.6</td><td>85.9</td><td>75.6</td><td>71.6</td><td>80.9</td></tr></table>


_(c) $\mathcal{I}\& \mathcal{F}$ on each dataset averaged over $1\sim 8$ interacted frames under interactive offline evaluation_


<table><tr><td>Method</td><td>EndoVis 2018</td><td>ESD</td><td>LVOSv2</td><td>LV-VIS</td><td>PUMaVOS</td><td>UVO</td><td>VIPSeg</td><td>Virtual KITTI 2</td><td>VOST</td><td>(average)</td></tr><tr><td>SAM + XMem++</td><td>71.4</td><td>87.8</td><td>72.9</td><td>85.2</td><td>63.7</td><td>74.7</td><td>82.5</td><td>63.9</td><td>52.7</td><td>72.8</td></tr><tr><td>SAM + Cutie</td><td>70.5</td><td>87.3</td><td>80.6</td><td>86.0</td><td>58.9</td><td>75.2</td><td>82.1</td><td>70.4</td><td>54.6</td><td>74.0</td></tr><tr><td>SAM 2</td><td>77.5</td><td>88.9</td><td>87.8</td><td>88.7</td><td>72.7</td><td>78.6</td><td>85.5</td><td>74.0</td><td>65.0</td><td>79.8</td></tr><tr><td>SAM 3</td><td>79.2</td><td>89.6</td><td>89.7</td><td>87.9</td><td>73.1</td><td>77.5</td><td>84.2</td><td>75.6</td><td>67.9</td><td>80.5</td></tr></table>


_(d) $\mathcal{I}\& \mathcal{F}$ on each dataset averaged over $1\sim 8$ interacted frames under interactive online evaluation_


_Figure 21 Interactive video segmentation of SAM 3 vs baselines under offline and online evaluation, following the setup in Ravi et al. (2024) with the same 9 zero-shot datasets and 3 clicks per interacted frame. The $\mathcal{I}\&\mathcal{F}$ under different numbers of interactive frames are shown in (a) and (b), while the $\mathcal{I}\&\mathcal{F}$ on each dataset is shown in (c) and (d)._


---


# F.6.1 Additional Model Outputs for Different Tasks


![图片 page53-1](images/page053_img01.png)


_Gloves chin strap pink flower white t-shirt a flowering plant the white gardening glove_


![图片 page53-3](images/page053_img02.png)


_couch a vase blue lamp countertop wooden table a large window the Glass Vase home furnishings_


![图片 page53-5](images/page053_img03.png)


_- a bowl  
- nightstand  
- the furniture  
- plush comforter  
- bottle of champagne  
- gold and brown pattern_


![图片 page53-7](images/page053_img04.png)


_man   
adult   
a person   
dirt road   
a line of ATVs   
the silver helmet   
a group of individuals_


![图片 page53-9](images/page053_img05.png)


_- spaetzle - the Plate
- cooked steak - a roast beef
- a sliced steak
- rich dark sauce
- a round gray plate
- wooden cutting board_


![图片 page53-11](images/page053_img06.png)


_tomato a tray a bowl the pot the pan a person the dish a metal pot a pink apron cutting board a wooden spoon the head of garlic a white countertop an assortment of fresh vegetables_


![图片 page53-13](images/page053_img07.png)


_Boy a Box the man a drawer the Adult blue shirt an individual the cardboard box_


_son   
the box   
a shorts   
young boy   
the block   
the small box   
short dark hair   
the large cardboard box_


![图片 page53-16](images/page053_img08.png)


_a hill a jacket a mountain men's clothing the large backpack_


![图片 page53-18](images/page053_img09.png)


_fruit blueberry the raspberry_


_the berry blackberry the dragon fruit ball_


![图片 page53-21](images/page053_img10.png)


_man   
ankle   
a white sole   
a white background_


![图片 page53-23](images/page053_img11.png)


_□ box a selection of Whiz products_


![图片 page53-25](images/page053_img12.png)


_tail's head_


![图片 page53-27](images/page053_img13.png)


_the red shirt_


_a clear blue sky_


_Figure 22 Example predictions on the SA-Co/Gold dataset. As these are model outputs, occasional errors may be present._


---


![图片 page54-0](images/page054_img01.png)


_Figure 23 Example video concept segmentation predictions on the SA-Co/VEval SA-V test set (top two rows) and the SA-Co/VEval SmartGlasses test set (bottom two rows)._


![图片 page54-2](images/page054_img02.png)


![图片 page54-3](images/page054_img03.png)


![图片 page54-4](images/page054_img04.png)


_Figure 24 Example predictions on the countbench dataset._


---


# G SAM3 Agent


# G.1 Agent Design


In this section, we introduce SAM 3 Agent, a visual agentic system that turns natural-language segmentation requests into precise masks through dynamically querying a multimodal LLM (MLLM) and SAM 3. Given an input image and a user request, an MLLM acts as a planner/controller: it analyzes the image, devises a step-by-step plan, invokes SAM 3 to generate masks, inspects the results, and finalizes candidate masks. After each action, the agent receives visual and textual feedback describing the updated environment state, enabling it to revise the plan and decide the next action. This perception-action loop continues until the agent is confident it has satisfied the goal (or determines that no valid mask exists), at which point it returns a final set of masks. The resulting pipeline handles queries far more complex than simple noun phrases which require understanding relationships between scene elements and visual common sense.


Each action consists of calling one of several "tools". We define the following four basic tools for the MLLM to call: segment Phrase, examine each mask, select masks and return, and report no mask. Among these four tools, select masks and return and report no mask are return tools, which will trigger a return function and end the current task. The other two functions: segment phrase and examine each mask, are intermediate tools, which will either call the SAM 3 model on a noun phrase or trigger an iterative process for the MLLM to examine each generated mask.


# Tool #1: Segment Phrase


(1) Definition: Use the Segment Anything 3 model to ground all instances of a simple noun phrase by generating segmentation mask(s) that cover those instances on the raw input image. At the same time, all previously generated mask(s) will be deleted and cannot be referred to in future messages.


(2) Use cases: Given a simple, direct, and singular noun phrase (not a referring expression that requires additional understanding/reasoning), segment Phrase will try to locate all object instance(s) on the raw input image that match the simple noun phrase you provided. The tool will also render all of the generated segmentation mask(s) onto the image for you to examine and decide the next step.


(3) Parameters: {"type": "object", "properties": {"text_prompt": {"type": "string", "description": "A short and simple noun phrase, e.g., rope, bird beak, speed monitor, brown handbag, person torso"}}, "required": ["text_prompt"]}


(4) Return type: A new image with differently colored segmentation mask(s) rendered on it, and a text message indicating the number of mask(s) generated by the Segment Anything 3 model for this "text_prompt" only.


# Tool #2: Examine Each Mask


(1) Definition: Use this tool when the segment Phrase tool generates multiple small or overlapping mask(s), making it difficult to distinguish the correct mask(s). Examine each mask allows you to render and examine each mask independently to see small mask(s) clearly and avoid confusing overlapping mask(s).


(2) Use cases: Sometimes there are multiple small mask(s) or overlapping mask(s) rendered on an image, making it difficult to distinguish each mask from others. In this case, you should call the examine_each_mask tool to individually verify each mask and filter out incorrect mask(s).


(3) Parameters: None


(4) Return type: A new image with colored segmentation mask(s) accepted by the examine_each_mask tool, and a text message indicating how many masks were accepted.


---


# Tool #3: Select Masks And Return


(1) Definition: Call this tool to select a subset of or all of the mask(s) rendered on the most recent image as your final output. When calling select_masks_and_return, you cannot select any mask(s) generated by previous rounds other than the most recent round in your "final_answer_masks". You can only use mask(s) from the most recent image in your message history.


(2) Use cases: Given an image with one or more segmentation mask(s) already rendered on it, select_masks_ and_return returns the set of mask(s) you select as the final output.


(3) Parameters: {"type": "object", "properties": {"final_answer�建设s": {"type": "array", "description": "An array of integers representing the selected mask(s) you want to choose as your final output, e.g., [1, 4, 5]"}}}, "required": ["final_answer�建设s"]}


(4) Return type: None (End of Conversation)


# Tool #4: Report No Mask


(1) Definition: Call this tool when you are absolutely sure that there are no object(s) in the image that match or answer the initial user input query.


(2) Use cases: Reporting that the given image does not contain any target object(s) that match or answer the initial user input query.


(3) Parameters: None


(4) Return type: None (End of Conversation)


After each intermediate tool call has been executed, the system will provide the MLLM with the following two pieces of information:


- The user input image with all generated and currently available segmentation masks rendered on it in a Set-of-Marks (Yang et al., 2023) manner. The masks are randomly colored and numbered from 1 to N in decreasing order of SAM 3 confidence scores received at the time of mask generation. The set of currently available masks, combined with the original user input image, defines the environment state of the SAM 3 Agent at the current time step.


- An automatically generated text message stating all changes from the previous environment state (e.g. how many masks have been generated by the segmentPhrase tool, or how many masks were removed by the examine_each_mask tool).


After analyzing the updated image with currently available masks rendered on it (current environment state) in the context of the initial user query (task goal), the MLLM must update its tool-calling plan and generate the next tool call (current action). We allow the MLLM to call each intermediate tool as many times as it needs, before arriving at a final set of segmentation masks on the input image (terminal state) that it is satisfied with.


Empirically, we observe that for especially challenging queries, SAM 3 Agent may produce as many as 60 steps of trial and error before being satisfied with its grounding outcome and calling a return tool. This results in an extremely long environment-state context history with each step containing a new image, pushing both the context limit and multi-image reasoning capability of even current state-of-the-art MLLMs.


To resolve this issue, we propose an aggressive context engineering mechanism that prunes all intermediate trial-and-error states between the initial user text query and the most recent agent call to the segment Phrase tool. We also discard all previously generated masks after each tool call to the segment Phrase tool, which avoids cluttering the rendered Set-of-Marks image with redundant masks. To avoid losing important failure experience from pruned steps, we provide a continuously updated list of all previously used (and discarded) SAM 3 noun phrase prompts for the model to note.


---


![图片 page57-0](images/page057_img01.png)


![图片 page57-1](images/page057_img02.png)


_In the animal kingdom, the males of certain species have distinctive features that set them apart from females. What characteristic in this image shows the animal is male?_


![图片 page57-3](images/page057_img03.png)


![图片 page57-4](images/page057_img04.png)


_the camera lens that is more suitable for photographing nearby objects_


![图片 page57-6](images/page057_img05.png)


![图片 page57-7](images/page057_img06.png)


_the transportation vehicle that does not require electricity or gasoline_


![图片 page57-9](images/page057_img07.png)


![图片 page57-10](images/page057_img08.png)


_If the person in the picture is the owner of the blue car and wants to drive it, which car door is least likely to be the one he enters?_


![图片 page57-12](images/page057_img09.png)


![图片 page57-13](images/page057_img10.png)


_What part of this picture is funny and out of place?_


![图片 page57-15](images/page057_img11.png)


![图片 page57-16](images/page057_img12.png)


_If the police wanted to identify this vehicle, which part of this picture would contain useful information?_


![图片 page57-18](images/page057_img13.png)


![图片 page57-19](images/page057_img14.png)


_Places where bees can suck nectar from_


![图片 page57-21](images/page057_img15.png)


![图片 page57-22](images/page057_img16.png)


_The white apple computer monitor displaying two web pages_


![图片 page57-24](images/page057_img17.png)


![图片 page57-25](images/page057_img18.png)


_What object in this picture might reflect this person's marital status?_


![图片 page57-27](images/page057_img19.png)


![图片 page57-28](images/page057_img20.png)


_I want to take a trip around the world, but I need some transportation to help me cross the oceans. What type of transportation in the picture can fulfill this requirement?_


![图片 page57-30](images/page057_img21.png)


![图片 page57-31](images/page057_img22.png)


_What is the person using to walk the dog?_


![图片 page57-33](images/page057_img23.png)


![图片 page57-34](images/page057_img24.png)


_In historical architecture, buildings often have entrances that consist of large openings with a curved or pointed top. What feature in the picture resembles such an entrance?_


_Figure 25 Successful examples of SAM 3 Agent (Qwen2.5-VL 72B) on the ReasonSeg (Lai et al., 2024) dataset for Reasoning Segmentation and the RefCOCOg (Kazemzadeh et al., 2014) dataset for Referring Expression Segmentation. For each example, see the original input image (left), textual user query (bottom), and final segmentation output (if applicable) from SAM 3 Agent (right)._


---


![图片 page58-0](images/page058_img01.png)


_(a) Original Input Image_


![图片 page58-2](images/page058_img02.png)


_(b) SAM3 Intermediate Masks_


![图片 page58-4](images/page058_img03.png)


_(c) Final Selected Mask_


_Figure 26 Error example of SAM 3 Agent (Qwen2.5-VL 72B) including the original input image and SAM 3 intermediate output masks. The textual user query is "A black object that protects you from the rain, being held by a person in jeans". The agent was able to reason and find a suitable grounding target for calling SAM 3 ("black umbrella"). However, it failed to select the correct mask during final mask selection due to a visual reasoning error by the MLLM._


# G.2 Qualitative Analysis


In this section, we provide success 25 and failure 26 examples of SAM 3 Agent on the ReasonSeg (Lai et al., 2024) and RefCOCOg (Kazemzadeh et al., 2014) datasets, as they are currently the most challenging and widely used reasoning segmentation and referring expression segmentation datasets. We also provide a complete reasoning trace example of SAM 3 Agent, demonstrating how SAM 3 Agent solves complex reasoning segmentation queries by leveraging SAM 3 for precise grounding and MLLM for visual reasoning.


Empirically, we observe that SAM 3 Agent is able to handle free-form textual queries of varying spatial and logical reasoning complexity. It is able to reject queries that do not point to any object in the given image and to generate multiple output masks for queries that match multiple objects in the given image. The MLLM backbone allows SAM 3 Agent to robustly handle queries of varying length and format. It also enables SAM 3 Agent to accurately segment fine-grained concepts that SAM 3 alone struggle with, such as specific aircraft types and fine-grained food categories.


# SAM3 Agent Complete Reasoning Trace Example


```
```cpp
{
    "role": "system",
    "content": {System prompt including tool definitions, agent instructions, and response format}
},
};
{
    "role": "user",
    "content": [
        "type": "image",
        "image": Figure{\ref{fig:agent_mario_1}}
    ],
        "type": "text",
        "text": "the stronger Mario"
    ]
}
},
```


---


```
"text": "<think> There is only one image in the context. The image shows a banner with pixel art characters resembling Mario. The user wants to ground "\the stronger Mario," which appears to be the character on the right side of the equation depicted on the banner. I will use the 'segment Phrase' function with the text prompt "\stronger Mario"\ to attempt to ground the target object.\n\nBefore proceeding, I remind myself that each call to 'segment Phrase' will delete all previously generated masks, so I need to ensure my plan is effective in one go. I will start by attempting to ground "\stronger Mario."\n\n<think>\n<tool>\{"\name": \}\"segment Phrase", "\parameters": \{"text_prompt": "\stronger Mario"\}\}</tool>\n\n\}
\},
\{
		"role": "user",
		"content": [ \{
			"type": "text",
			"text": "SAM3 did not generate any masks for the prompt 'stronger Mario', please call the segment Phrase tool again with a different, perhaps more general and simple noun phrase prompt."
		\}
\},
\{
		"role": "assistant",
		"content": [ \{
			"type": "text",
			"text": "\think": There is still only one image in the context. The previous attempt to ground "\stronger Mario" did not yield any masks. I need to try a simpler and more general noun phrase that still captures the essence of the target. Given the pixel art style, I'll try grounding "\\Mario\\" to see if it captures the character on the right side of the equation.\n\nI remind myself that each call to 'segment Phrase' will delete all previously generated masks, so I need to ensure my plan is effective in one go. I will now attempt to ground "\Mario."\\n\\n</think>\n<tool>\{"\name": "\segment Phrase", "\parameters": \{"text_prompt": "\Mario"\}\}</tool>\n"
	\}
\},
\{
		"role": "user",
		"content": [ \{
			"type": "text",
			"text": "SAM3 did not generate any masks for the prompt 'Mario', please call the segment Phrase tool again with a different, perhaps more general and simple noun phrase prompt."
	\}
\},
\{
		"role": "assistant",
		"content": [ \{
			[type]: "text",
			"text": "\think": There is still only one image in the context. The previous attempts to ground "\stronger Mario" and "\\Mario\\" did not yield any masks. Given the pixel art style and the context of the image, I will try a more general and simple noun phrase related to the character's appearance. I will attempt to ground "\pixel character" to see if it captures the target.\n\nI remind myself that each call to 'segment Phrase' will delete all previously generated masks, so I need to ensure my plan is effective in one go. I will now attempt to ground "\\pixel character."\\n\\n</think>\n<tool>\{"\name": "\segment Phrase", "\parameters": \{"text_prompt": "\pixel character"\}\}</tool>\n"
	\}
\},
\{
		"role": "user",
		"content": [ \{
			[type]: "text",
			"text": "The original user query was: 'the stronger Mario', and the SAM3 prompt you used was: 'pixel character'. SAM3 generated 4 masks for the prompt 'pixel character', which are rendered in the latest image, now please analyze this new image and decide your next action."
	\},
		\}
\},
```


---


![图片 page60-0](images/page060_img01.png)


_(a) Original Input Image_


![图片 page60-2](images/page060_img02.png)


_(b) SAM3 Intermediate Masks_


![图片 page60-4](images/page060_img03.png)


_(c) Final Output Mask_


_Figure 27 Images from the Complete Reasoning Trace Example of SAM 3 Agent (Qwen2.5-VL 72B), including images (a) and (b) from the agent context and image (c) containing the final agent output mask. The textual user query is "the stronger Mario". The SAM 3 Intermediate Masks are generated by SAM 3 using the text prompt "pixel character"._


```
"role": "assistant",  
"content": [  
{  
    "type": "text",  
    "text": "<think> There are two images in the context now: the raw input image and the current output image with four masks rendered. Mask 2 corresponds to the \""stronger Mario\" character, which is the target of the user's query. Masks 1, 3, and 4 are not relevant to the query. \\n\nSince I have identified the correct mask, I will proceed to select it as the final answer. \\n\n</think>\n<tool> {{"name":"select)masks_and_return\"}, {"parameters":"final_answer)masks\": [2]}} </tool>}  
}  
},  
{  
    "role": "user",  
    "content": [  
        {  
            "type": "text",  
            "text": "Final Answer:"  
        },  
        {  
            "type": "image",  
            "image": Figure{\ref{fig:agent_mario_3}}  
        }  
    }  
},
```


# G.3 Full Quantitative Results


We report the full quantitative experiment results of SAM 3 Agent on ReasonSeg in Tab. 40, OmniLabel in Tab. 41, and RefCOCO-Seg in Tab. 42. SAM 3 Agent achieves the best results on both ReasonSeg and OmniLabel in a zero-shot manner, without training on any referring expression segmentation or reasoning segmentation data. SAM 3 Agent also surpasses previous zero-shot state-of-the-art results on RefCOCO+ and RefCOCOg, and is close to best methods that leverage the training datasets. We hypothesize that on RefCOCO, where all masks come from the MSCOCO dataset and each query points to exactly one ground-truth object mask, training-based methods learn the specific dataset annotation biases. We show examples of such annotation biases in the RefCOCO-Seg datasets in Fig. 28. SAM 3 Agent, being a zero-shot method, is unable to exploit these (generally undesirable) biases.


---


<table><tr><td colspan="2">Model</td><td colspan="2">Training</td><td colspan="2">Val Set</td><td colspan="2">Test Set</td><td colspan="2">Test (Short)</td><td colspan="2">Test (Long)</td></tr><tr><td>Name</td><td>Version</td><td>RES</td><td>ReasonSeg</td><td>gIoU</td><td>cIoU</td><td>gIoU</td><td>cIoU</td><td>gIoU</td><td>cIoU</td><td>gIoU</td><td>cIoU</td></tr><tr><td>SEEM</td><td>-</td><td>×</td><td>×</td><td>25.5</td><td>21.2</td><td>24.3</td><td>18.7</td><td>20.1</td><td>11.5</td><td>25.6</td><td>20.8</td></tr><tr><td>Grounded SAM</td><td>-</td><td>×</td><td>×</td><td>26.0</td><td>14.5</td><td>21.3</td><td>16.4</td><td>17.8</td><td>10.8</td><td>22.4</td><td>18.6</td></tr><tr><td>OVSeg</td><td>-</td><td>×</td><td>×</td><td>28.5</td><td>18.6</td><td>26.1</td><td>20.8</td><td>18.0</td><td>15.5</td><td>28.7</td><td>22.5</td></tr><tr><td>GLaMM</td><td>Vicuna 7B</td><td>✓</td><td>×</td><td>47.4</td><td>47.2</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>SAM4MLLM</td><td>Qwen-VL 7B</td><td>✓</td><td>×</td><td>46.7</td><td>48.1</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>SAM4MLLM</td><td>LLaVA1.6 8B</td><td>✓</td><td>×</td><td>58.4</td><td>60.4</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Seg-Zero</td><td>Qwen2.5-VL 3B</td><td>✓</td><td>×</td><td>58.2</td><td>53.1</td><td>56.1</td><td>48.6</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Seg-Zero</td><td>Qwen2.5-VL 7B</td><td>✓</td><td>×</td><td>62.6</td><td>62.0</td><td>57.5</td><td>52.0</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>X-SAM</td><td>Phi3 3.8B</td><td>✓</td><td>✓</td><td>56.6</td><td>32.9</td><td>57.8</td><td>41.0</td><td>47.7</td><td>48.1</td><td>56.0</td><td>40.8</td></tr><tr><td>HyperSeg</td><td>Phi2 3B</td><td>✓</td><td>✓</td><td>59.2</td><td>56.7</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Kang et al.</td><td>LLaVA1.5 7B</td><td>×</td><td>×</td><td>-</td><td>52.4</td><td>-</td><td>48.7</td><td>-</td><td>48.0</td><td>-</td><td>49.1</td></tr><tr><td>Kang et al.</td><td>LLaVA1.5 13B</td><td>×</td><td>×</td><td>-</td><td>60.5</td><td>-</td><td>49.9</td><td>-</td><td>48.7</td><td>-</td><td>51.0</td></tr><tr><td>LISA</td><td>LLaVA 7B</td><td>✓</td><td>×</td><td>44.4</td><td>46.0</td><td>36.8</td><td>34.1</td><td>37.6</td><td>34.4</td><td>36.6</td><td>34.7</td></tr><tr><td>LISA</td><td>LLaVA 7B</td><td>✓</td><td>✓</td><td>52.9</td><td>54.0</td><td>47.3</td><td>34.1</td><td>40.6</td><td>40.6</td><td>49.4</td><td>51.0</td></tr><tr><td>LISA</td><td>LLaVA 13B</td><td>✓</td><td>×</td><td>48.9</td><td>46.9</td><td>44.8</td><td>45.8</td><td>39.9</td><td>43.3</td><td>46.4</td><td>46.5</td></tr><tr><td>LISA</td><td>LLaVA 13B</td><td>✓</td><td>✓</td><td>56.2</td><td>62.9</td><td>51.7</td><td>51.1</td><td>44.3</td><td>42.0</td><td>54.0</td><td>54.3</td></tr><tr><td>LISA</td><td>Llama2 13B</td><td>✓</td><td>✓</td><td>60.0</td><td>67.8</td><td>51.5</td><td>51.3</td><td>43.9</td><td>45.8</td><td>54.0</td><td>53.8</td></tr><tr><td>LISA</td><td>LLaVA1.5 7B</td><td>✓</td><td>×</td><td>53.6</td><td>52.3</td><td>48.8</td><td>47.1</td><td>48.3</td><td>48.8</td><td>49.2</td><td>48.9</td></tr><tr><td>LISA</td><td>LLaVA1.5 7B</td><td>✓</td><td>✓</td><td>61.3</td><td>62.9</td><td>55.6</td><td>56.9</td><td>48.3</td><td>46.3</td><td>57.9</td><td>59.7</td></tr><tr><td>LISA</td><td>LLaVA1.5 13B</td><td>×</td><td>×</td><td>57.7</td><td>60.3</td><td>53.8</td><td>50.8</td><td>50.8</td><td>50.0</td><td>54.7</td><td>50.9</td></tr><tr><td>LISA</td><td>LLaVA1.5 13B</td><td>✓</td><td>✓</td><td>65.0</td><td>72.9</td><td>61.3</td><td>62.2</td><td>55.4</td><td>50.6</td><td>63.2</td><td>65.3</td></tr><tr><td>RSVP</td><td>LLaVA1.6 7B</td><td>×</td><td>×</td><td>59.2</td><td>56.7</td><td>56.9</td><td>50.7</td><td>47.9</td><td>42.0</td><td>58.4</td><td>53.0</td></tr><tr><td>RSVP</td><td>Qwen2-VL 7B</td><td>×</td><td>×</td><td>58.6</td><td>48.5</td><td>56.1</td><td>51.6</td><td>48.5</td><td>44.3</td><td>57.1</td><td>53.0</td></tr><tr><td>RSVP</td><td>Gemini1.5-Flash</td><td>×</td><td>×</td><td>56.9</td><td>49.2</td><td>57.1</td><td>59.2</td><td>47.3</td><td>40.2</td><td>60.2</td><td>65.6</td></tr><tr><td>RSVP</td><td>GPT-4o</td><td>×</td><td>×</td><td>64.7</td><td>63.1</td><td>60.3</td><td>60.0</td><td>55.4</td><td>50.4</td><td>61.9</td><td>62.5</td></tr><tr><td>Gemini Seg</td><td>Gemini2.5 Flash</td><td>?</td><td>?</td><td>28.3</td><td>13.3</td><td>30.6</td><td>9.2</td><td>16.5</td><td>8.0</td><td>35.0</td><td>9.5</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 7B</td><td>×</td><td>×</td><td>62.2</td><td>49.1</td><td>63.0</td><td>53.5</td><td>59.4</td><td>43.5</td><td>64.1</td><td>56.2</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 72B</td><td>×</td><td>×</td><td>74.6</td><td>65.1</td><td>70.8</td><td>64.0</td><td>70.3</td><td>55.7</td><td>71.0</td><td>66.3</td></tr><tr><td>SAM 3 Agent</td><td>Llama4Maverick</td><td>×</td><td>×</td><td>68.5</td><td>61.5</td><td>67.1</td><td>60.9</td><td>66.8</td><td>59.4</td><td>67.2</td><td>61.3</td></tr><tr><td>SAM 3 Agent</td><td>Qwen3-VL 8B Thinking</td><td>×</td><td>×</td><td>68.5</td><td>57.4</td><td>70.2</td><td>67.3</td><td>70.5</td><td>62.5</td><td>70.2</td><td>68.6</td></tr><tr><td>SAM 3 Agent</td><td>Qwen3-VL 235B Thinking</td><td>×</td><td>×</td><td>76.6</td><td>67.0</td><td>73.7</td><td>71.8</td><td>75.1</td><td>67.2</td><td>73.3</td><td>73.0</td></tr><tr><td>SAM 3 Agent</td><td>Gemini2.5 Pro</td><td>×</td><td>×</td><td>77.0</td><td>66.0</td><td>74.0</td><td>72.1</td><td>75.8</td><td>67.5</td><td>73.4</td><td>73.2</td></tr></table>


_Table 40 SAM 3 Agent experiments on the ReasonSeg dataset (Lai et al., 2024) for Reasoning Segmentation. Training-RES indicates whether the model has been fine-tuned on the Referring Expression Segmentation task. Training-ReasonSeg indicates whether the model has been fine-tuned on the ReasonSeg training set. The overall best performances are shown in bold and the best zero-shot performances for models not trained on the ReasonSeg training set are underlined. Notable baselines include: SEEM (Zou et al., 2023), Grounded SAM (Ren et al., 2024b), OVSeg (Liang et al., 2023), GLaMM (Rasheed et al., 2024), SAM4MLLM (Chen et al., 2024b), Seg-Zero (Liu et al., 2025), X-SAM (Wang et al., 2025a), HyperSeg (Wei et al., 2024), (Kang et al., 2025), LISA (Lai et al., 2024), RSVP (Lu et al., 2025) and Gemini-seg (Paul Voigtlaender, Valentin Gabeur and Rohan Doshi, 2025)_


![图片 page61-2](images/page061_img01.png)


![图片 page61-3](images/page061_img02.png)


_atv_


![图片 page61-5](images/page061_img03.png)


![图片 page61-6](images/page061_img04.png)


_carrots_


![图片 page61-8](images/page061_img05.png)


![图片 page61-9](images/page061_img06.png)


_orange piece two to the left of the previous hint_


![图片 page61-11](images/page061_img07.png)


![图片 page61-12](images/page061_img08.png)


_front bowl_


_Figure 28 Examples of annotation bias and ground truth errors from the RefCOCO-Seg datasets (Kazemzadeh et al., 2014; Mao et al., 2016). For each example, see the original dataset ground truth annotation (left image), the textual user query (bottom text), and the SAM 3 Agent (Qwen2.5-VL 72B) final segmentation output (right image). Our error analysis reveals such annotation bias and ground truth errors account for the majority of low-IoU predictions by SAM 3 Agent on the RefCOCO-Seg datasets._


---


_Model_


_2023 Val Set (AP @ IoU=0.50:0.95)_


<table><tr><td>Name</td><td>Version</td><td>AP</td><td>AP-categ</td><td>AP-descr</td><td>AP-descr-pos</td><td>AP-descr-S</td><td>AP-descr-M</td><td>AP-descr-L</td></tr><tr><td>FIBER</td><td>FIBER-B</td><td>25.7</td><td>30.3</td><td>22.3</td><td>34.8</td><td>38.6</td><td>19.5</td><td>12.4</td></tr><tr><td>GLIP</td><td>GLIP-T</td><td>19.3</td><td>23.6</td><td>16.4</td><td>25.8</td><td>29.4</td><td>14.8</td><td>8.2</td></tr><tr><td>GLIP</td><td>GLIP-L</td><td>25.8</td><td>32.9</td><td>21.2</td><td>33.2</td><td>37.7</td><td>18.9</td><td>10.8</td></tr><tr><td>Zhao et al.</td><td>GLIP-T</td><td>22.2</td><td>27.2</td><td>18.8</td><td>29.0</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Zhao et al.</td><td>FIBER-B</td><td>28.1</td><td>32.1</td><td>25.1</td><td>36.5</td><td>-</td><td>-</td><td>-</td></tr><tr><td>DesCo</td><td>GLIP-T</td><td>23.8</td><td>27.4</td><td>21.0</td><td>30.4</td><td>-</td><td>-</td><td>-</td></tr><tr><td>DesCo</td><td>FIBER-B</td><td>29.3</td><td>31.6</td><td>27.3</td><td>37.7</td><td>-</td><td>-</td><td>-</td></tr><tr><td>GLEE</td><td>Lite</td><td>20.3</td><td>37.5</td><td>14.0</td><td>19.1</td><td>23.0</td><td>12.7</td><td>10.0</td></tr><tr><td>GLEE</td><td>Lite-Scale</td><td>22.7</td><td>35.5</td><td>16.7</td><td>22.3</td><td>33.7</td><td>14.3</td><td>10.2</td></tr><tr><td>GLEE</td><td>Plus</td><td>25.4</td><td>46.7</td><td>17.5</td><td>23.9</td><td>28.4</td><td>16.3</td><td>12.5</td></tr><tr><td>GLEE</td><td>Plus-Scale</td><td>27.0</td><td>44.5</td><td>19.4</td><td>25.9</td><td>36.0</td><td>17.2</td><td>12.4</td></tr><tr><td>Real</td><td>Swin-B</td><td>-</td><td>-</td><td>36.5</td><td>52.1</td><td>54.4</td><td>33.2</td><td>25.5</td></tr><tr><td>LED</td><td>Qwen2</td><td>27.9</td><td>33.9</td><td>23.7</td><td>36.2</td><td>-</td><td>-</td><td>-</td></tr><tr><td>LED</td><td>InternLM2-2B</td><td>27.9</td><td>33.4</td><td>23.9</td><td>36.1</td><td>-</td><td>-</td><td>-</td></tr><tr><td>LED</td><td>InternLM2-6B</td><td>26.3</td><td>32.0</td><td>22.4</td><td>34.3</td><td>-</td><td>-</td><td>-</td></tr><tr><td>ROD-MLLM</td><td>Vicuna 7B</td><td>-</td><td>-</td><td>25.3</td><td>30.9</td><td>31.8</td><td>24.5</td><td>21.0</td></tr><tr><td>WSCL</td><td>GLIP-T</td><td>24.3</td><td>23.9</td><td>24.7</td><td>34.4</td><td>39.3</td><td>21.6</td><td>16.4</td></tr><tr><td>WSCL</td><td>FIBER-B</td><td>30.5</td><td>31.6</td><td>29.5</td><td>40.3</td><td>43.7</td><td>26.3</td><td>21.3</td></tr><tr><td>WSCL</td><td>Desco-GLIP</td><td>26.5</td><td>27.1</td><td>25.9</td><td>35.6</td><td>38.1</td><td>23.2</td><td>18.7</td></tr><tr><td>WSCL</td><td>Desco-FIBER</td><td>32.0</td><td>33.1</td><td>30.9</td><td>40.4</td><td>45.2</td><td>27.7</td><td>22.9</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 7B</td><td>38.8</td><td>41.1*</td><td>36.7</td><td>42.8</td><td>52.6</td><td>34.3</td><td>26.6</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 72B</td><td>41.6</td><td>41.1*</td><td>42.0</td><td>52.6</td><td>56.0</td><td>40.4</td><td>33.2</td></tr><tr><td>SAM 3 Agent</td><td>Llama4 Maverick</td><td>36.5</td><td>41.1*</td><td>32.8</td><td>43.0</td><td>43.7</td><td>30.9</td><td>27.5</td></tr><tr><td>SAM 3 Agent</td><td>Qwen3-VL 8B Thinking</td><td>40.9</td><td>41.1*</td><td>40.7</td><td>52.9</td><td>53.3</td><td>39.5</td><td>32.4</td></tr><tr><td>SAM 3 Agent</td><td>Qwen3-VL 235 Thinking</td><td>44.4</td><td>41.1*</td><td>48.4</td><td>59.4</td><td>58.0</td><td>47.8</td><td>41.5</td></tr><tr><td>SAM 3 Agent</td><td>Gemini2.5 Pro</td><td>43.1</td><td>41.1*</td><td>45.3</td><td>63.8</td><td>53.8</td><td>45.1</td><td>37.7</td></tr></table>


_Table 41 SAM 3 Agent experiments on the OmniLabel dataset (Schulter et al., 2023) (val 2023) for generalized referring expression comprehension (box prediction). * indicates predictions generated by the base SAM 3 model without MLLM. The overall best performances are shown in bold. Notable baselines include: FIBER (Dou et al., 2022), GLIP (Li et al., 2022b), (Zhao et al., 2024), DesCo (Li et al., 2023c), GLEE (Wu et al., 2024a), Real (Chen et al., 2025), LED (Zhou et al., 2025), ROD-MLLM (Yin et al., 2025), and WSCL (Park et al., 2024)._


---


<table><tr><td colspan="2">Model</td><td colspan="2">Training</td><td colspan="3">RefCOCO</td><td colspan="3">RefCOCO+</td><td colspan="2">RefCOCOg</td></tr><tr><td>Name</td><td>Version</td><td>RES</td><td>val</td><td>testA</td><td>testB</td><td>val</td><td>testA</td><td>testB</td><td>val (U)</td><td>test (U)</td><td>val (G)</td></tr><tr><td>LISA</td><td>LLaVA 7B</td><td>✓</td><td>74.9</td><td>79.1</td><td>72.3</td><td>65.1</td><td>70.8</td><td>58.1</td><td>67.9</td><td>70.6</td><td>-</td></tr><tr><td>GSVA</td><td>13B</td><td>✓</td><td>79.2</td><td>81.7</td><td>77.1</td><td>70.3</td><td>73.8</td><td>63.6</td><td>75.7</td><td>77.0</td><td>-</td></tr><tr><td>GLaMM</td><td>Vicuna 7B</td><td>✓</td><td>79.5</td><td>83.2</td><td>76.9</td><td>72.6</td><td>78.7</td><td>64.6</td><td>74.2</td><td>74.9</td><td>-</td></tr><tr><td>SAM4MLLM</td><td>LLaVA1.6 7B</td><td>✓</td><td>79.6</td><td>82.8</td><td>76.1</td><td>73.5</td><td>77.8</td><td>65.8</td><td>74.5</td><td>75.6</td><td>-</td></tr><tr><td>SAM4MLLM</td><td>LLaVA1.6 8B</td><td>✓</td><td>79.8</td><td>82.7</td><td>74.7</td><td>74.6</td><td>80.0</td><td>67.2</td><td>75.5</td><td>76.4</td><td>-</td></tr><tr><td>GLEE</td><td>Plus</td><td>✓</td><td>79.5</td><td>-</td><td>-</td><td>68.3</td><td>-</td><td>-</td><td>70.6</td><td>-</td><td>-</td></tr><tr><td>GLEE</td><td>Pro</td><td>✓</td><td>80.0</td><td>-</td><td>-</td><td>69.6</td><td>-</td><td>-</td><td>72.9</td><td>-</td><td>-</td></tr><tr><td>DETRIS</td><td>DETRIS-L</td><td>✓</td><td>81.0</td><td>81.9</td><td>79.0</td><td>75.2</td><td>78.6</td><td>70.2</td><td>74.6</td><td>75.3</td><td>-</td></tr><tr><td>UniLSeg</td><td>UniLSeg-20</td><td>✓</td><td>80.5</td><td>81.8</td><td>78.4</td><td>72.7</td><td>77.0</td><td>67.0</td><td>78.4</td><td>79.5</td><td>-</td></tr><tr><td>UniLSeg</td><td>UniLSeg-100</td><td>✓</td><td>81.7</td><td>83.2</td><td>79.9</td><td>73.2</td><td>78.3</td><td>68.2</td><td>79.3</td><td>80.5</td><td>-</td></tr><tr><td>PSALM</td><td>Phi1.5 1.3B</td><td>✓</td><td>83.6</td><td>84.7</td><td>81.6</td><td>72.9</td><td>75.5</td><td>70.1</td><td>73.8</td><td>74.4</td><td>-</td></tr><tr><td>EVF-SAM</td><td>RC</td><td>✓</td><td>82.1</td><td>83.7</td><td>80.0</td><td>75.2</td><td>78.3</td><td>70.1</td><td>76.8</td><td>77.4</td><td>-</td></tr><tr><td>EVF-SAM</td><td>Extra Data</td><td>✓</td><td>82.4</td><td>84.2</td><td>80.2</td><td>76.5</td><td>80.0</td><td>71.9</td><td>78.2</td><td>78.3</td><td>-</td></tr><tr><td>RICE</td><td>Qwen2.5-7B</td><td>✓</td><td>83.5</td><td>85.3</td><td>81.7</td><td>79.4</td><td>82.8</td><td>75.4</td><td>79.8</td><td>80.4</td><td>-</td></tr><tr><td>MLCD-seg</td><td>Qwen2.5-7B</td><td>✓</td><td>83.6</td><td>85.3</td><td>81.5</td><td>79.4</td><td>82.9</td><td>75.6</td><td>79.7</td><td>80.5</td><td>-</td></tr><tr><td>HyperSeg</td><td>Phi2 2.7B</td><td>✓</td><td>84.8</td><td>85.7</td><td>83.4</td><td>79.0</td><td>83.5</td><td>75.2</td><td>79.4</td><td>78.9</td><td>-</td></tr><tr><td>X-SAM</td><td>Phi3 3.8B</td><td>✓</td><td>85.1</td><td>87.1</td><td>83.4</td><td>78.0</td><td>81.0</td><td>74.4</td><td>83.8</td><td>83.9</td><td>-</td></tr><tr><td>GL-CLIP</td><td>ResNet-50</td><td>×</td><td>32.7</td><td>35.3</td><td>30.1</td><td>37.7</td><td>40.7</td><td>34.9</td><td>41.6</td><td>42.9</td><td>44.0</td></tr><tr><td>GL-CLIP</td><td>ViT-B/32</td><td>×</td><td>32.9</td><td>34.9</td><td>30.1</td><td>38.4</td><td>42.1</td><td>32.7</td><td>42.0</td><td>42.0</td><td>42.7</td></tr><tr><td>CaR</td><td>ViT-B/16</td><td>×</td><td>33.6</td><td>35.4</td><td>30.5</td><td>34.2</td><td>36.0</td><td>31.0</td><td>36.7</td><td>36.6</td><td>36.6</td></tr><tr><td>Ref-Diff</td><td>VAE</td><td>×</td><td>37.2</td><td>38.4</td><td>37.2</td><td>37.3</td><td>40.5</td><td>33.0</td><td>44.0</td><td>44.5</td><td>44.3</td></tr><tr><td>TAS</td><td>ResNet-50</td><td>×</td><td>39.9</td><td>42.9</td><td>35.9</td><td>44.0</td><td>50.6</td><td>36.4</td><td>47.7</td><td>47.4</td><td>48.7</td></tr><tr><td>TAS</td><td>ViT-B/32</td><td>×</td><td>39.8</td><td>41.1</td><td>36.2</td><td>43.6</td><td>49.1</td><td>36.5</td><td>46.6</td><td>46.8</td><td>48.1</td></tr><tr><td>IteRPrimeE</td><td>-</td><td>×</td><td>40.2</td><td>46.5</td><td>33.9</td><td>44.2</td><td>51.6</td><td>35.3</td><td>46.0</td><td>45.1</td><td>45.8</td></tr><tr><td>Pseudo-RIS</td><td>CRIS</td><td>×</td><td>39.8</td><td>44.8</td><td>33.0</td><td>42.2</td><td>46.3</td><td>34.5</td><td>43.7</td><td>43.4</td><td>43.8</td></tr><tr><td>Pseudo-RIS</td><td>ETRIS</td><td>×</td><td>41.1</td><td>48.2</td><td>33.5</td><td>44.3</td><td>51.4</td><td>35.1</td><td>46.0</td><td>46.7</td><td>46.8</td></tr><tr><td>LGD+DINO</td><td>ViT-B/32</td><td>×</td><td>49.5</td><td>54.7</td><td>41.0</td><td>49.6</td><td>58.4</td><td>38.6</td><td>50.3</td><td>51.1</td><td>52.5</td></tr><tr><td>VLM-VG</td><td>ResNet-50</td><td>×</td><td>47.7</td><td>51.8</td><td>44.7</td><td>41.2</td><td>45.9</td><td>34.7</td><td>46.6</td><td>47.1</td><td>-</td></tr><tr><td>VLM-VG</td><td>ResNet-101</td><td>×</td><td>49.9</td><td>53.1</td><td>46.7</td><td>42.7</td><td>47.3</td><td>36.2</td><td>48.0</td><td>48.5</td><td>-</td></tr><tr><td>HybridGL</td><td>ViT-B/32</td><td>×</td><td>49.5</td><td>53.4</td><td>45.2</td><td>43.4</td><td>49.1</td><td>37.2</td><td>51.3</td><td>51.6</td><td>-</td></tr><tr><td>Kang et al.</td><td>LLaVA-1.5 7B</td><td>×</td><td>74.2</td><td>76.5</td><td>70.4</td><td>62.5</td><td>65.2</td><td>56.0</td><td>64.2</td><td>68.1</td><td>-</td></tr><tr><td>Kang et al.</td><td>LLaVA-1.5 13B</td><td>×</td><td>76.1</td><td>78.9</td><td>72.8</td><td>64.1</td><td>67.1</td><td>57.3</td><td>67.7</td><td>69.0</td><td>-</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 7B</td><td>×</td><td>59.4</td><td>64.3</td><td>55.0</td><td>51.4</td><td>57.0</td><td>44.9</td><td>57.2</td><td>58.8</td><td>59.7</td></tr><tr><td>SAM 3 Agent</td><td>Qwen2.5-VL 72B</td><td>×</td><td>71.6</td><td>74.9</td><td>66.0</td><td>64.9</td><td>70.8</td><td>57.9</td><td>68.8</td><td>70.2</td><td>70.4</td></tr><tr><td>SAM 3 Agent</td><td>Llama4 Maverick</td><td>×</td><td>71.7</td><td>76.5</td><td>66.5</td><td>65.1</td><td>71.2</td><td>57.6</td><td>67.7</td><td>68.7</td><td>68.4</td></tr><tr><td>SAM 3 Agent</td><td>Qwen3-VL 8B Thinking</td><td>×</td><td>68.6</td><td>72.3</td><td>63.9</td><td>59.3</td><td>64.5</td><td>55.6</td><td>66.4</td><td>66.9</td><td>67.8</td></tr><tr><td>SAM 3 Agent</td><td>Qwen3-VL 235 Thinking</td><td>×</td><td>73.9</td><td>77.0</td><td>69.7</td><td>66.9</td><td>70.9</td><td>62.2</td><td>72.1</td><td>72.9</td><td>72.5</td></tr><tr><td>SAM 3 Agent</td><td>Gemini2.5 Pro</td><td>×</td><td>75.5</td><td>77.6</td><td>71.0</td><td>67.3</td><td>71.1</td><td>63.4</td><td>73.4</td><td>74.0</td><td>74.6</td></tr></table>


_Table 42 SAM 3 Agent experiments on the RefCOCO / RefCOCO+ / RefCOCOg datasets (Kazemzadeh et al., 2014; Mao et al., 2016) for Referring Expression Segmentation (RES). Training-RES indicates whether the model has been fine-tuned on the RefCOCO/RefCOCO+/RefCOCOg segmentation datasets (notice that nearly all MLLMs were trained on RefCOCO/RefCOCO+/RefCOCOg bbox datasets). The overall best performances are shown in bold and the best zero-shot performances for models not trained on the RES task are underlined. SAM 3 Agent achieves state-of-the-art performance on RefCOCO+/RefCOCOg in the zero-shot setting, and is close to best fine-tuned models. Notable baselines include: LISA (Lai et al., 2024), GSVA (Xia et al., 2024), GLaMM (Rasheed et al., 2024), SAM4MLLM (Chen et al., 2024b), GLEE (Wu et al., 2024a), DETRIS (Huang et al., 2025), UniLSeg (Liu et al., 2024b), PSALM (Zhang et al., 2024c), EVF-SAM (Zhang et al., 2024b), RICE (Xie et al., 2025), MLCD-seg (An et al., 2024), HyperSeg (Wei et al., 2024), X-SAM (Wang et al., 2025a), GL-CLIP (Yu et al., 2023b), CaR (Sun et al., 2024), Ref-Diff (Ni et al., 2023), TAS (Suo et al., 2023), IteRPrimeE (Wang et al., 2025c), Pseudo-RIS (Yu et al., 2024), LGD+DINO (Li et al., 2025), VLM-VG (Wang et al., 2025b), HybridGL (Liu & Li, 2025), and (Kang et al., 2025)._


---


# H Model and annotation cards


# H.1 Data annotation card


# Task Formulation.


1. At a high level, what are the subjective aspects of your task? There is ambiguity in the task. Annotators may have multiple valid interpretations of what should be masked for a given phrase. E.g. If a person is wearing a backpack should a mask for the phrase 'person' include the backpack? If the person is standing next to a painting that contains a person, should that person be masked too? We accept this ambiguity in the task, and in the gold set we use reviews from three different annotators to help capture multiple interpretations.


2. What assumptions do you make about annotators? Annotators worked full time on the annotation task, which allowed for frequently sharing feedback that led to improved annotations. Annotators were proficient in English and completed adequate research to understand concepts that they were not familiar with. This research allowed us to annotate more fine-grained or specific concepts, like car brands. Annotators were detail-oriented looking for all possible instances of the phrase in the image. We focused more on annotation quality over annotation speed to allow annotators to carefully look for all instances.


3. How did you choose the specific wording of your task instructions? What steps, if any, were taken to verify the clarity of task instructions and wording for annotators? We provided detailed guidelines that included numerous examples of correct and incorrect annotations. We broke down the task into different scenarios, and the expected outcome for each scenario. We made frequent guideline updates to handle ambiguities and address new corner cases surfaced by the vendor. The vendor trained the raters on the updated guidelines, and QA'ed the annotators to ensure adoption. We maintained a log of vendor-posed questions and answers around guideline clarifications. We met with the vendor weekly to provide feedback on annotation quality and surface common mistake patterns. This decreased repeat errors, and increased the quality of vendor QA's.


4. What, if any, risks did your task pose for annotators and were they informed of the risks prior to engagement with the task? Annotators were instructed to reject objectionable content and flag phrases that were harmful or offensive to ground.


5. What are the precise instructions that were provided to annotators? The instructions varied for each of the annotation tasks. For images, we had annotators work on three separate tasks. 1) Verify the quality of masks for a given phrase in an image 2) Check if masks were exhaustively annotated in an image for a given phrase and 3) Add any missing masks and correct mask annotations such that all instances of the phrase were masked in the image. For video, there were two separate tasks. 1) Exhaustively annotate all instances of the phrase in the video and 2) Verify whether all instances are annotated with high-quality masklets in the video.


# Selecting Annotations.


1. Are there certain perspectives that should be privileged? If so, how did you seek these perspectives out? All annotators had a minimum of B-2 English proficiency. Annotators had previous segmentation experience. Annotators researched fine-grained concepts that they were unfamiliar with.


2. Are there certain perspectives that would be harmful to include? If so, how did you screen these perspectives out? No.


3. Were sociodemographic characteristics used to select annotators for your task? If so, please detail the process. No.


4. If you have any aggregated socio-demographic statistics about your annotator pool, please describe. Do you have reason to believe that sociodemographic characteristics of annotators may have impacted how they annotated the data? Why or why not? We worked with annotators based in APAC and EMEA. The sociodemographic characteristics of annotators may have some impact on the annotated data. Across different regions, words can differ in their meanings and the same concept may look different across regions.


5. Consider the intended context of use of the dataset and the individuals and communities that may be impacted by a model trained on this dataset. Are these communities represented in your annotator pool? Our annotator pool does not represent all communities that will use the SAM 3. Annotators researched concepts they were unfamiliar with. When annotators were unsure, they researched concepts to better understand different visual representations of the concept. If annotators were still unsure, they rejected the job as unsure in order to make sure that our annotations only contained confident responses. Annotators flagged concepts that were harmful in context of the image or the video.


# Platform and Infrastructure Choices.


1. What annotation platform did you utilize? At a high level, what considerations informed your decision to choose this platform? Did the chosen platform sufficiently meet the requirements you outlined for annotator pools? Are any aspects not covered? We used an internal annotation platform.


2. What, if any, communication channels did your chosen platform offer to facilitate communication with annotators? How did this channel of communication influence the annotation process and/or resulting annotations?


The research team QA'ed the vendors' quality team and annotators, shared feedback and met weekly with the vendor to align on the guidelines, clarify ambiguities and surface common mistake patterns. The research team maintained a spreadsheet where they answered the vendor's questions requiring the desired annotations for specific jobs that were corner cases or ambiguous. The guidelines were frequently updated to include new corner cases surfaced. These processes helped align the vendor to our desired output, which allowed the vendor to more effectively QA the annotators and provide per-annotator feedback which decreased repeat errors. A chat thread was also maintained between the research team and vendor.


3. How much were annotators compensated? Did you consider any particular pay standards, when determining their compensation? If so, please describe. The annotators were compensated with an hourly wage set by the vendor.


# Dataset Analysis and Evaluation.


---


1. How do you define the quality of annotations in your context, and how did you assess the quality in the dataset you constructed?


Annotation quality was based more on ensuring completeness and validity of the annotator's interpretation. We defined quality across three axes: 1) mask quality (e.g. masks should not have holes or missing pieces) 2) mask concept correctness (e.g there should not be a mask around a dog when the concept is "cat") 3) mask exhaustivity (e.g. all instances including small background instances should be masked). We set a high bar for quality, and aligned with the vendor on the requirements of a correct annotation. For each task, annotators underwent a 2-day training session led by the vendor, followed by annotating jobs from a training queue. They were only eligible to move into the production annotation queues after the vendor QA or research team reviewed their annotations and approved of the quality. The vendor QA team continuously reviewed production annotations, covering covering $10\%$ avg of all annotations, ranging from $5\% - 20\%$ dependent on task complexity across the duration of the program. The research team manually reviewed small subsets of the production annotations and shared feedback weekly.


We ensure high data quality by letting annotators reject low confidence or vague annotations. Annotators were asked to research concepts they were unfamiliar with or unsure if they were present in the media. If, after researching, annotators were still unsure or the concept was considered vague (e.g. "sunlight") the annotators were instructed to reject the job as unsure.


In addition, all video annotations are manually reviewed and only those that meet the criteria across all three axes are accepted for use in training and evaluation, ensuring that the dataset contains only high-quality, validated data.


2. Have you conducted any analysis on disagreement patterns? If so, what analyses did you use and what were the major findings? Did you analyze potential sources of disagreement? The PCS task is inherently ambiguous and the sources of annotation disagreement are explained in §2. We further demonstrate the effect of this ambiguity in §E.4. For SA-Co/Gold, we stored all 3 annotator responses in order to capture the ambiguity in the task.


3. How do the individual annotator responses relate to the final labels released in the dataset? For images, each image, phrase pair underwent three annotation tasks: 1) mask quality verification of SAM-3 PL masks 2) exhaustivity verification of accepted masks for the phrase and 3) manually adding missing masks until all instances of the concept were masked. For video annotations, individual annotator responses are validated and only annotations that meet the required quality criterion are accepted. The final labels released in the video subset consist exclusively of these accepted annotation.


For SA-Co/Gold: Gold subsets were 3x multi-reviewed. For mask acceptance, all three annotators had to accept as a mask as high quality for it to be included in the dataset. Image, phrase pairs where at least one of the raters marked the phrase as non-exhaustively annotated in step two were sent to manual annotation. For manual annotation, all three of the annotators responses were saved as separate versions of the annotations for the give image, phrase pair. This helped capture the natural ambiguity in the task and different valid interpretations of a concept.


For SA-Co/Silver: Silver subsets were 1x reviewed. The first annotator's accepted masks were given to the exhaustivity annotator. If the exhaustivity annotator marked the phrase, set of masks as exhaustive, this was the


# Dataset Release and Maintenance.


1. Do you have reason to believe the annotations in this dataset may change over time? Do you plan to update your dataset? The dataset contains annotations for public images or external public datasets. Some images or datasets may become unavailable over time.


2. Are there any conditions or definitions that, if changed, could impact the utility of your dataset? Concepts may change visually - for example masks annotated with 'smartphone' may no longer represent a modern day smartphone. New phrases or types of items will not be represented.


3. Will you attempt to track, impose limitations on, or otherwise influence how your dataset is used? If so, how? Our benchmark is for model evaluation, and should not be used for training.


4. Were annotators informed about how the data is externalized? If changes to the dataset are made, will they be informed? No.


5. Is there a process by which annotators can later choose to withdraw their data from the dataset? If so, please detail. No.


# H.2 Model card


---


# Model Overview


<table><tr><td>Name</td><td>SAM 3 (Segment Anything Model 3)</td></tr><tr><td>Version</td><td>1.0</td></tr><tr><td>Date</td><td>2025</td></tr><tr><td>Organization</td><td>Meta SAM Team</td></tr><tr><td>Mode type</td><td>Promptsable segmentation model</td></tr><tr><td>Architecture</td><td>See Section 3</td></tr><tr><td>Repository</td><td>https://github.com/facebookresearch/sam3</td></tr><tr><td>License</td><td>SAM license (detailed in the SAM 3 repository)</td></tr><tr><td colspan="2">Intended Use</td></tr><tr><td>Primary intended users</td><td>SAM 3 was designed as a model for promtable concept segmentation (PCS) and promptable visual segmentation in images and videos. The model was primarily developed for research use cases. SAM 3 is released under the license detailed in the SAM 3 repository.</td></tr><tr><td>Out-of-scope use cases</td><td>SAM 3 does not support complex text queries or queries including referring expressions. See license for restrictions.</td></tr><tr><td>Caveats and recommendations</td><td>SAM 3 generally performs well in zero-shot settings, but will benefit from fine-tuning for niche domains. In video, the cost of inference scales linearly with the number of objects being tracked. See §B for detailed discussion around limitations.</td></tr><tr><td colspan="2">Relevant Factors</td></tr><tr><td>Groups</td><td>SAM 3 was developed for open world vocabulary. See §B for limitations on text prompts.</td></tr><tr><td>Instrumentation and environment</td><td>SAM 3 was trained on a diverse set of media pools that vary in instrumentation (e.g. medical instruments, underwater imagery, camera traps) and settings (e.g. everyday scenes, driving, robotics, wildlife).</td></tr><tr><td colspan="2">Metrics</td></tr><tr><td rowspan="2">Evaluation Data</td><td>We evaluate the performance of SAM 3 using metrics tailored to the specific task. For image tasks, we use the following metrics:· PCS in Images with Text: cgF1, IL MCC, pmF1, AP. See §E.3 for details.· PCS in Images with Exemplars: cgF1, AP. See §E.3 for details.· Instance Segmentation, Box Detection, Few Shot Box Detection: cgF1, AP· Semantic and Interactive Instance Segmentation (SAM 1 task): mIoU· Counting: MAE, AccuracyFor video tasks, we use the following metrics:· PCS in Videos: cgF1, pHOTA, mAP. See §F.5 for details.· VOS: J&amp;F, G· Promptable video segmentation (SAM 2 task): offline average J&amp;F, online average J&amp;F</td></tr><tr><td>For PROMPTABLE concept segmentation, SAM 3 is evaluated using the SA-Co benchmark which is composed of:· SA-Co/Gold· SA-Co/Bronze· SA-Co/VEval· SA-Co/Silver· SA-Co/BioEach of these subsets is composed of a series of data sources, see §E.2 for details. The SA-Co benchmark annotations are released publicly at https://github.com/facebookresearch/sam3</td></tr><tr><td>Training Data</td><td>SAM 3 is trained using the following datasets:· SA-Co/HQ· SA-Co/SYN· SA-Co/VIDEO-EXT· SA-Co/EXT, SA-Co/VIDEO, SA-Co/SYN, SA-Co/HQ, SA-Co/VIDEO-EXT are composed of multiple data sources, see Table 26 in §E.1 for the list of data sources for subset. See Table 18 in §C.4.1 for details as to which datasets are used during which training stages.</td></tr><tr><td colspan="2">Ethical Considerations</td></tr><tr><td>DataCost and impact of compute</td><td rowspan="3">See §H.1 for data annotation cardThe released SAM 3 was trained on 172k A100 GPU hours and 86k H200 GPU hours. This corresponds to an estimated 142k-176k kWh and emissions between 65.3 and 77.6 metric tons of CO2e (Patterson et al., 2021; Lacoste et al., 2019).The emissions from training the released SAM 3 model are equivalent to 166k miles driven by an average gasoline-powered passenger vehicle (Agency, 2022).The behavior of SAM 3 is undefined for opinionated or subjective queries (e.g. magnificent painting) and should not be used to make subjective judgments about objects or people. Users should evaluate the safety of SAM 3 tailored to their specific use case.We implore users to use their best judgment.</td></tr><tr><td>Risks and harms</td></tr><tr><td>Use cases</td></tr></table>


_Table 43 Model card for SAM 3 following the structure in (Mitchell et al., 2019)_


---


# References


United States Environmental Protection Agency. Greenhouse gas equivalencies calculator, 2022. URL https://www.epa.gov/energy/greenhouse-gas-equivalencies-calculator.


Xiang An, Kaicheng Yang, Xiangzi Dai, Ziyong Feng, and Jiankang Deng. Multi-label cluster discrimination for visual representation learning. In European Conference on Computer Vision, pp. 428-444. Springer, 2024.


Ali Athar, Jonathon Luiten, Paul Voigtlaender, Tarasha Khurana, Achal Dave, Bastian Leibe, and Deva Ramanan. Burst: A benchmark for unifying object recognition, segmentation and tracking in video. In Proceedings of the IEEE/CVF winter conference on applications of computer vision, pp. 1674-1683, 2023.


Hexin Bai, Wensheng Cheng, Peng Chu, Juehuan Liu, Kai Zhang, and Haibin Ling. Gmot-40: A benchmark for generic multiple object tracking. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 6719-6728, 2021.


Kevin Barnard, Elaine Liu, Kristine Walz, Brian Schlining, Nancy Jacobsen Stout, and Lonny Lundsten. DeepSea MOT: A benchmark dataset for multi-object tracking on deep-sea video. arXiv preprint arXiv:2509.03499, 2025. doi: 10.48550/arXiv.2509.03499.


Philipp Bergmann, Tim Meinhardt, and Laura Leal-Taixe. Tracking without bells and whistles. In Proceedings of the IEEE/CVF international conference on computer vision, pp. 941-951, 2019.


Alex Bewley, Zongyuan Ge, Lionel Ott, Fabio Ramos, and Ben Upcroft. Simple online and realtime tracking. In 2016 IEEE international conference on image processing (ICIP), pp. 3464-3468. IEEE, 2016.


Lucas Beyer, Andreas Steiner, André Susano Pinto, Alexander Kolesnikov, Xiao Wang, Daniel Salz, Maxim Neumann, Ibrahim Alabdulmohsin, Michael Tschannen, Emanuele Bugliarello, et al. PaliGemma: A versatile 3B VLM for transfer. arXiv preprint arXiv:2407.07726, 2024.


Alexey Bochkovskiy, Chien-Yao Wang, and Hong-Yuan Mark Liao. Yolov4: Optimal speed and accuracy of object detection, 2020. URL https://arxiv.org/abs/2004.10934.


Daniel Bolya, Chaitanya Ryali, Judy Hoffman, and Christoph Feichtenhofer. Window attention is bugged: How not to interpolate position embeddings. In International Conference on Learning Representations, 2024.


Daniel Bolya, Po-Yao Huang, Peize Sun, Jang Hyun Cho, Andrea Madotto, Chen Wei, Tengyu Ma, Jiale Zhi, Jathushan Rajasegaran, Hanoona Rasheed, Junke Wang, Marco Monteiro, Hu Xu, Shiyu Dong, Nikhila Ravi, Daniel Li, Piotr Dollár, and Christoph Feichtenhofer. Perception encoder: The best visual embeddings are not at the output of the network. arXiv:2504.13181, 2025.


Zhi Cai, Songtao Liu, Guodong Wang, Zeming Li, Zheng Ge, Xiangyu Zhang, and Di Huang. Align-detr: Enhancing end-to-end object detection with aligned loss. In 35th British Machine Vision Conference 2024, BMVC 2024, Glasgow, UK, November 25-28, 2024. BMVA, 2024. URL https://papers.bmvc2024.org/0211.pdf.


Jinkun Cao, Jiangmiao Pang, Xinshuo Weng, Rawal Khirodkar, and Kris Kitani. Observation-centric sort: Rethinking sort for robust multi-object tracking. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 9686-9696, 2023.


Nicolas Carion, Francisco Massa, Gabriel Synnaeve, Nicolas Usunier, Alexander Kirillov, and Sergey Zagoruyko. End-to-end object detection with transformers. In European conference on computer vision, pp. 213-229. Springer, 2020.


Qiang Chen, Xiangbo Su, Xinyu Zhang, Jian Wang, Jiahui Chen, Yunpeng Shen, Chuchu Han, Ziliang Chen, Weixiang Xu, Fanrong Li, et al. Lw-detr: A transformer replacement to yolo for real-time detection. arXiv preprint arXiv:2406.03459, 2024a.


Yi-Chia Chen, Wei-Hua Li, Cheng Sun, Yu-Chiang Frank Wang, and Chu-Song Chen. Sam4mllm: Enhance multi-modal large language model for referring expression segmentation. In European Conference on Computer Vision, pp. 323-340. Springer, 2024b.


Yuming Chen, Jiangyan Feng, Haodong Zhang, Lijun Gong, Feng Zhu, Rui Zhao, Qibin Hou, Ming-Ming Cheng, and Yibing Song. Re-aligning language to visual objects with an agentic workflow. In International Conference on Learning Representations, 2025.


Bowen Cheng, Alexander G. Schwing, and Alexander Kirillov. Per-pixel classification is not all you need for semantic segmentation. In NeurIPS, 2021.


---


Jang Hyun Cho, Andrea Madotto, Effrosyni Mavroudi, Triantafyllos Afouras, Tushar Nagarajan, Muhammad Maaz, Yale Song, Tengyu Ma, Shuming Hu, Hanoona Rasheed, Peize Sun, Po-Yao Huang, Daniel Bolya, Suyog Jain, Miguel Martin, Huiyu Wang, Nikhila Ravi, Shashank Jain, Temmy Stark, Shane Moon, Babak Damavandi, Vivian Lee, Andrew Westbury, Salman Khan, Philipp Krahenbuhl, Piotr Dlár, Lorenzo Torresani, Kristen Grauman, and Christoph Feichtenhofer. Perceptionlm: Open-access data and models for detailed visual understanding. arXiv:2504.13180, 2025.


Kevin Clark, Minh-Thang Luong, Quoc V Le, and Christopher D Manning. ELECTRA: Pre-training text encoders as discriminators rather than generators. In ICLR, 2020.


Gheorghe Comanici, Eric Bieber, Mike Schaekermann, Ice Pasupat, Noveen Sachdeva, Inderjit Dhillon, Marcel Blistein, Ori Ram, Dan Zhang, Evan Rosen, et al. Gemini 2.5: Pushing the frontier with advanced reasoning, multimodality, long context, and next generation agentic capabilities. arXiv preprint arXiv:2507.06261, 2025.


Marius Cordts, Mohamed Omran, Sebastian Ramos, Timo Rehfeld, Markus Enzweiler, Rodrigo Benenson, Uwe Franke, Stefan Roth, and Bernt Schiele. The cityscapes dataset for semantic urban scene understanding. In Proc. of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2016.


Achal Dave, Piotr Dlár, Deva Ramanan, Alexander Kirillov, and Ross Girshick. Evaluating large-vocabulary object detectors: The devil is in the details, 2022. URL https://arxiv.org/abs/2102.01066.


Matt Deitke, Christopher Clark, Sangho Lee, Rohun Tripathi, Yue Yang, Jae Sung Park, Mohammadreza Salehi, Niklas Muennighoff, Kyle Lo, Luca Soldaini, et al. Molmo and pixmo: Open weights and open data for state-of-the-art vision-language models. In Proceedings of the Computer Vision and Pattern Recognition Conference, pp. 91-104, 2025.


Henghui Ding, Kaining Ying, Chang Liu, Shuting He, Xudong Jiang, Yu-Gang Jiang, Philip HS Torr, and Song Bai. Mosev2: A more challenging dataset for video object segmentation in complex scenes. arXiv preprint arXiv:2508.05630, 2025.


Kexin Ding, Mu Zhou, He Wang, Olivier Gevaert, Dimitris Metaxas, and Shaoting Zhang. A large-scale synthetic pathological dataset for deep learning-enabled segmentation of breast cancer. *Scientific Data*, 10(1):231, 2023.


Shuangrui Ding, Rui Qian, Xiaoyi Dong, Pan Zhang, Yuhang Zang, Yuhang Cao, Yuwei Guo, Dahua Lin, and Jiaqi Wang. Sam2long: Enhancing sam 2 for long video segmentation with a training-free memory tree. arXiv preprint arXiv:2410.16268, 2024.


Zheng Ding, Jieke Wang, and Zhuowen Tu. Open-vocabulary universal image segmentation with maskclip. arXiv preprint arXiv:2208.08984, 2022.


Zi-Yi Dou, Aishwarya Kamath, Zhe Gan, Pengchuan Zhang, Jianfeng Wang, Linjie Li, Zicheng Liu, Ce Liu, Yann LeCun, Nanyun Peng, Jianfeng Gao, and Lijuan Wang. Coarse-to-fine vision-language pre-training with fusion in the backbone, 2022. URL https://arxiv.org/abs/2206.07643.


Abhimanyu Dubey, Abhinav Jauhri, Abhinav Pandey, Abhishek Kadian, Ahmad Al-Dahle, Aiesha Letman, Akhil Mathur, Alan Schelten, Amy Yang, Angela Fan, et al. The llama 3 herd of models. arXiv e-prints, pp. arXiv-2407, 2024.


Christoffer Edlund, Timothy R Jackson, Nabeel Khalid, Nicola Bevan, Timothy Dale, Andreas Dengel, Sheraz Ahmed, Johan Trygg, and Rickard Sjögren. Livecell—a large-scale dataset for label-free live cell segmentation. Nature methods, 18(9):1038-1045, 2021.


Christoph Feichtenhofer, Axel Pinz, and Andrew Zisserman. Detect to track and track to detect. In Proceedings of the IEEE international conference on computer vision, pp. 3038-3046, 2017.


FFmpeg developers. FFmpeg. https://ffmpeg.org/.


Shenghao Fu, Qize Yang, Qijie Mo, Junkai Yan, Xihan Wei, Jingke Meng, Xiaohua Xie, and Wei-Shi Zheng. Llmdet: Learning strong open-vocabulary object detectors under the supervision of large language models. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, 2025.


Jevgenij Gamper, Navig Alemi Koohbanani, Ksenija Benes, Ali Khuram, and Nasir Rajpoot. Pannuke: an open pan-cancer histology dataset for nuclei instance segmentation and classification. In European Congress on Digital Pathology, pp. 11-19. Springer, 2019.


---


Jevgenij Gamper, Navid Alemi Koohbanani, Simon Graham, Mostafa Jahanifar, Syed Ali Khurram, Ayesha Azam, Katherine Hewitt, and Nasir Rajpoot. Pannuke dataset extension, insights and baselines. arXiv preprint arXiv:2003.10778, 2020.


Kristen Grauman, Andrew Westbury, Eugene Byrne, Zachary Chavis, Antonino Furnari, Rohit Girdhar, Jackson Hamburger, Hao Jiang, Miao Liu, Xingyu Liu, Miguel Martin, Tushar Nagarajan, Iija Radosavovic, Santhosh Kumar Ramakrishnan, Fiona Ryan, Jayant Sharma, Michael Wray, Mengmeng Xu, Eric Zhongcong Xu, Chen Zhao, Siddhant Bansal, Dhruv Batra, Vincent Cartillier, Sean Crane, Tien Do, Morrie Doulaty, Akshay Erapalli, Christoph Feichtenhofer, Adriano Fragomeni, Qichen Fu, Christian Fuegen, Abrham Gebreselasie, Cristina Gonzalez, James Hillis, Xuhua Huang, Yifei Huang, Wenqi Jia, Weslie Khoo, Jachym Kolar, Satwik Kottur, Anurag Kumar, Federico Landini, Chao Li, Yanghao Li, Zhenqiang Li, Karttikeya Mangalam, Raghava Modhugu, Jonathan Munro, Tullie Murrell, Takumi Nishiyasu, Will Price, Paola Ruiz Puentes, Merey Ramazanova, Leda Sari, Kiran Somasundaram, Audrey Southerland, Yusuke Sugano, Ruijie Tao, Minh Vo, Yuchen Wang, Xindi Wu, Takuma Yagi, Yunyi Zhu, Pablo Arbelaez, David Crandall, Dima Damen, Giovanni Maria Farinella, Bernard Ghanem, Vamsi Krishna Ithapu, C. V. Jawahar, Hanbyul Joo, Kris Kitani, Haizhou Li, Richard Newcombe, Aude Oliva, Hyun Soo Park, James M. Rehg, Yoichi Sato, Jianbo Shi, Mike Zheng Shou, Antonio Torralba, Lorenzo Torresani, Mingfei Yan, and Jitendra Malik. Ego4d: Around the World in 3,000 Hours of Egocentric Video. In IEEE/CVF Computer Vision and Pattern Recognition (CVPR), 2022.


Xiuye Gu, Tsung-Yi Lin, Weicheng Kuo, and Yin Cui. Open-vocabulary object detection via vision and language knowledge distillation. arXiv preprint arXiv:2104.13921, 2021.


Agrim Gupta, Piotr Dollar, and Ross Girshick. Lvis: A dataset for large vocabulary instance segmentation. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 5356-5364, 2019.


Kaiming He, Xinlei Chen, Saining Xie, Yanghao Li, Piotr Dolkar, and Ross Girshick. Masked autoencoders are scalable vision learners. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 16000-16009, 2022.


Byeongho Heo, Song Park, Dongyoon Han, and Sangdoo Yun. Rotary position embedding for vision transformer. arXiv preprint arXiv:2403.13298, 2024.


Lingyi Hong, Zhongying Liu, Wenchao Chen, Chenzhi Tan, Yuang Feng, Xinyu Zhou, Pinxue Guo, Jinglun Li, Zhaoyu Chen, Shuyong Gao, et al. Lvos: A benchmark for large-scale long-term video object segmentation. arXiv preprint arXiv:2404.19326, 2024.


Matthew Honnibal, Ines Montani, Sofie Van Landeghem, and Adriane Boyd. spaCy: Industrial-strength Natural Language Processing in Python. 2020. doi: 10.5281/zenodo.1212303.


Grant Van Horn, Oisin Mac Aodha, Yang Song, Alexander Shepard, Hartwig Adam, Pietro Perona, and Serge J. Belongie. The inaturalist challenge 2017 dataset. CoRR, abs/1707.06642, 2017. URL http://arxiv.org/abs/1707.06642.


Zhengdong Hu, Yifan Sun, Jingdong Wang, and Yi Yang. DAC-DETR: Divide the attention layers and conquer. In Advances in Neural Information Processing Systems, 2023.


Jiaqi Huang, Zunnan Xu, Ting Liu, Yong Liu, Haonan Han, Kehong Yuan, and Xiu Li. Densely connected parameter-efficient tuning for referring image segmentation. In Proceedings of the AAAI Conference on Artificial Intelligence, pp. 3653-3661, 2025.


Ding Jia, Yuhui Yuan, Haodi He, Xiaopei Wu, Haojun Yu, Weihong Lin, Lei Sun, Chao Zhang, and Han Hu. Detrs with hybrid matching. arXiv preprint arXiv:2207.13080, 2022.


Menglin Jia, Mengyun Shi, Mikhail Sirotenko, Yin Cui, Claire Cardie, Bharath Hariharan, Hartwig Adam, and Serge J. Belongie. Fashionpedia: Ontology, segmentation, and an attribute localization dataset. CoRR, abs/2004.12276, 2020. URL https://arxiv.org/abs/2004.12276.


Junjie Jiang, Zelin Wang, Manqi Zhao, Yin Li, and DongSheng Jiang. Sam2mot: A novel paradigm of multi-object tracking by segmentation. arXiv preprint arXiv:2504.04519, 2025.


Qing Jiang, Feng Li, Zhaoyang Zeng, Tianhe Ren, Shilong Liu, and Lei Zhang. T-rex2: Towards generic object detection via text-visual prompt synergy. In European Conference on Computer Vision, pp. 38-57. Springer, 2024.


Arne Hoffhues Jonathon Luiten. Trackeval. https://github.com/JonathonLuiten/TrackEval, 2020.


Aishwarya Kamath, Mannat Singh, Yann LeCun, Gabriel Synnaeve, Ishan Misra, and Nicolas Carion. Mdetr-modulated detection for end-to-end multi-modal understanding. In Proceedings of the IEEE/CVF international conference on computer vision, pp. 1780-1790, 2021.


---


Seil Kang, Jinyeong Kim, Junhyeok Kim, and Seong Jae Hwang. Your large vision-language model only needs a few attention heads for visual grounding. In Proceedings of the Computer Vision and Pattern Recognition Conference, pp. 9339-9350, 2025.


Kakani Katija, Eric C. Orenstein, Brian Schlining, Lonny Lundsten, Kevin Barnard, Giovanna Sainz, Oceane Boulais, Benjamin G. Woodward, and Katy Croff Bell. Fathomnet: A global underwater image training set for enabling artificial intelligence in the ocean. CoRR, abs/2109.14646, 2021. URL https://arxiv.org/abs/2109.14646.


Sahar Kazemzadeh, Vicente Ordonez, Mark Matten, and Tamara Berg. Referitgame: Referring to objects in photographs of natural scenes. In Proceedings of the 2014 conference on empirical methods in natural language processing (EMNLP), pp. 787-798, 2014.


Lei Ke, Henghui Ding, Martin Danelljan, Yu-Wing Tai, Chi-Keung Tang, and Fisher Yu. Video mask transfiner for high-quality video instance segmentation. In European Conference on Computer Vision, pp. 731-747. Springer, 2022.


Alexander Khazatsky, Karl Pertsch, Suraj Nair, Ashwin Balakrishna, Sudeep Dasari, Siddharth Karamcheti, Soroush Nasiriany, Mohan Kumar Srirama, Lawrence Yunliang Chen, Kirsty Ellis, Peter David Fagan, Joey Hejna, Masha Itkina, Marion Lepert, Yecheng Jason Ma, Patrick Tree Miller, Jimmy Wu, Suneel Belkhale, Shivin Dass, Huy Ha, Arhan Jain, Abraham Lee, Youngwoon Lee, Marius Memmel, Sungjae Park, Iija Radosavovic, Kaiyuan Wang, Albert Zhan, Kevin Black, Cheng Chi, Kyle Beltran Hatch, Shan Lin, Jingpei Lu, Jean Mercat, Abdul Rehman, Pannag R Sanketi, Archit Sharma, Cody Simpson, Quan Vuong, Homer Rich Walke, Blake Wulfe, Ted Xiao, Jonathan Heewon Yang, Arefeh Yavary, Tony Z. Zhao, Christopher Agia, Rohan Baijal, Mateo Guaman Castro, Daphne Chen, Qiuyu Chen, Trinity Chung, Jaimyn Drake, Ethan Paul Foster, Jensen Gao, Vitor Guizilini, David Antonio Herrera, Minho Heo, Kyle Hsu, Jiaheng Hu, Muhammad Zubair Irshad, Donovon Jackson, Charlotte Le, Yunshuang Li, Kevin Lin, Roy Lin, Zehan Ma, Abhiram Maddukuri, Suvir Mirchandani, Daniel Morton, Tony Nguyen, Abigail O'Neill, Rosario Scalise, Derick Seale, Victor Son, Stephen Tian, Emi Tran, Andrew E. Wang, Yilin Wu, Annie Xie, Jingyun Yang, Patrick Yin, Yunchu Zhang, Osbert Bastani, Glen Berseth, Jeannette Bohg, Ken Goldberg, Abhinav Gupta, Abhishek Gupta, Dinesh Jayaraman, Joseph J Lim, Jitendra Malik, Roberto Martin-Martin, Subramanian Ramamoorthy, Dorsa Sadigh, Shuran Song, Jiajun Wu, Michael C. Yip, Yuke Zhu, Thomas Kollar, Sergey Levine and Chelsea Finn. Droid: A large-scale in-the-wild robot manipulation dataset. 2024.


Rawal Khirodkar, Timur Bagautdinov, Julieta Martinez, Su Zhaoen, Austin James, Peter Selednik, Stuart Anderson, and Shunsuke Saito. Sapiens: Foundation for human vision models, 2024. URL https://arxiv.org/abs/2408.12569.


Alexander Kirillov, Eric Mintun, Nikhila Ravi, Hanzi Mao, Chloe Rolland, Laura Gustafson, Tete Xiao, Spencer Whitehead, Alexander C Berg, Wan-Yen Lo, et al. Segment anything. In Proceedings of the IEEE/CVF international conference on computer vision, pp. 4015-4026, 2023.


Ranjay Krishna, Yuke Zhu, Oliver Groth, Justin Johnson, Kenji Hata, Joshua Kravitz, Stephanie Chen, Yannis Kalantidis, Li-Jia Li, David A Shamma, et al. Visual genome: Connecting language and vision using crowdsourced dense image annotations. International journal of computer vision, 123(1):32-73, 2017.


Alina Kuznetsova, Hassan Rom, Neil Alldrin, Jasper Uijlings, Ivan Krasin, Jordi Pont-Tuset, Shahab Kamali, Stefan Popov, Matteo Malloci, Alexander Kolesnikov, et al. The open images dataset v4: Unified image classification, object detection, and visual relationship detection at scale. International journal of computer vision, 128(7):1956-1981, 2020.


Alexandre Lacoste, Alexandra Luccioni, Victor Schmidt, and Thomas Dandres. Quantifying the carbon emissions of machine learning. arXiv preprint arXiv:1910.09700, 2019.


Xin Lai, Zhuotao Tian, Yukang Chen, Yanwei Li, Yuhui Yuan, Shu Liu, and Jiaya Jia. Lisa: Reasoning segmentation via large language model. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 9579-9589, 2024.


Hoang-An Le, Partha Das, Thomas Mensink, Sezer Karaoglu, and Theo Gevers. EDEN: Multimodal Synthetic Dataset of Enclosed garDEN Scenes. In Proceedings of the IEEE/CVF Winter Conference of Applications on Computer Vision (WACV), 2021.


Chunyuan Li, Haotian Liu, Liunian Li, Pengchuan Zhang, Jyoti Aneja, Jianwei Yang, Ping Jin, Houdong Hu, Zicheng Liu, Yong Jae Lee, et al. Elevater: A benchmark and toolkit for evaluating language-augmented visual models. Advances in Neural Information Processing Systems, 35:9287-9301, 2022a.


Feng Li, Qing Jiang, Hao Zhang, Tianhe Ren, Shilong Liu, Xueyan Zou, Hu-Sheng Xu, Hongyang Li, Chun yue Li, Jianwei Yang, Lei Zhang, and Jianfeng Gao. Visual in-context prompting. 2024 IEEE/CVF Conference on Computer


---


Vision and Pattern Recognition (CVPR), pp. 12861-12871, 2023a. URL https://api_semanticscholar.org/CorpusID:265351501.


Jiachen Li, Qing Xie, Renshu Gu, Jinyu Xu, Yongjian Liu, and Xiaohan Yu. Lgd: Leveraging generative descriptions for zero-shot referring image segmentation. arXiv preprint arXiv:2504.14467, 2025.


Junnan Li, Dongxu Li, Silvio Savarese, and Steven Hoi. Blip-2: Bootstrapping language-image pre-training with frozen image encoders and large language models, 2023b. URL https://arxiv.org/abs/2301.12597.


Liuian Li, Zi-Yi Dou, Nanyun Peng, and Kai-Wei Chang. Desco: Learning object recognition with rich language descriptions. Advances in Neural Information Processing Systems, 36:37511-37526, 2023c.


Liuian Harold Li, Pengchuan Zhang, Haotian Zhang, Jianwei Yang, Chunyuan Li, Yiwu Zhong, Lijuan Wang, Lu Yuan, Lei Zhang, Jenq-Neng Hwang, et al. Grounded language-image pre-training. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 10965-10975, 2022b.


Siyuan Li, Martin Danelljan, Henghui Ding, Thomas E Huang, and Fisher Yu. Tracking every thing in the wild. In European Conference on Computer Vision, 2022c.


Yanghao Li, Hanzi Mao, Ross Girshick, and Kaiming He. Exploring plain vision transformer backbones for object detection. In European Conference on Computer Vision, pp. 280-296. Springer, 2022d.


Feng Liang, Bichen Wu, Xiaoliang Dai, Kunpeng Li, Yinan Zhao, Hang Zhang, Peizhao Zhang, Peter Vajda, and Diana Marculescu. Open-vocabulary semantic segmentation with mask-adapted clip. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 7061-7070, 2023.


LILA BC. WCS camera traps. URL https://lila.science/datasets/wscameraatrips.


Tsung-Yi Lin, Michael Maire, Serge Belongie, James Hays, Pietro Perona, Deva Ramanan, Piotr Dolkar, and C Lawrence Zitnick. Microsoft coco: Common objects in context. In European conference on computer vision, pp. 740-755. Springer, 2014.


Yutong Lin, Yuhui Yuan, Zheng Zhang, Chen Li, Nanning Zheng, and Han Hu. Detr doesn't need multi-scale or locality design. In Proceedings of the IEEE/CVF International Conference on Computer Vision, pp. 6545-6554, 2023.


Shilong Liu, Zhaoyang Zeng, Tianhe Ren, Feng Li, Hao Zhang, Jie Yang, Chun yue Li, Jianwei Yang, Hang Su, Jun-Juan Zhu, and Lei Zhang. Grounding dino: Marrying dino with grounded pre-training for open-set object detection. In European Conference on Computer Vision, 2023. URL https://api-semanticscholar.org/CorpusID:257427307.


Shilong Liu, Zhaoyang Zeng, Tianhe Ren, Feng Li, Hao Zhang, Jie Yang, Qing Jiang, Chunyuan Li, Jianwei Yang, Hang Su, et al. Grounding dino: Marrying dino with grounded pre-training for open-set object detection. In European conference on computer vision, pp. 38-55. Springer, 2024a.


Ting Liu and Siyuan Li. Hybrid global-local representation with augmented spatial guidance for zero-shot referring image segmentation. In Proceedings of the Computer Vision and Pattern Recognition Conference, pp. 29634-29643, 2025.


Yong Liu, Cairong Zhang, Yitong Wang, Jiahao Wang, Yujiu Yang, and Yansong Tang. Universal segmentation at arbitrary granularity with language instruction. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 3459-3469, 2024b.


Yuqi Liu, Bohao Peng, Zhisheng Zhong, Zihao Yue, Fanbin Lu, Bei Yu, and Jiaya Jia. Seg-zero: Reasoning-chain guided segmentation via cognitive reinforcement. arXiv preprint arXiv:2503.06520, 2025.


Shangbang Long, Siyang Qin, Dmitry Panteleev, Alessandro Bissacco, Yasuhisa Fujii, and Michalis Raptis. Towards end-to-end unified scene text detection and layout analysis. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 2022.


Shangbang Long, Siyang Qin, Dmitry Panteleev, Alessandro Bissacco, Yasuhisa Fujii, and Michalis Raptis. Icdar 2023 competition on hierarchical text detection and recognition. arXiv preprint arXiv:2305.09750, 2023.


Ilya Loshchilov and Frank Hutter. Decoupled weight decay regularization. *ICLR*, 2019.


Yi Lu, Jiawang Cao, Yongliang Wu, Bozheng Li, Licheng Tang, Yangguang Ji, Chong Wu, Jay Wu, and Wenbo Zhu. Rsvp: Reasoning segmentation via visual prompting and multi-modal chain-of-thought. arXiv preprint arXiv:2506.04277, 2025.


---


Jonathon Luiten, Aljosa Osep, Patrick Dendorfer, Philip Torr, Andreas Geiger, Laura Leal-Taixe, and Bastian Leibe. Hota: A higher order metric for evaluating multi-object tracking. International Journal of Computer Vision, pp. 548-578, 2021.


Jun Ma, Zongxin Yang, Sumin Kim, Bihui Chen, Mohammed Baharoon, Adibvafa Fallahpour, Reza Asakereh, Hongwei Lyu, and Bo Wang. Medsam2: Segment anything in 3d medical images and videos. arXiv preprint arXiv:2504.03600, 2025.


Junhua Mao, Jonathan Huang, Alexander Toshev, Oana Camburu, Alan L Yuille, and Kevin Murphy. Generation and comprehension of unambiguous object descriptions. In Proceedings of the IEEE conference on computer vision and pattern recognition, pp. 11-20, 2016.


Xiaofeng Mao, Yuefeng Chen, Yao Zhu, Da Chen, Hang Su, Rong Zhang, and Hui Xue. Coco-o: A benchmark for object detectors under natural distribution shifts, 2023. URL https://arxiv.org/abs/2307.12730.


Tim Meinhardt, Alexander Kirillov, Laura Leal-Taixe, and Christoph Feichtenhofer. Trackformer: Multi-object tracking with transformers. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 8844-8854, 2022.


Matthias Minderer, Alexey Gritsenko, Austin Stone, Maxim Neumann, Dirk Weissenborn, Alexey Dosovitskiy, Aravindh Mahendran, Anurag Arnab, Mostafa Dehghani, Zhuoran Shen, et al. Simple open-vocabulary object detection. In European conference on computer vision, pp. 728-755. Springer, 2022.


Matthias Minderer, Alexey Gritsenko, and Neil Houlsby. Scaling open-vocabulary object detection, 2024. URL https://arxiv.org/abs/2306.09683.


Chaitanya Mitash, Fan Wang, Shiyang Lu, Vikedo Terhuja, Tyler Garaas, Felipe Polido, and Manikantan Nambi. Armbench: An object-centric benchmark dataset for robotic manipulation. arXiv preprint arXiv:2303.16382, 2023.


Margaret Mitchell, Simone Wu, Andrew Zaldivar, Parker Barnes, Lucy Vasserman, Ben Hutchinson, Elena Spitzer, Inioluwa Deborah Raji, and Timnit Gebru. Model cards for model reporting. In Proceedings of the conference on fairness, accountability, and transparency, pp. 220-229, 2019.


Sharada Prasanna Mohanty, Gaurav Singhal, Eric Antoine Scuccimarra, Djilani Kebaili, Harris Héritier, Victor Boulanger, and Marcel Salathé. The food recognition benchmark: Using deeplearning to recognize food on images, 2021. URL https://arxiv.org/abs/2106.14977.


Roozbeh Mottaghi, Xianjie Chen, Xiaobai Liu, Nam-Gyu Cho, Seong-Whan Lee, Sanja Fidler, Raquel Urtasun, and Alan Yuille. The role of context for object detection and semantic segmentation in the wild. In IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2014.


National Gallery of Art. Public domain collection dataset. URL https://www.nga.gov/artworks/ free-images-and-open-access.


Minheng Ni, Yabo Zhang, Kailai Feng, Xiaoming Li, Yiwen Guo, and Wangmeng Zuo. Ref-diff: Zero-shot referring image segmentation with generative models. arXiv preprint arXiv:2308.16777, 2023.


Maxime Oquab, Timothee Darcet, Theo Moutakanni, Huy Vo, Marc Szafraniec, Vasil Khalidov, Pierre Fernandez, Daniel Haziza, Francisco Massa, Alaaeldin El-Nouby, Russell Howes, Po-Yao Huang, Hu Xu, Vasu Sharma, Shang-Wen Li, Wojciech Galuba, Mike Rabbat, Mido Assran, Nicolas Ballas, Gabriel Synnaeve, Ishan Misra, Herve Jegou, Julien Mairal, Patrick Labatut, Armand Joulin, and Piotr Bojanowski. Dinov2: Learning robust visual features without supervision. Transactions on Machine Learning Research, 2024. URL https://openreview.net/forum?id=a68SUt6zFt.


Roni Paiss, Ariel Ephrat, Omer Tov, Shiran Zada, Inbar Mosseri, Michal Irani, and Tali Dekel. Teaching clip to count to ten. In Proceedings of the IEEE/CVF International Conference on Computer Vision, pp. 3170-3180, 2023.


Kwanyong Park, Kuniaki Saito, and Donghyun Kim. Weak-to-strong compositional learning from generative models for language-based object detection. In European Conference on Computer Vision, pp. 1-19. Springer, 2024.


David Patterson, Joseph Gonzalez, Quoc Le, Chen Liang, Lluis-Miquel Munguia, Daniel Rothchild, David So, Maud Texier, and Jeff Dean. Carbon emissions and large neural network training. arXiv preprint arXiv:2104.10350, 2021.


Paul Voigtlaender, Valentin Gabeur and Rohan Doshi. Conversational image segmentation with Gemini 2.5. https://developers.googleblog.com/en/conversational-image-segmentation-gemini-2-5/, 2025.


---


Bryan A Plummer, Kevin J Shih, Yichen Li, Ke Xu, Svetlana Lazebnik, Stan Sclaroff, and Kate Saenko. Revisiting image-language networks for open-ended phrase detection. IEEE transactions on pattern analysis and machine intelligence, 44(4):2155-2167, 2020.


Jordi Pont-Tuset, Federico Perazzi, Sergi Caelles, Pablo Arbeláez, Alex Sorkine-Hornung, and Luc Van Gool. The 2017 davis challenge on video object segmentation. arXiv preprint arXiv:1704.00675, 2017a.


Jordi Pont-Tuset, Federico Perazzi, Sergi Caelles, Pablo Arbeláez, Alexander Sorkine-Hornung, and Luc Van Gool. The 2017 davis challenge on video object segmentation. arXiv:1704.00675, 2017b.


PySceneDetect Developers. PySceneDetect. https://www.scenedetect.com/.


Jiyang Qi, Yan Gao, Yao Hu, Xinggang Wang, Xiaoyu Liu, Xiang Bai, Serge Belongie, Alan Yuille, Philip Torr, and Song Bai. Occluded video instance segmentation: A benchmark. International Journal of Computer Vision, 2022.


Alec Radford, Jong Wook Kim, Chris Hallacy, Aditya Ramesh, Gabriel Goh, Sandhini Agarwal, Girish Sastry, Amanda Askell, Pamela Mishkin, Jack Clark, et al. Learning transferable visual models from natural language supervision. In International conference on machine learning, pp. 8748-8763. PmLR, 2021.


Vikram V. Ramaswamy, Sing Yu Lin, Dora Zhao, Aaron B. Adcock, Laurens van der Maaten, Deepti Ghadiyaram, and Olga Russakovsky. Geode: a geographically diverse evaluation dataset for object recognition. In NeurIPS Datasets and Benchmarks, 2023.


Viresh Ranjan, Udbhav Sharma, Thu Nguyen, and Minh Hoai. Learning to count everything. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 3394-3403, 2021.


Hanoona Rasheed, Muhammad Maaz, Sahal Shaji, Abdelrahman Shaker, Salman Khan, Hisham Cholakkal, Rao M Anwer, Eric Xing, Ming-Hsuan Yang, and Fahad S Khan. Glamm: Pixel grounding large multimodal model. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 13009-13018, 2024.


Nikhila Ravi, Valentin Gabeur, Yuan-Ting Hu, Ronghang Hu, Chaitanya Ryali, Tengyu Ma, Haitham Khedr, Roman Rädle, Chloe Rolland, Laura Gustafson, et al. SAM 2: Segment anything in images and videos. arXiv preprint arXiv:2408.00714, 2024.


Tianhe Ren, Qing Jiang, Shilong Liu, Zhaoyang Zeng, Wenlong Liu, Han Gao, Hongjie Huang, Zhengyu Ma, Xiaoke Jiang, Yihao Chen, et al. Grounding dino 1.5: Advance the" edge" of open-set object detection. arXiv preprint arXiv:2405.10300, 2024a.


Tianhe Ren, Shilong Liu, Ailing Zeng, Jing Lin, Kunchang Li, He Cao, Jiayu Chen, Xinyu Huang, Yukang Chen, Feng Yan, et al. Grounded sam: Assembling open-world models for diverse visual tasks. arXiv preprint arXiv:2401.14159, 2024b.


Tianhe Ren, Yihao Chen, Qing Jiang, Zhaoyang Zeng, Yuda Xiong, Wenlong Liu, Zhengyu Ma, Junyi Shen, Yuan Gao, Xiaoke Jiang, Xingyu Chen, Zhuheng Song, Yuhong Zhang, Hongjie Huang, Han Gao, Shilong Liu, Hao Zhang, Feng Li, Kent Yu, and Lei Zhang. Dino-x: A unified vision model for open-world object detection and understanding, 2025. URL https://arxiv.org/abs/2411.14347.


Peter Robicheaux, Matvei Popov, Anish Madan, Isaac Robinson, Joseph Nelson, Deva Ramanan, and Neehar Peri. Roboflow100-vl: A multi-domain object detection benchmark for vision-language models. arXiv preprint arXiv:2505.20612, 2025.


Olga Russakovsky, Jia Deng, Hao Su, Jonathan Krause, Sanjeev Satheesh, Sean Ma, Zhiheng Huang, Andrej Karpathy, Aditya Khosla, Michael Bernstein, et al. Imagenet large scale visual recognition challenge. International journal of computer vision, 115(3):211-252, 2015.


Chaitanya Ryali, Yuan-Ting Hu, Daniel Bolya, Chen Wei, Haoqi Fan, Po-Yao Huang, Vaibhav Aggarwal, Arkabandhu Chowdhury, Omid Poursaeed, Judy Hoffman, Jitendra Malik, Yanghao Li, and Christoph Feichtenhofer. Hiera: A hierarchical vision transformer without the bells-and-whistles. ICML, 2023.


SA-FARI Dataset. https://www.conserationxlabs.com/sa-fari.


Victor Sanh, Lysandre Debut, Julien Chaumont, and Thomas Wolf. Distilbert, a distilled version of bert: smaller, faster, cheaper and lighter. ArXiv, abs/1910.01108, 2019.


Samuel Schulter, Yumin Suh, Konstantinos M Dafnis, Zhixing Zhang, Shiyu Zhao, Dimitris Metaxas, et al. Omnilabel: A challenging benchmark for language-based object detection. In Proceedings of the IEEE/CVF International Conference on Computer Vision, pp. 11953-11962, 2023.


---


Shuai Shao, Zeming Li, Tianyuan Zhang, Chao Peng, Gang Yu, Xiangyu Zhang, Jing Li, and Jian Sun. Objects365: A large-scale, high-quality dataset for object detection. In Proceedings of the IEEE/CVF international conference on computer vision, pp. 8430-8439, 2019.


Yunhang Shen, Chaoyou Fu, Peixian Chen, Mengdan Zhang, Ke Li, Xing Sun, Yunsheng Wu, Shaohui Lin, and Rongrong Ji. Aligning and prompting everything all at once for universal visual perception. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR), pp. 13193-13203, June 2024.


Jianlin Su, Yu Lu, Shengfeng Pan, Bo Wen, and Yunfeng Liu. Roformer: Enhanced transformer with rotary position embedding. arXiv preprint arXiv:2104.09864, 2021.


Peize Sun, Jinkun Cao, Yi Jiang, Rufeng Zhang, Enze Xie, Zehuan Yuan, Changhu Wang, and Ping Luo. Transtrack: Multiple object tracking with transformer. arXiv preprint arXiv:2012.15460, 2020.


Shuyang Sun, Runjia Li, Philip Torr, Xiuye Gu, and Siyang Li. Clip as rnn: Segment countless visual concepts without training endeavor. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 13171-13182, 2024.


Yucheng Suo, Linchao Zhu, and Yi Yang. Text augmented spatial-aware zero-shot referring image segmentation. arXiv preprint arXiv:2310.18049, 2023.


Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N Gomez, Lukasz Kaiser, and Illia Polosukhin. Attention is all you need. In I. Guyon, U. Von Luxburg, S. Bengio, H. Wallach, R. Fergus, S. Vishwanathan, and R. Garnett (eds.), Advances in Neural Information Processing Systems, volume 30. Curran Associates, Inc., 2017. URL https://proceedings.neurips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf.


Denny Vrandecic and Markus Krötzsch. Wikidata: a free collaborative knowledgebase. Commun. ACM, 57(10):78-85, September 2014. ISSN 0001-0782. doi: 10.1145/2629489. URL https://doi.org/10.1145/2629489.


Hao Wang, Limeng Qiao, Zequn Jie, Zhijian Huang, Chengjian Feng, Qingfang Zheng, Lin Ma, Xiangyuan Lan, and Xiaodan Liang. X-sam: From segment anything to any segmentation. arXiv preprint arXiv:2508.04655, 2025a.


Haochen Wang, Cilin Yan, Shuai Wang, Xiaolong Jiang, Xu Tang, Yao Hu, Weidi Xie, and Efstratios Gavves. Towards open-vocabulary video instance segmentation. In proceedings of the IEEE/CVF international conference on computer vision, pp. 4057-4066, 2023.


Peng Wang, Shuai Bai, Sinan Tan, Shijie Wang, Zhihao Fan, Jinze Bai, Keqin Chen, Xuejing Liu, Jialin Wang, Wenbin Ge, et al. Qwen2-vl: Enhancing vision-language model's perception of the world at any resolution. arXiv preprint arXiv:2409.12191, 2024.


Shijie Wang, Dahun Kim, Ali Taalimi, Chen Sun, and Weicheng Kuo. Learning visual grounding from generative vision and language model. In 2025 IEEE/CVF Winter Conference on Applications of Computer Vision (WACV), pp. 8057-8067. IEEE, 2025b.


Yuji Wang, Jingchen Ni, Yong Liu, Chun Yuan, and Yansong Tang. Iterprime: Zero-shot referring image segmentation with iterative grad-cam refinement and primary word emphasis. In Proceedings of the AAAI Conference on Artificial Intelligence, pp. 8159-8168, 2025c.


Cong Wei, Yujie Zhong, Haoxian Tan, Yong Liu, Zheng Zhao, Jie Hu, and Yujiu Yang. Hyperseg: Towards universal visual segmentation with large language model. arXiv preprint arXiv:2411.17606, 2024.


Nicolai Wojke, Alex Bewley, and Dietrich Paulus. Simple online and realtime tracking with a deep association metric. In 2017 IEEE international conference on image processing (ICIP), pp. 3645-3649. IEEE, 2017.


Junfeng Wu, Yi Jiang, Qihao Liu, Zehuan Yuan, Xiang Bai, and Song Bai. General object foundation model for images and videos at scale. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 3783-3795, 2024a.


Junfeng Wu, Yi Jiang, Qihao Liu, Zehuan Yuan, Xiang Bai, and Song Bai. General object foundation model for images and videos at scale. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 3783-3795, 2024b.


Zhuofan Xia, Dongchen Han, Yizeng Han, Xuran Pan, Shiji Song, and Gao Huang. Gsva: Generalized segmentation via multimodal large language models. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 3858-3869, 2024.


---


Yin Xie, Kaicheng Yang, Xiang An, Kun Wu, Yongle Zhao, Weimo Deng, Zimin Ran, Yumeng Wang, Ziyong Feng, Roy Miles, et al. Region-based cluster discrimination for visual representation learning. arXiv preprint arXiv:2507.20025, 2025.


Hu Xu, Po-Yao Huang, Xiaqing Ellen Tan, Ching-Feng Yeh, Jacob Kahn, Christine Jou, Gargi Ghosh, Omer Levy, Luke Zettlemoyer, Wen-tau Yih, et al. Altogether: Image captioning via re-aligning alt-text. arXiv preprint arXiv:2410.17251, 2024a.


Hu Xu, Saining Xie, Xiaoqing Ellen Tan, Po-Yao Huang, Russell Howes, Vasu Sharma, Shang-Wen Li, Gargi Ghosh, Luke Zettlemoyer, and Christoph Feichtenhofer. Demystifying clip data, 2024b. URL https://arxiv.org/abs/2309.16671.


N. Xu, L. Yang, Yuchen Fan, Dingcheng Yue, Yuchen Liang, Jianchao Yang, and Thomas S. Huang. Youtube-vos: A large-scale video object segmentation benchmark. ArXiv, abs/1809.03327, 2018.


Yifan Xu, Mengdan Zhang, Chaoyou Fu, Peixian Chen, Xiaoshan Yang, Ke Li, and Changsheng Xu. Multi-modal queried object detection in the wild. Advances in Neural Information Processing Systems, 36:4452-4469, 2023.


Cheng-Yen Yang, Hsiang-Wei Huang, Wenhao Chai, Zhongyu Jiang, and Jenq-Neng Hwang. *Samurai: Adapting segment anything model for zero-shot visual tracking with motion-aware memory.* arXiv preprint arXiv:2411.11922, 2024.


Jianwei Yang, Hao Zhang, Feng Li, Xueyan Zou, Chunyuan Li, and Jianfeng Gao. Set-of-mark prompting unleashes extraordinary visual grounding in gpt-4v. arXiv preprint arXiv:2310.11441, 2023.


Heng Yin, Yuqiang Ren, Ke Yan, Shouhong Ding, and Yongtao Hao. Rod-mllm: Towards more reliable object detection in multimodal large language models. In Proceedings of the Computer Vision and Pattern Recognition Conference, pp. 14358-14368, 2025.


En Yu, Tiancai Wang, Zhuoling Li, Yang Zhang, Xiangyu Zhang, and Wenbing Tao. Motrv3: Releasefetch supervision for end-to-end multi-object tracking. arXiv preprint arXiv:2305.14298, 2023a.


Fisher Yu, Haofeng Chen, Xin Wang, Wenqi Xian, Yingying Chen, Fangchen Liu, Vashisht Madhavan, and Trevor Darrell. Bdd100k: A diverse driving dataset for heterogeneous multitask learning. In The IEEE Conference on Computer Vision and Pattern Recognition (CVPR), June 2020.


Seonghoon Yu, Paul Hongsuck Seo, and Jeany Son. Zero-shot referring image segmentation with global-local context features. In Proceedings of the IEEE/CVF conference on computer vision and pattern recognition, pp. 19456-19465, 2023b.


Seonghoon Yu, Paul Hongsuck Seo, and Jeany Son. Pseudo-ris: Distinctive pseudo-supervision generation for referring image segmentation. In European Conference on Computer Vision, pp. 18-36. Springer, 2024.


Rowan Zellers, Jiasen Lu, Ximing Lu, Youngjae Yu, Yanpeng Zhao, Mohammadreza Salehi, Aditya Kusupati, Jack Hessel, Ali Farhadi, and Yejin Choi. Merlot reserve: Multimodal neural script knowledge through vision and language and sound. In CVPR, 2022.


Fangao Zeng, Bin Dong, Yuang Zhang, Tiancai Wang, Xiangyu Zhang, and Yichen Wei. Motr: End-to-end multiple-object tracking with transformer. In European conference on computer vision, pp. 659-675. Springer, 2022.


Xiaohua Zhai, Alexander Kolesnikov, Neil Houlsby, and Lucas Beyer. Scaling vision transformers, 2022. URL https://arxiv.org/abs/2106.04560.


Hao Zhang, Feng Li, Shilong Liu, Lei Zhang, Hang Su, Jun Zhu, Lionel M Ni, and Heung-Yeung Shum. Dino: Detr with improved denoising anchor boxes for end-to-end object detection. arXiv preprint arXiv:2203.03605, 2022a.


Haotian Zhang, Pengchuan Zhang, Xiaowei Hu, Yen-Chun Chen, Liunian Li, Xiyang Dai, Lijuan Wang, Lu Yuan, Jenq-Neng Hwang, and Jianfeng Gao. Glipv2: Unifying localization and vision-language understanding. Advances in Neural Information Processing Systems, 35:36067-36080, 2022b.


Tao Zhang, Xiangtai Li, Hao Fei, Haobo Yuan, Shengqiong Wu, Shunping Ji, Chen Change Loy, and Shuicheng Yan. Omg-llava: Bridging image-level, object-level, pixel-level reasoning and understanding. Advances in neural information processing systems, 37:71737-71767, 2024a.


Yifu Zhang, Peize Sun, Yi Jiang, Dongdong Yu, Fucheng Weng, Zehuan Yuan, Ping Luo, Wenyu Liu, and Xinggang Wang. Bytetrack: Multi-object tracking by associating every detection box. In European conference on computer vision, pp. 1-21. Springer, 2022c.


---


Yuxuan Zhang, Tianheng Cheng, Lianghui Zhu, Rui Hu, Lei Liu, Heng Liu, Longjin Ran, Xiaoxin Chen, Wenyu Liu, and Xinggang Wang. Evf-sam: Early vision-language fusion for text-prompted segment anything model. arXiv preprint arXiv:2406.20076, 2024b.


Zheng Zhang, Yeyao Ma, Enming Zhang, and Xiang Bai. Psalm: Pixelwise segmentation with large multi-modal model. In European Conference on Computer Vision, pp. 74-91. Springer, 2024c.


Zhixiong Zhang, Shuangrui Ding, Xiaoyi Dong, Songxin He, Jianfan Lin, Junsong Tang, Yuhang Zang, Yuhang Cao, Dahua Lin, and Jiaqi Wang. Sec: Advancing complex video object segmentation via progressive concept construction. arXiv preprint arXiv:2507.15852, 2025.


Shiyu Zhao, Long Zhao, Yumin Suh, Dimitris N Metaxas, Manmohan Chandraker, Samuel Schulter, et al. Generating enhanced negatives for training language-based object detectors. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 13592-13602, 2024.


Bolei Zhou, Hang Zhao, Xavier Puig, Tete Xiao, Sanja Fidler, Adela Barriuso, and Antonio Torralba. Semantic understanding of scenes through the ade20k dataset. International Journal of Computer Vision, 127(3):302-321, 2019.


Yang Zhou, Shiyu Zhao, Yuxiao Chen, Zhenting Wang, Can Jin, and Dimitris N Metaxas. Led: Llm enhanced open-vocabulary object detection without human curated data generation. arXiv preprint arXiv:2503.13794, 2025.


Pengfei Zhu, Longyin Wen, Dawei Du, Xiao Bian, Heng Fan, Qinghua Hu, and Haibin Ling. Detection and tracking meet drones challenge. IEEE Transactions on Pattern Analysis and Machine Intelligence, 44(11):7380-7399, 2021.


Xizhou Zhu, Weijie Su, Lewei Lu, Bin Li, Xiaogang Wang, and Jifeng Dai. Deformable detr: Deformable transformers for end-to-end object detection. arXiv preprint arXiv:2010.04159, 2020.


Xueyan Zou, Jianwei Yang, Hao Zhang, Feng Li, Linjie Li, Jianfeng Wang, Lijuan Wang, Jianfeng Gao, and Yong Jae Lee. Segment everything everywhere all at once. Advances in neural information processing systems, 36:19769-19782, 2023.
