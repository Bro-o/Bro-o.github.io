---
title: "캡스톤 - ImageNet 전처리 코드 작성 및 리뷰"
excerpt: "ImageNet 1/10로 줄이기, ResNet50 Maxpool 까지 진행한 텐서 추출"

categories:
    - Code
tags:
    - Capstone
    - Code Review
    

date: 2022-04-23
---

## 1. 필요한 이유
캡스톤을 진행하며 ResNet50 사이에 압축/복원 모델을 제작한다. 테스트를 하는데 사용할 데이터로 ImageNet을 선택했다. 그런데 ImageNet의 train이 약 120만개, valid가 5만개로 그 수가 너무 많아 학습 및 평가하는데 너무 많은 시간이 든다고 판단되어 1/10, 1/5 -> 12만개, 1만개로 줄이기로 결정.
또 ResNet50의 첫번째 Layer인 MaxPool까지 진행한 후 압축/복원 모델을 통과하고 나머지 Layer를 EdgeCPS에서 진행하므로 MaxPool까지 통과한 Tensor를 가지고 ResNet50 전체를 돌 필요 없이 압축/복원 모델만 빠르게 학습 및 평가를 진행한다.
![](\assets\images\Capstone_model.PNG)
### To Do
- ImageNet
train 데이터 개수 1/10 -> 12만개  
val 데이터 개수 1/5 -> 1만개  
- maxpool 통과한 텐서 pickle로 저장
하나의 이미지에 대해서 (tensor, labels) 튜플로 저장  

## 2. 내가 작성 한 Code
처음 작성한 스크립트이다.[2698e70](https://github.com/happysansam/Capstone/commit/2698e70479975144783f58843f3e2562b89403bc) 솔직하게 파이썬 코드를 코랩이나 주피터로 실행한 경험이 대부분이여서 하나의 스크립트로 짜는 부분에 있어서 매우 초보이다. 코드를 깔끔하게 짜는 방법을 배울 필요가 있다.
### 파일 열기 및 추출
```python
import re
title = []
with open(dir, 'r') as f:
    lines = f.readlines()

for line in lines:
    title.append(re.findall('n\d+', line))

title = sum(title, []) # 리스트 2차원 -> 1차원

```
파이썬에서 정규표현식을 사용하기 위해서는 re 모듈이 필요하다. findall 함수를 이용하여 n00000000처럼 되어있는 부분을 모두 찾기 위하여 'n/d+'라 작성함. n은 말그대로 문자, /d는 숫자 0~9, +는 앞의 숫자 여러개까지 찾는다는 의미이다. sum을 이용하면 리스트 2차원에서 1차원으로 바꿀 수 있다.

### 무작위로 선택
```python
np.random.choice(os.listdir(data+i),len(os.listdir(data+i))//10)
```
> np.random.choice(a, size, replace=True, p)

### Pickle로 저장
```python
for inputs, labels in train_loader:
	inputs = inputs.cuda()

	inputs = net(inputs)
	labels = labels.item()
	result = (inputs, labels)

	f = open('maxpool_tensor.pkl', 'ab')
	pickle.dump(result, f)
	f.close()
```
'ab'는 바이너리로 이어서 쓰겠다는 의미이다. dump로 피클 파일에 작성한다.

## 3. Code Review 수정 사항
처음으로 코드 리뷰를 받았다. 긴장되면서도 내가 부족한 부분이 많다는 점을 느꼈다. 아직 공부할게 많다! 아직 젊다고 생각하고 꾸준하게 해서 전문가가 되고 싶다.!
### 수정이 필요한 부분
[d2ea2a3](https://github.com/happysansam/Capstone/commit/d2ea2a3d69187452f52a02cbd7cce56259f37dae)
- 라이브러리 또는 모듈 호출을 전부 위에서 하기
- train된 ResNet50을 사용하여 Maxpool까지 진행하기
우리는 압축/복원 모델만 제작해서 학습 및 평가가 필요하기 때문에 ResNet부분은 이미 학습된 모델을 가져와 tensor를 통과시켜야함
> self.resnet_model = torchvision.models.resnet50(pretrained=True)
- 전부 함수로 작성하고 main에서 시작해서 호출하는 형식으로 짜기
- 하드코딩 된 부분 수정하기