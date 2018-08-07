---
title: RDKit으로 분자의 Conformer 다루기

tags:
  - chemistry
  - rdkik
---

RDKit으로 분자의 3차원 구조를 계산하기 위해 conformer optimize 하는 방법 정리.

## 라이브러리
RDKit은 documentation이 불친절해서 어느 패키지에 뭐가 있는지 알기 어려운 편이다. 아래는 필요한 패키지들과 용도
* Chem : 기본적인 rdkit.mol을 다루는 메소드들을 담고 있음
* Chem.AllChem : Optimization algorithm 들을 담고 있음

~~~
from rdkit import Chem
from rdkit.Chem import AllChem
~~~

## 분자 만들기
SMILES를 받아서 rdkit.mol을 만들자.

~~~
# 예시 분자는 benzene
smi = "C1ccccC1"
mol = Chem.MolFromSmiles(smi)
mol = Chem.AddHs(mol)  # Energy 계산을 위해선 H가 포함된 것이 더 정확하다
~~~

## Conformer optimize
RDKit은 크게 세 가지 방법을 제공한다.
* ETKDG [Landrum et al. 2015](https://doi.org/10.1021/acs.jcim.5b00654){:target="_blank"}
* UFF(Universal Force Field) [Rappe et al. 1992](https://doi.org/10.1021/ja00051a040){:target="_blank"}
* MMFF(Merck Molecular Force Field) [Halgren et al. 1996](https://doi.org/10.1002/(SICI)1096-987X(199604)17:5/6%3C490::AID-JCC1%3E3.0.CO;2-P){:target="_blank"}

이중 UFF와 MMFF는 메소드 이름만 다르고 방법은 똑같다. ETKDG가 가장 속도가 빠르며 UFF랑 MMFF는 비슷. ETKDG 논문을 참조.

~~~
# ETKDG Method
status = AllChem.EmbedMolecule(mol, AllChem.ETKDG())

# status = 0 이면 success, 나머진 failure

conf = mol.GetConformer()
~~~
~~~
# UFF / MMFF Method
AllChem.EmbedMultipleConfs(mol, 10)  # 원하는 수만큼 conformer pool 생성
li = AllChem.UFFOptimizeMoleculeConfs(mol, maxIters=2000)  # UFF의 경우
li = AllChem.MMFFOptimizeMoleculeConfs(mol, maxIters=2000)  # MMFF의 경우

# li는 tuple의 list로 [(status, energy)] 형태임. status = 0는 converged, 1은 need more iter, -1은 failure
# 이중 가장 energy가 낮은 index를 고르면 된다

conf = mol.GetConformer(idx)
~~~

## 원자의 3차원 좌표 얻기
가장 안정한 conformer를 찾으면 개별 원자의 3차원 좌표를 얻을 수 있다.

~~~
position = conf.GetAtomPosition(atom_idx)
x, y, z = position.x, position.y, position.z
~~~