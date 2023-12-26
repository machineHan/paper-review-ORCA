# paper-review-ORCA

## abstract
Pre trained model을 fine-tuning하여 사용하는 것은 잘 개척된 분야에서는 아주 큰 성공을 거뒀다. 하지만 pretrain model이 적은 분야에서는 큰 이점이 없다. 
>ORCA는 cross-modal fine-tuning framework으로 pretrained model을 다른 modal로 바꾸는 어플리케이션이다.

Align-then-refine workflow를 따른다.

우선 ORCA는 인풋 모달을 pretrained model의 modal에 맞게 embedded feature distribution을 만든다
그리고 pretrained model은 만들어진 embedded feature distribution을 가지고 fine-tuning한다.

## Introduction
Pretrained model을 fine-tuning하여 사용하는 것은 language, vision, speech등 여러분야에서 뛰어난 성능을 거뒀다. 그리고 이 transfer learning은 모두 동일 modal내에서의 전이만을 다루고 있다.

> 그렇다면 different modal transfer, 즉 NLP에서 잘쓰여지는 BERT를 이미지 분류에서 사용하면 어떻게 될까?
>또는 vision tranformer를 speech recognition에 사용하면 어떻게 될까?

실제로 이러한 시도는 학습하기 어려운 분야에 대한 시도에서 처음 이뤄졌다.
Physical, life science, healthcare, finance 가 그러한 분야이다. 왜 학습하기 어렵냐, ML 전문지식과 해당 분야의 전문지식 둘다 요하기 때문이다.

이러한 문제에 대해 AutoML이라는 간단한 기술이 있긴하지만, from scratch 부터 모델을 훈련해야한다. 
충분한 데이터로 훈련된 분야의 pretrained model을  덜 개척된 분야에 사용하는 것은 충분히 잠재력이 있다. 

기존에 제안된 cross modal transfer에 대한 기술이 있긴하지만, 전혀 일반적이지 않고 ad-hoc하며 추가적인 작업이 많이 요하다. 게다가 좋은 성능도 아니다. 
>우리는 일반적이며 성능도 잘 나오는 cross-modal fine-tuning framework를 만들어보겠다.

키 아이디어는 task agnostic fune-tuning 전에 인풋데이터를 task-specific data alignment를 하는것이다.
이렇게 하면 B모달이 A모달로 훈련된 모델의 가중치를 왜곡없이 잘 쓸 수 있다. 즉 학습된 A modal feature를 embedded feature of B가 사용 가능하다.

    ORCA는 총 3단계로 간추릴 수 있다.
    1. target input을 body에서 사용할 수 있도록 메핑시키는 Embedding network 
    2. embedding network는 embedded target이랑 source reference와의 distributional distance가 최소화 되로록 훈련한다.
    3. Fine-tuned entire target model

ORCA의 휴율성을 너비(다양한 환경에서의 성능 = generality), 깊이(특정 테스크에서의 성능 = competitive performance), 기존의 있던 다른 기술과의 비교로 판별하겠다.


##ORCA Workflow

타겟 도메인 : 우리가 적용하고싶은 테스크
소스 도메인 : 해결을 위해 사용할 테스크
>Ex) BERT모델을 이용하여 (소스 도메인) 음성 인지(타겟 도메인)를 작업하겠다.
소스 모델 : pretrained model using source dataset
타겟 모델 : entire model set = embedder + body(pretrained source model) + predictor


타겟과 소스는 feature space, label space, probability distribution 까지 너무 다르다. 자명한 얘기

우리는 소스모델 기반으로 타겟 모델을 만들 것이다. 즉 소스 데이터로 기반으로 만들어진 타겟 모델을 최적화하는데 타겟데이터로 접근할 것이다.

타겟모델의 타겟 로스 식을 보면 명시적으로 source data에 대한 내용은 없이 target data만으로 이뤄져 있지만 확실히 in-modal, cross-modal transfer가 포함되어있다. 하지만 수학적인 두개의 차이점을 찾기 어려운 점에서, 직관적으로 따지면 cross-modal data가 in-modal데이터에 비해 차이가 있을 것이다.


### Dimensionality Alignment

ORCA는 크게 3부분으로 나뉜다. Embedder, body, predictor.  하나하나 뜯어보겠다

Custom Embedding Network, Custom Prediction Head 
: 둘다 패드에 정리한거 참고


### Embedder Learning for Distribution Alignment

직관적으로, 비슷하지만 다른 모달로 바꾸는것은 쉬워보인다. 당연히 완전히 다른 모달끼리의 변환도 어려워 보인다.

우리는 그래서 소스 모델(바디) 에 들어가기전에 최대한 embedder를 pretrain 하겠다.
어떤 식으로?  Embedded target feature랑 source featrue가 최대한 같게한다 == 소스와 타겟의 유사도를 확립한다
타겟 데이터 + 라벨과 소스 데이터 + 라벨을 가지고 각각 joint distribution을 구하고 둘의 차이를 최소화하는 식으로 학습

이 joint distribution distance를 측정하는 방식이 다양하지만 다다음 섹션에서 다루겠다.


### Weight Refining for Downstream Adaptation	

임베더를 사전훈련하고, 이후 부터는 전체 모델(embedder, body, predictor)을 fine-tuning한다. 타겟 로스를 줄이는 방향으로 학습. 그냥 버트 모델 앞뒤로 embedder, predictors Layer가 붙은 모델이라고 생각하면 편함. BERT모델의 확장버전

embedder와 predictor는 body와 일치시키도록 훈련한다 .

### Evaluation of Distribution Alignment Metrics

Embedder pertaining step 에서 모델 훈련시 
minimize the distance between the joint distribution of the target embeddings and source embedding 방향으로 훈련한다.

3개의 matrix의 distribution을 측정하는 방법을 3가지 소개한다.

    1. Pairwise Euclidean distance : 단순히 거리와 범위만을 측정해서 계산
    2. Moment-based maximum mean discrepancy (MMD) : feature mean을 활용하여 계산
    3. Optimal transport dataset distance (OTDD) : feature + lable을 활용하여 계산

OTDD가 제일좋음. “아마” embedder align에서 라벨정보가 사용되는 방식일 것이다.
자세한 식을 알고 싶으면 OTDD에 관한 논문을 읽어도록

OTDD는 타겟과 소스에 대해 각 클래스 라벨을 in-class feature로 나타낸다. 이러하면 둘이 공유 공간에서 나타나개 되고 이로인해 다른 라벨을 가지고 있는 타겟, 소스간에 distribution distance를 측정할수 있게됐다.
게다가 OTDD는 개별 데이터포인트를 정렬하는것 말고도 같은 라벨을 가진 feature들을 그룹화한다. 이거 떄문에 fine-tuning에 크게 기여한다.

근데 OTDD는 계산량이 너무 많다. 그래서 효과적인 근사치를 구하겠다.




우리는 최상의 alignment skill을 제공하는 것이 목표가 아니다. 일반적인 framework를 만드는 것이다.
