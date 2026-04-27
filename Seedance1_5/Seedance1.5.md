
# Seedance 1.5 pro: A Native Audio-Visual Joint Generation Foundation Model


ByteDance Seed


# Abstract


Recent strides in video generation have paved the way for unified audio-visual generation. In this work, we present Seedance 1.5 pro, a foundational model engineered specifically for native, joint audio-video generation. Leveraging a dual-branch Diffusion Transformer architecture, the model integrates a cross-modal joint module with a specialized multi-stage data pipeline, achieving exceptional audio-visual synchronization and superior generation quality. To ensure practical utility, we implement meticulous post-training optimizations, including Supervised Fine-Tuning (SFT) on high-quality datasets and Reinforcement Learning from Human Feedback (RLHF) with multi-dimensional reward models. Furthermore, we introduce an acceleration framework that boosts inference speed by over \(10 \times\). Seedance 1.5 pro distinguishes itself through precise multilingual and dialect lip-syncing, dynamic cinematic camera control, and enhanced narrative coherence, positioning it as a robust engine for professional-grade content creation. Seedance 1.5 pro is now accessible on Volcano Engine<sup>α</sup>.


Official Page: https://seed.bytedance.com/seedance1_5_pro
\(^{\alpha}\)Model ID: Doubao-Seedance-1.5-pro


_Video Capabilities Evaluation_


![图片 page1-7](images/page001_img01.png)


_Audio Capabilities Evaluation_


![图片 page1-9](images/page001_img02.png)


_Figure 1 Overall Evaluation. Left: Video Evaluation; Right: Audio Evaluation._


---


# 1 Introduction


Over the past year, the field of visual (video) generation [2, 5, 9, 11-13] has seen rapid development. The emergence of proprietary commercial systems, such as Veo, Sora, Kling series and Seedance [3], alongside open-source models like Wan [11] and Hunyuan Video 1.5 [6], has significantly propelled the widespread adoption of video generation technologies across both academia and industry. More recently, substantial progress has been made in joint audio-video generation. The release of Wan 2.5, Kling 2.6, and Sora 2 marks a solid step forward, transforming video generation capabilities into practical, utility-driven tools.


In this work, we present Seedance 1.5 pro, a foundational model designed with native support for joint video-audio generation. It is capable of performing a diverse range of tasks, including text-to-audio-video synthesis and image-guided audio-video generation. Seedance 1.5 pro incorporates the following key technical advancements:


- Comprehensive Audio-Visual Data Framework. We present a holistic data framework for high-quality video-audio generation that integrates a multi-stage curation pipeline, an advanced captioning system, and scalable infrastructure. The pipeline prioritizes video-audio coherence, motion expressiveness, and curriculum-based data scheduling, while our captioning system provides rich, professional-grade descriptions for both video and audio modalities. This robust framework is underpinned by an efficient engineering infrastructure optimized for massive multi-modal data processing.


- Unified Multimodal Joint Generation Architecture. To achieve native video-audio joint synthesis, we propose a unified framework based on the MMDiT [1] architecture. This design facilitates deep cross-modal interaction, ensuring precise temporal synchronization and semantic consistency between visual and auditory streams. By leveraging multi-task pre-training on large-scale mixed-modality datasets, our model achieves robust generalization across diverse downstream tasks, including Text-to-Video-Audio (T2VA), Image-to-Video-Audio (I2VA), and unimodal video generation (T2V, I2V).


- Meticulous Post-training Optimization. We utilized high-quality audio-video datasets for Supervised Fine-Tuning (SFT), followed by a Reinforcement Learning from Human Feedback (RLHF[7, 14-16]) algorithm specifically tailored for audio-video contexts. Specifically, our multi-dimensional reward model enhances performance in Text-to-Video (T2V) and Image-to-Video (I2V) tasks, improving motion quality, visual aesthetics, and audio fidelity. Moreover, targeted infrastructure optimizations to our RLHF pipeline have yielded a nearly \(3 \times\) improvement in training speed.


- Efficient Inference Acceleration. We further optimized a multi-stage distillation framework [4, 8, 10] to substantially reduce the Number of Function Evaluations (NFE) required during generation. By integrating inference infrastructure optimizations—such as quantization and parallelism, we achieved an end-to-end acceleration exceeding \(10 \times\) while preserving model performance.


![图片 page2-8](images/page002_img01.png)


_Figure 2 Overview of training and inference pipeline of Seedance 1.5 pro._


Seedance 1.5 pro aims to elevate the ceiling of visual impact and motion expressiveness. Seedance 1.5 pro demonstrates comprehensive advancements in multilingual adaptation, motion expressiveness, camera


---


scheduling, and dialect delivery. Its performance in scenarios such as Chinese film production, short dramas, and traditional performing arts demonstrates significant potential for practical application in multi-shot video generation workflows:


- Precise Audio-Visual Synchronization and Multilingual Support. The model realizes high audio-visual consistency during generation, significantly improving the alignment accuracy of lip movements, intonation, and performance rhythm. It natively supports multiple languages and regional dialects, accurately capturing their unique vocal prosody and emotional tension. This capability imparts a natural performance quality to stylized narratives, such as comedies and animations, thereby greatly enhancing character atmosphere and expressiveness.


- Cinematic Camera Control and Dynamic Tension. The model possesses autonomous camera scheduling capabilities, enabling the execution of complex movements such as continuous long takes and dolly zooms (Hitchcock zoom). Additionally, it achieves cinematic scene transitions and professional color grading, substantially enhancing the dynamic tension of the video.


- Enhanced Semantic Understanding and Narrative Coherence. Through strengthened semantic understanding, Seedance 1.5 pro achieves precise analysis of narrative contexts. It significantly improves the overall narrative coordination of audio-visual segments, providing strong support for professional-grade content creation.


Seedance 1.5 pro is scheduled to be integrated into multiple platforms, including Doubao<sup>1</sup> and Jimeng<sup>2</sup>, by December 2025. We envision this model serving as a vital productivity tool, designed to enhance both professional workflows and daily creative applications.


# 2 Evaluation


As AIGC video generation evolves toward deeper multimodal integration, the incorporation of audio capabilities has become a pivotal factor in transforming video generation from high-quality visual assets into holistic and production-ready works. By achieving precise audio-visual synchronization and coherent emotional alignment, the model outputs are no longer fragmented visuals, but represent increasingly mature productions characterized by narrative completeness and immersive quality.


This chapter provides a comprehensive analysis of the Seedance 1.5 pro model. Section 2.1 details the internal evaluation, outlining our assessment methodology in both video and audio dimensions and presenting comparative analyses against state-of-the-art (SOTA) video generation models. Finally, we highlight the distinct advantages of the model in diverse application scenarios.


# 2.1 Comprehensive Evaluation


In contrast to third-party benchmarks that prioritize general user preference, we established a comprehensive, multi-dimensional framework designed to assess model performance across diverse domains and capabilities. By integrating rigorous benchmark construction with specific evaluation criteria, this framework provides granular insights that guide model development and facilitate rapid iteration. Building upon SeedVideoBench-1.0, which was grounded in the analysis of real-world user prompts, we introduce the enhanced SeedVideoBench-1.5. Relative to its predecessor, this version significantly expands the coverage of industry-specific scenarios, such as advertising and micro-dramas, while incorporating advanced metrics for audio evaluation. Consistent with our established methodology, we collaborate with professional film directors to codify these criteria and engaged experts from film production, cinematography, and design to conduct expert-level human assessments.


# 2.1.1 SeedVideoBench 1.5


For the video dimension, we provide a detailed taxonomy of evaluation cases and attribute labels covering core aspects such as subjects, motion dynamics, interactions, and camera movements. The updated benchmark


<sup>1</sup>https://www.doubao.com/chat/create-video


\(^{2}\)https://jimeng.jianying.com/ai-tool/video/generate


---


also integrates labels for diversified application scenarios, including advertising, social media content, and short-form narrative content. In light of the Seedance 1.5 capability for joint audio-video generation, we have upgraded the evaluation framework to include a comprehensive audio dimension. The primary label categories are defined as follows:


- Human Voice Types: As a critical component of video creation, this category encompasses speech, singing, and non-verbal vocalizations (e.g., laughter). SeedVideoBench 1.5 introduces a comprehensive taxonomy with fine-grained sub-dimensions to capture the diverse requirements of human voice generation.


- Human Voice Attributes: These labels characterize specific vocal qualities, including timbre, accent, and emotional tone.


- Non-Speech Audio (Sound Effects/Music): This category includes all environmental and musical elements essential for perceptual realism and scene coherence. The labeling system classifies audio by source (e.g., animals, mechanical tools), acoustic properties, musical genres, and technical parameters.


The audio labeling methodology follows consistent principles across both T2V and I2V tasks. However, in I2V contexts, audio synthesis is explicitly conditioned on the visual cues present in the reference image to ensure semantic consistency and cross-modal coherence.


# 2.1.2 Video Evaluation Metrics


In SeedVideoBench 1.5, we extend our prior focus on motion dynamics, prompt adherence, aesthetic quality, and subject consistency by introducing specialized evaluation metrics designed to better reflect professional production requirements. The key updates are detailed below.


- Motion Quality: Motion quality remains the most immediate and critical concern for users. While baseline attributes—such as stability, physical plausibility, and temporal accuracy—remain mandatory, this benchmark iteration places renewed emphasis on Video Vividness, which has emerged as an increasingly important indicator for downstream professional applications. As generative capabilities mature across the field, foundational motion metrics are likely to approach performance saturation. Consequently, practitioners in advertising and film production increasingly demand video outputs that exhibit higher levels of vividness across diverse scenarios. Notably, among several state-of-the-art models, we observe a common trade-off in which slow-motion generation is employed to artificially enhance perceived stability, a strategy that significantly degrades motion vividness and expressive quality. To address this limitation, we evaluate video vividness as a composite perceptual metric across four principal dimensions: action, camera movement, atmosphere, and emotion. The following two were in particular examined:


Action Dimension: Vividness in action is characterized by nuanced facial expressions, aesthetically refined body poses, high-fidelity reproduction of fine-grained motions, and realistic interactions with the environment. Together, these factors determine the perceptual authenticity and emotional expressiveness of generated motion.


Camera Dimension: Cinematic composition and dynamic camera movements play a critical role in enhancing visual expressiveness, strengthening emotional delivery, and supporting temporal narrative coherence across shots.


- Prompt Following: While strict adherence to user instructions remains necessary when users explicitly request word-for-word execution, the definition of prompt adherence has been updated to reflect the improved semantic understanding and intent recognition of modern generative models. Rather than emphasizing surface-level keyword matching, the evaluation now prioritizes consistency with the user's underlying intent.


In narrative-driven social media scenarios, models are permitted a degree of intent-aligned creative flexibility, provided that the core user intention is preserved. Such flexibility may include completing missing visual details, refining narrative structure, or generating dialogue that better aligns with the intended emotional tone—thereby contributing to stronger storytelling quality and cross-modal expressive coherence.


---


# 2.1.3 Audio Evaluation Metrics


Our audio evaluation metrics are designed to quantitatively characterize model performance across multiple dimensions, establishing a roadmap for advancing audio generation toward cinematic-level production standards. The evaluation framework comprises four core components:


- Audio Prompt Following: Analogous to text-video alignment, this metric assesses the fidelity with which vocal elements, dialogue, and sound effects adhere to user instructions and intended semantics, without semantic drift—even during creative extrapolation (i.e., intent-consistent content expansion). Common failure modes include the omission of specified sound effects, linguistic or dialectal inaccuracies, and audio–vision mismatches (e.g., speech generation without corresponding lip motion).


- Audio Quality: This metric quantifies the intrinsic acoustic quality of the output, encompassing both vocal and non-vocal components. Key evaluation criteria include the presence of artifacts (e.g., clipping, truncation), spatial soundstage rendering, timbre realism, and overall signal clarity.


- Audio-Visual Synchronization: This metric measures the temporal alignment between auditory and visual streams. It evaluates the synchronization of speech-to-lip dynamics (e.g., mitigating perceptual ventriloquism effects), the alignment of sound effects with visual events, and the presence of auditory cues corresponding to salient on-screen actions.


- Audio Expressiveness: This metric evaluates the extent to which audio augments the emotional resonance of the video. It considers the thematic appropriateness of background music (BGM), the emotional inflection of speech, and the audio's contribution to atmospheric immersion and narrative coherence and depth.


# 2.1.4 Human Evaluation


Video. Leveraging the SeedVideoBench 1.5 framework, we conducted a comprehensive comparative evaluation of Seedance 1.5 pro against several state-of-the-art video generation models across both Text-to-Video (T2V) and Image-to-Video (I2V) tasks. The comparative baselines include Kling 2.5, Kling 2.6, Veo 3.1, and Seedance 1.0 Pro. To ensure a robust assessment, we employed a dual-metric evaluation protocol:


- Absolute Score: This metric utilizes a 5-point Likert scale (ranging from 1 for "Extremely Dissatisfied" to 5 for "Extremely Satisfied") to facilitate a standardized performance comparison across different models.


- Good-Same-Bad (GSB): This metric involves pairwise comparisons to assess relative video quality, thereby enabling a more granular differentiation of model outputs.


![图片 page5-12](images/page005_img01.png)


_Figure 3 Video Absolute Evaluation for Text-to-Video task._


Figures 3 and 4 present the absolute evaluation scores of the video generation models in the Text-to-Video (T2V) and Image-to-Video (I2V) task. Seedance 1.5 pro demonstrates a significant improvement over its


---


predecessor, Seedance 1.0 Pro. Specifically, in T2V generation, Seedance 1.5 pro achieves a leading position in instruction following (alignment). Furthermore, it exhibits strong competitiveness in terms of visual aesthetics and motion dynamics, as well as in Image-to-Video (I2V) tasks.


![图片 page6-1](images/page006_img01.png)


_Figure 4 Video Absolute Evaluation for Image-to-Video task._


Audio. Figure 5 and 6 presents a multi-dimensional side-by-side (GSB) comparative evaluation of audio performance between the Seedance 1.5 pro and several competing systems. While models such as Veo 3.1, Wan 2.5, Kling 2.6, and Sora 2 demonstrate robust audio-generation capabilities, the Seedance 1.5 pro exhibits distinct advantages in several key dimensions:


![图片 page6-4](images/page006_img02.png)


_Figure 5 Audio GSB Evaluation for Text-to-Video task._


Superiority in Chinese-Language Audio. Seedance 1.5 pro consistently outperforms Veo 3.1 in Chinese vocal generation. It demonstrates a clear advantage in synthesizing dialogue, dialects, and monologues within Chinese-language contexts, reliably producing accurate responses with high articulatory clarity. Notably, the model remains largely free from common artifacts such as syllable dropping or mispronunciation.


Enhanced Audio-Visual Synchronization. Seedance 1.5 pro model excels in aligning both vocal tracks and sound effects with visual cues. In terms of lip-audio synchronization, the model accurately corresponds to the number and identity of speaking characters, effectively mitigating errors related to mouth motion redundancy or omission that typically result in audio-visual temporal misalignment. In this regard, its performance surpasses both Veo 3.1 and Kling 2.6.


---


![图片 page7-0](images/page007_img01.png)


_Figure 6 Audio GSB Evaluation for Image-to-Video task._


Comparative Analysis of Audio Expressiveness. Sora 2 demonstrates strong competence in emotional expressiveness, delivering particularly vivid emotional inflection in its audio outputs. By contrast, Seedance 1.5 pro maintains a more balanced and controlled expressiveness profile, achieving consistent emotional alignment with visual content while avoiding over-exaggeration. This characteristic is especially advantageous in professional production scenarios that require stable tone control and narrative coherence.


# 2.2 Potential Advantages in Application Scenarios


Seedance 1.5 pro further strengthens its capabilities in character expressiveness and stylized visual construction, with particularly strong performance in Chinese-language contexts. Compared to previous generations, Seedance 1.5 pro demonstrates substantial improvements in character dialogue, including support for multiple Chinese dialects, as well as in the execution of complex camera dynamics. These advancements make the model well-suited for Chinese film production, short-form micro-dramas, and theatrical storytelling scenarios.


Specifically, Seedance 1.5 pro maintains consistent lip synchronization, vocal tonality, and performance rhythm across continuous shots. The model exhibits robust performance in dialect-rich settings, such as Sichuanese, Taiwan Mandarin, Cantonese, and Shanghainese, producing natural prosody and speech patterns that closely resemble authentic regional usage. These capabilities are particularly beneficial for genre-specific storytelling, including comedy and comic-style narratives, where dialectical cadence and performative tension play a critical role in establishing atmosphere and comedic timing.


In terms of visual composition, Seedance 1.5 pro demonstrates mature control over complex camera operations. The model reliably executes orbital, arc, and tracking shots while preserving visual style consistency between generated sequences and reference imagery, thereby ensuring cinematic visual continuity across scenes. Moreover, leveraging improved prompt understanding, Seedance 1.5 pro can autonomously introduce novel subjects and actions that remain aligned with the narrative genre, enhancing overall scene coherence and immersive continuity.


![图片 page7-7](images/page007_img02.png)


Additionally, Seedance 1.5 pro demonstrates a heightened sensitivity to the traditional Chinese performance


---


system within stage opera scenarios. While its mastery of specific vocal styles across different opera sub-genres is still evolving, the model is already capable of capturing the distinctive cadence and flavor of operatic speech (Nianbai). By integrating nuanced performance details, such as the orchid hand gesture (Lanhua Zhi) and the stylized eye expressions typical of comedic roles, Seedance 1.5 pro effectively constructs a performance atmosphere deeply rooted in Eastern operatic aesthetics.


![图片 page8-1](images/page008_img01.png)


In realistic cinematic close-up shots, Seedance 1.5 pro sustains emotional continuity through subtle and coherent facial micro-expressions. Even in segments with minimal dialogue, the model preserves the integrity of character performance, providing creators with greater narrative flexibility and space for interpretive silence.


![图片 page8-3](images/page008_img02.png)


![图片 page8-4](images/page008_img03.png)


In summary, the advance of Seedance 1.5 pro in Chinese-language generation, stylized visual rendering, complex camera control, and traditional theatrical expression enable a high degree of narrative expressiveness and seamless audio-visual integration. Collectively, these capabilities enhance the creative controllability of the model in various application domains, including Chinese film production, short-form drama creation, and opera-inspired audiovisual storytelling.


---


# References


[1] Patrick Esser, Sumith Kulal, Andreas Blattmann, Rahim Entezari, Jonas Müller, Harry Saini, Yam Levi, Dominik Lorenz, Axel Sauer, Frederic Boesel, et al. Scaling rectified flow transformers for high-resolution image synthesis. In _Forty-first international conference on machine learning_, 2024.


[2] Yu Gao, Lixue Gong, Qiushan Guo, Xiaoxia Hou, Zhichao Lai, Fanshi Li, Liang Li, Xiaochen Lian, Chao Liao, Liyang Liu, et al. Seedream 3.0 technical report. arXiv preprint arXiv:2504.11346, 2025.


[3] Yu Gao, Haoyuan Guo, Tuyen Hoang, Weilin Huang, Lu Jiang, Fangyuan Kong, Huixia Li, Jiashi Li, Liang Li, Xiaojie Li, et al. Seedance 1.0: Exploring the boundaries of video generation models. arXiv preprint arXiv:2506.09113, 2025.


[4] Zhengyang Geng, Mingyang Deng, Xingjian Bai, J Zico Kolter, and Kaiming He. Mean flows for one-step generative modeling. arXiv preprint arXiv:2505.13447, 2025.


[5] Lixue Gong, Xiaoxia Hou, Fanshi Li, Liang Li, Xiaochen Lian, Fei Liu, Liyang Liu, Wei Liu, Wei Lu, Yichun Shi, et al. Seedream 2.0: A native chinese-english bilingual image generation foundation model. arXiv preprint arXiv:2503.07703, 2025.


[6] Weijie Kong, Qi Tian, Zijian Zhang, Rox Min, Zuozhuo Dai, Jin Zhou, Jiangfeng Xiong, Xin Li, Bo Wu, Jianwei Zhang, et al. Hunyuanvideo: A systematic framework for large video generative models. arXiv preprint arXiv:2412.03603, 2024.


[7] Jie Liu, Gongye Liu, Jiajun Liang, Yangguang Li, Jiaheng Liu, Xintao Wang, Pengfei Wan, Di Zhang, and Wanli Ouyang. Flow-grpo: Training flow matching models via online rl. arXiv preprint arXiv:2505.05470, 2025.


[8] Yuxi Ren, Xin Xia, Yanzuo Lu, Jiacheng Zhang, Jie Wu, Pan Xie, Xing Wang, and Xuefeng Xiao. Hyper-sd: Trajectory segmented consistency model for efficient image synthesis. Advances in Neural Information Processing Systems, 37:117340-117362, 2025.


[9] Team Seedream, Yunpeng Chen, Yu Gao, Lixue Gong, Meng Guo, Qiushan Guo, Zhiyao Guo, Xiaoxia Hou, Weilin Huang, Yixuan Huang, et al. Seedream 4.0: Toward next-generation multimodal image generation. arXiv preprint arXiv:2509.20427, 2025.


[10] Huiyang Shao, Xin Xia, Yuhong Yang, Yuxi Ren, Xing Wang, and Xuefeng Xiao. Rayflow: Instance-aware diffusion acceleration via adaptive flow trajectories. arXiv preprint arXiv:2503.07699, 2025.


[11] Team Wan, Ang Wang, Baole Ai, Bin Wen, Chaojie Mao, Chen-Wei Xie, Di Chen, Feiwu Yu, Haiming Zhao, Jianxiao Yang, et al. Wan: Open and advanced large-scale video generative models. arXiv preprint arXiv:2503.20314, 2025.


[12] Bing Wu, Chang Zou, Changlin Li, Duojun Huang, Fang Yang, Hao Tan, Jack Peng, Jianbing Wu, Jiangfeng Xiong, Jie Jiang, et al. Hunyuanvideo 1.5 technical report. arXiv preprint arXiv:2511.18870, 2025.


[13] Chenfei Wu, Jiahao Li, Jingren Zhou, Junyang Lin, Kaiyuan Gao, Kun Yan, Sheng-ming Yin, Shuai Bai, Xiao Xu, Yilei Chen, et al. Qwen-image technical report. arXiv preprint arXiv:2508.02324, 2025.


[14] Jie Wu, Yu Gao, Zilyu Ye, Ming Li, Liang Li, Hanzhong Guo, Jie Liu, Zeyue Xue, Xiaoxia Hou, Wei Liu, et al. Rewarddance: Reward scaling in visual generation. arXiv preprint arXiv:2509.08826, 2025.


[15] Zeyue Xue, Jie Wu, Yu Gao, Fangyuan Kong, Lingting Zhu, Mengzhao Chen, Zhiheng Liu, Wei Liu, Qiushan Guo, Weilin Huang, et al. Dancegrpo: Unleashing grpo on visual generation. arXiv preprint arXiv:2505.07818, 2025.


[16] Jiacheng Zhang, Jie Wu, Yuxi Ren, Xin Xia, Huafeng Kuang, Pan Xie, Jiashi Li, Xuefeng Xiao, Min Zheng, Lean Fu, et al. Unifl: Improve stable diffusion via unified feedback learning. arXiv preprint arXiv:2404.05595, 2024.


---


# Appendix


# A Contributions and Acknowledgments


All Authors of Seedance are listed in alphabetical order by their last names.


# Authors


Heyi Chen


Siyan Chen


Xin Chen


Yanfei Chen


Ying Chen


Zhuo Chen


Feng Cheng


Tianheng Cheng


Xinqi Cheng


Xuyan Chi


Jian Cong


Jing Cui


Qinpeng Cui


Qide Dong


Junliang Fan


Jing Fang


Zeta Fang


Chengjian Feng


Han Feng


Mingyuan Gao


Yu Gao


Dong Guo


Qiushan Guo


Boyang Hao


Hongxiang Hao


Qingkai Hao


Bibo He


Qian He


Tuyen Hoang


Ruoqing Hu


Xi Hu


Weilin Huang


Zhaoyang Huang


Zhongyi Huang


Donglei Ji


Jianwen Jiang


Siqi Jiang


Yunpu Jiang


Zhuo Jiang


Ashley Kim


Jianan Kong


Zhichao Lai


Shanshan Lao


Yichong Leng


Ai Li


Feiya Li


Gen Li


Huixia Li


JiaShi Li


Liang Li


Ming Li


Shanshan Li


Tao Li


Xian Li


Xiaojie Li


Xiaoyang Li


Xingxing Li


Yameng Li


Yifu Li


Yiying Li


Chao Liang


Han Liang


Jianzhong Liang


Ying Liang


Zhiqiang Liang


Wang Liao


Yalin Liao


Heng Lin


Kengyu Lin


Shanchuan Lin


Xi Lin


Zhijie Lin


Feng Ling


Fangfang Liu


Gaohong Liu


Jiawei Liu


Jie Liu


Jihao Liu


Shouda Liu


Shu Liu


Sichao Liu


Songwei Liu


Xin Liu


Xue Liu


Yibo Liu


Zikun Liu


Zuxi Liu


Junlin Lyu


Lecheng Lyu


Qian Lyu


Han Mu


Xiaonan Nie


Jingzhe Ning


Xitong Pan


Yanghua Peng


Lianke Qin


Xueqiong Qu


Yuxi Ren


Kai Shen


Guang Shi


Lei Shi


Yan Song


Yinglong Song


Fan Sun


Li Sun


Renfei Sun


Yan Sun


Zeyu Sun


Wenjing Tang


Yaxue Tang


Zirui Tao


Feng Wang


Furui Wang


Jinran Wang


Junkai Wang


Ke Wang


Kexin Wang


Qingyi Wang


Rui Wang


Sen Wang


Shuai Wang


Tingru Wang


Weichen Wang


Xin Wang


Yanhui Wang


Yue Wang


Yuping Wang


Yuxuan Wang


Ziyu Wang


Guoqiang Wei


Wanru Wei


Di Wu


Guohong Wu


Hanjie Wu


Jian Wu


Jie Wu


Ruolan Wu


Xinglong Wu


Yonghui Wu


---


Ruiqi Xia


Liang Xiang


Fei Xiao


XueFeng Xiao


Pan Xie


Shangyi Xie


Shuang Xu


Jinlan Xue


Shen Yan


Bangbang Yang


Ceyuan Yang


Jiaqi Yang


Runkai Yang


Tao Yang


Yang Yang


Yihang Yang


ZhiXian Yang


Ziyan Yang


Songting Yao


Yifan Yao


Zilyu Ye


Bowen Yu


Jian Yu


Chujie Yuan


Linxiao Yuan


Sichun Zeng


Weihong Zeng


Xuejiao Zeng


Yan Zeng


Chuntao Zhang


Heng Zhang


Jingjie Zhang


Kuo Zhang


Liang Zhang


Liying Zhang


Manlin Zhang


Ting Zhang


Weida Zhang


Xiaohe Zhang


Xinyan Zhang


Yan Zhang


Yuan Zhang


Zixiang Zhang


Fengxuan Zhao


Huating Zhao


Yang Zhao


Hao Zheng


Jianbin Zheng


Xiaozheng Zheng


Yangyang Zheng


Yijie Zheng


Jiexin Zhou


Jiahui Zhu


Kuan Zhu


Shenhan Zhu


Wenjia Zhu


Benhui Zou


Feilong Zuo
