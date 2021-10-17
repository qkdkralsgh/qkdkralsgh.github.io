---
layout: single
title: "[논문리뷰] Faster R-CNN"
categories: Deep-learning
tag: [Deep-learning, DL]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---
안녕하세요! 이번 포스팅 내용은 Faster R-CNN 논문리뷰입니다.


# Faster R-CNN

### Intro

- R-CNN에서는 3가지 모듈 (region proposal, classification, bounding box regression)을 각각 따로따로 수행한다.
  **(1)** region proposal 추출 → 각 region proposal별로 CNN 연산 → **(2)** classification, **(3)** bounding box regression
- Fast R-CNN에서는 region proposal을 CNN level로 통과시켜 classification, bounding box regression을 하나로 묶었다.
  **(1)** region proposal 추출 → 전체 image CNN 연산 → RoI projection, RoI Pooling → **(2)** classification, bounding box regression 
- 그러나 여전히 region proposal인 Selective search알고리즘을 **CNN외부에서 연산**하므로 RoI 생성단계가 병목이다.
  따라서 **Faster R-CNN**에서는 **detection에서 쓰인 conv feature을 RPN에서도 공유**해서
  **RoI생성역시 CNN level에서 수행**하여 속도를 향상시킨다.
  
  **Region Proposal도 Selective search 쓰지말고 CNN - (classification / bounding box regression) 이 네트워크 안에서 같이 해보자!**


---

### Faster R-CNN

- Selective search가 느린이유는 cpu에서 돌기 때문이다.
  따라서 Region proposal 생성하는 네트워크도 gpu에 넣기 위해서 Conv layer에서 생성하도록 하자는게 아이디어이다.

- Faster R-CNN은 한마디로 RPN + Fast R-CNN이라할 수 있다.
  Faster R-CNN은 Fast R-CNN구조에서 conv feature map과 RoI Pooling사이에 RoI를 생성하는
  Region Proposal Network가 추가된 구조이다.

<center><img src="/assets/images/Faster_R-CNN1.png" width="50%" height="50%"></center>

그리고 Faster R-CNN에서는 RPN 네트워크에서 사용할 CNN과
Fast R-CNN에서 classification, bbox regression을 위해 사용한 CNN 네트워크를 공유하자는 개념에서 나왔다.
    

<center><img src="/assets/images/Faster_R-CNN2.png"></center>


결국 위 그림에서와 같이 CNN을 통과하여 생성된 conv feature map이 RPN에 의해 RoI를 생성한다.

주의해야할 것이 생성된 RoI는 feature map에서의 RoI가 아닌 original image에서의 RoI이다.

(그래서 코드 상에서도 anchor box의 scale은 original image 크기에 맞춰서 (128, 256, 512)와 같이 생성하고 이 anchor box와 network의 output 값 사이의 loss를 optimize하도록 훈련시킨다.)

따라서 original image위에서 생성된 RoI는 아래 그림과 같이 conv feature map의 크기에 맞게 rescaling된다.

    
<center><img src="/assets/images/Faster_R-CNN3.png" width="50%" height="50%"></center>

    
**<center>feature map에 투영된 RoI</center>**

    
이렇게 feature map에 RoI가 투영되고 나면 FC layer에 의해 classification과 bbox regression이 수행된다.

    
<center><img src="/assets/images/Faster_R-CNN1.png" width="70%" height="70%"></center>
    
    
위 그림에서 보다시피 마지막에 FC layer를 사용하기에 input size를 맞춰주기 위해 RoI pooling을 사용한다.

RoI pooling을 사용하니까 RoI들의 size가 달라도 되는것처럼 original image의 input size도 달라도된다.

그러나 구현할때 코드를 보면 original image의 size는 같은 크기로 맞춰주는데 그 이유는 **"vgg의 경우 244x224, resNet의 경우 min : 600, max : 1024 등으로 맞춰줄때 성능이 가장 좋기 때문이다"** original image를 resize할때 손실되는 data가 존재하듯이 **feature map을 RoI pooling에서 max pooling을 통해 resize할때 손실되는 data 역시 존재한다.** 

따라서 **이때 손실되는 data와 input image 자체를 resize할때 손실되는 data 사이의 Trade off** 가 각각 vgg의 경우 224x224, resNet은 600~1024이기에 input size를 고정시킨 것이다.
