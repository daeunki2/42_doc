# Project Guide

## 프로젝트의 목표

N-puzzle 프로젝트의 목표는 주어진 퍼즐의 초기 상태에서 시작하여
**Snail Solution 형태의 목표 상태까지 도달하는 이동 경로를 프로그램으로 찾는 것**이다.

이를 위해 **A\* (A-star) Search Algorithm 또는 그 변형**을 구현해야 한다.

퍼즐은 `N × N` 크기의 정사각형 보드이며,

- 하나의 빈 공간
- `1`부터 시작하는 서로 다른 숫자 타일

로 구성된다.

가능한 이동은 단 하나다.

> 빈 공간과 상하좌우로 인접한 타일 하나를 서로 교환한다.

대각선 이동은 허용되지 않는다.

즉 프로그램이 해결해야 하는 문제는 다음과 같다.

```text
Initial State
      │
      ▼
Valid Moves
      │
      ▼
Search
      │
      ▼
Snail Goal
```

Subject에서는 작은 `3 × 3` 퍼즐조차 해결하는 데 수 초씩 걸리는 구현은
합리적인 성능으로 보지 않는다.

---

# 1. 전체 요구사항

프로그램은 크게 다음 기능을 제공해야 한다.

```text
Input
  │
  ├── Puzzle File
  │
  └── Random Puzzle
  │
  ▼
Parsing
  │
  ▼
Validation
  │
  ▼
Solvability Check
  │
  ├── Unsolvable ──► 사용자에게 알리고 종료
  │
  ▼
Goal Generation
  │
  ▼
A* Search
  │
  ├── Heuristic 1
  ├── Heuristic 2
  └── Heuristic 3+
  │
  ▼
Solution
  │
  ├── Complexity in Time
  ├── Complexity in Size
  ├── Number of Moves
  └── Ordered Sequence of States
```

---

# 2. 언어 선택

구현 언어와 컴파일러는 자유롭게 선택할 수 있다.

단, Subject는 언어에 따라 시간 및 공간 효율이 달라질 수 있으며,
이 선택이 프로그램의 성능에 영향을 줄 수 있다는 점을 명시하고 있다.

또한 일반적인 rule을 가진 `Makefile`을 제공해야 한다.

따라서 이 과제에서는 단순히 정답을 찾는 것뿐 아니라
**탐색 과정에서 시간과 메모리를 어떻게 사용하는지도 중요하다.**

---

# 3. 다양한 크기의 Puzzle

프로그램은 `3 × 3`에만 맞춰 작성하면 안 된다.

Subject에서는 예시로 다음과 같은 크기를 언급한다.

```text
3
4
5
17
...
```

즉 다음과 같은 하드코딩은 피해야 한다.

```python
goal = [
    [1, 2, 3],
    [8, 0, 4],
    [7, 6, 5]
]
```

대신 Puzzle의 크기 `N`을 기준으로 보드와 Goal State를 처리할 수 있어야 한다.

```text
N
│
├── N × N Board
│
├── N에 따른 Snail Goal
│
└── N에 따른 Search State
```

큰 Puzzle을 어디까지 현실적으로 해결할 수 있는지는
Search Algorithm, Heuristic, State 표현 및 자료구조의 효율에 영향을 받는다.

---

# 4. Input

프로그램은 두 종류의 시작 상태를 처리해야 한다.

## Puzzle File

Subject Appendix에서는 다음과 같은 입력을 예시로 제공한다.

```text
# this is a comment
3

3 2 6 # another comment
1 4 0
8 7 5
```

첫 번째 유효한 값:

```text
3
```

은 Puzzle의 크기다.

그 다음 `3`개의 행이 실제 보드를 나타낸다.

```text
3 2 6
1 4 0
8 7 5
```

여기서:

```text
0
```

은 빈 공간을 의미한다.

---

## Comment

`#` 뒤의 내용은 Comment로 취급할 수 있다.

따라서 다음 두 형태 모두 처리할 수 있어야 한다.

```text
# comment

3

3 2 6
1 4 0
8 7 5
```

그리고:

```text
3
3 2 6 # comment
1 4 0
8 7 5
```

또한 Subject의 입력 예시는 숫자가 정렬되어 있지 않아도 허용되어야 함을 보여준다.

```text
0 10 5 7
11 14 4 8
1 2 6 13
12 3 15 9
```

즉 Parser는 시각적인 정렬에 의존해서는 안 된다.

---

# 5. Random Puzzle

파일 입력뿐 아니라 프로그램이 직접 생성한
**Random State**도 처리해야 한다.

따라서 전체 구조를 다음처럼 생각할 수 있다.

```text
               ┌── File Parser
Input State ───┤
               └── Random Generator
                       │
                       ▼
                  Puzzle State
```

어떤 방식으로 들어온 Puzzle이든 이후 Search에서는 동일한 State 표현으로
처리할 수 있도록 설계하는 것이 자연스럽다.

---

# 6. Puzzle Validation

Subject는 입력 파일의 예시는 제공하지만,
모든 잘못된 입력 사례를 하나씩 정의하지는 않는다.

따라서 구현에서는 최소한 Puzzle State로 사용할 수 있는 입력인지 검증할 필요가 있다.

예를 들어 `N × N` 퍼즐이라면 다음과 같은 조건을 확인할 수 있다.

```text
Board 크기 = N × N

필요한 값:
0 ... N² - 1
```

따라서 Parser / Validator에서는 일반적으로 다음과 같은 상황을 검사하게 된다.

```text
잘못된 size
행 개수 오류
각 행의 길이 오류
숫자가 아닌 값
범위를 벗어난 값
중복된 숫자
필요한 숫자의 누락
```

여기서 중요한 것은 두 종류의 검사를 구분하는 것이다.

```text
Valid
  │
  │ 입력 형식과 Puzzle 구성이 올바른가?
  ▼
Solvable
  │
  │ Goal까지 실제로 도달 가능한가?
  ▼
Search
```

**Valid Puzzle이라고 해서 반드시 Solvable Puzzle인 것은 아니다.**

---

# 7. Goal State — Snail Solution

이 프로젝트의 Goal은 일반적인 오름차순 Goal이 아니라
**Snail Solution**이다.

`3 × 3`의 경우:

```text
1 2 3
8 0 4
7 6 5
```

숫자가 바깥쪽에서 시작하여 시계 방향으로 안쪽으로 들어가는 형태다.

크기가 커지면 같은 규칙을 반복한다.

개념적으로:

```text
→ → → ↓
↑     ↓
↑     ↓
← ← ← ↓
```

와 같은 방향으로 숫자를 채워 나간다.

따라서 Goal State 역시 `N`에 따라 생성할 수 있어야 한다.

```text
N
│
▼
Snail Goal Generator
│
▼
Goal State
```

---

# 8. Solvability

입력된 Puzzle은 풀 수 없는 상태일 수도 있다.

Subject에서는 이러한 경우:

> 사용자에게 Puzzle이 풀 수 없다는 사실을 알리고 종료

하도록 요구한다.

따라서 Search를 무조건 실행하는 구조보다는 다음과 같은 흐름이 필요하다.

```text
Puzzle
   │
   ▼
Valid?
   │
   ▼
Solvable?
   │
   ├── NO ──► Inform User ──► Exit
   │
   └── YES
        │
        ▼
      Search
```

Solvability를 어떤 원리로 판단하는지는 별도의 Puzzle Representation 페이지에서 자세히 다룬다.

---

# 9. Search Algorithm

Mandatory Part에서는 **A\* Search Algorithm 또는 그 변형**을 구현해야 한다.

A*의 기본적인 평가 구조는 다음과 같다.

```text
f(n) = g(n) + h(n)
```

각 이동의 Transition Cost는 Subject에서 항상 `1`로 정의되어 있다.

따라서 한 번 이동할 때마다:

```text
g(next) = g(current) + 1
```

이 된다.

Search Algorithm의 구체적인 동작과 `opened`, `closed` 등의 구조는
Search Algorithms 페이지에서 자세히 다룬다.

---

# 10. Heuristic

사용자는 최소 **3개의 관련성 있는 Heuristic Function** 중 하나를 선택할 수 있어야 한다.

그중 하나는 반드시:

**Manhattan Distance**

여야 한다.

나머지 두 개 이상은 자유롭게 선택할 수 있다.

하지만 Subject에는 중요한 조건이 있다.

> Heuristic은 admissible해야 한다.

즉 단순히 아무 숫자나 반환하는 함수는 Heuristic으로 인정되지 않는다.

전체적으로는 다음과 같은 구조가 가능하다.

```text
                 ┌── Manhattan Distance
                 │
User Selection ──┼── Heuristic 2
                 │
                 └── Heuristic 3
                         │
                         ▼
                       A*
```

각 Heuristic의 계산 방법과 **왜 admissible한지**는
Heuristics 페이지에서 별도로 다룬다.

---

# 11. Search 결과

Goal을 찾았다고 프로그램이 바로 끝나면 안 된다.

Subject에서는 Search 종료 후 여러 정보를 출력하도록 요구한다.

## Complexity in Time

Search 과정에서 `opened` set으로부터 선택된 State의 총 개수.

즉 대략적으로:

```text
Search가 실제로 얼마나 많은 State를 선택하여 탐색했는가?
```

를 나타낸다.

---

## Complexity in Size

Search 중 **동시에 메모리에 표현되어 있던 State의 최대 개수**.

즉:

```text
Search 과정에서 최대 어느 정도의 State를 메모리에 유지했는가?
```

를 나타낸다.

---

## Number of Moves

Initial State에서 Goal State까지 도달하는 데 필요한 이동 횟수를 출력해야 한다.

예:

```text
Number of moves: 14
```

---

## Solution Path

최종적으로 찾은 State들을 순서대로 출력해야 한다.

예를 들어:

```text
Initial
   │
   ▼
State 1
   │
   ▼
State 2
   │
   ▼
State 3
   │
   ▼
Goal
```

따라서 Search 과정에서는 Goal을 발견하는 것뿐 아니라
**어떤 State를 통해 현재 State에 도달했는지 추적할 수 있어야 한다.**

---

# 12. Mandatory 요구사항 정리

최소 구현 범위를 정리하면 다음과 같다.

| 기능 | 필수 |
|---|---|
| 다양한 N 크기 처리 | O |
| File Input | O |
| Random State | O |
| Snail Goal | O |
| Solvability 판단 | O |
| A* 또는 변형 | O |
| Transition Cost = 1 | O |
| Manhattan Distance | O |
| 추가 Heuristic 최소 2개 | O |
| Admissible Heuristic | O |
| Complexity in Time 출력 | O |
| Complexity in Size 출력 | O |
| Move Count 출력 | O |
| Solution State Sequence 출력 | O |
| Makefile | O |

---

# 13. Bonus

Mandatory Part가 완벽하게 동작하는 경우에만 Bonus가 평가된다.

Bonus에서는 `g(x)`와 `h(x)`를 조절하여 같은 프로그램을:

```text
Uniform-cost Search

Greedy Search
```

방식으로 실행할 수 있도록 한다.

A*가:

```text
f(n) = g(n) + h(n)
```

이라면 `g`와 `h`의 사용 방식을 변경하면서 서로 다른 Search Strategy를 구성할 수 있다는 것이 핵심이다.

이 부분은 단순한 기능 추가라기보다,

> A*, Uniform-cost, Greedy Search가 무엇을 기준으로 다음 State를 선택하는가?

를 비교하기 위한 Bonus라고 볼 수 있다.

---

# 14. 평가에서 설명해야 하는 것

Subject는 Defense에서 단순히 프로그램 실행만 보는 것이 아니라
**구현 선택을 설명할 수 있어야 한다고 명시한다.**

특히 다음 항목을 준비해야 한다.

### Search Algorithm

어떤 A* Variant를 구현했는가?

왜 그 방식을 선택했는가?

---

### Heuristic

어떤 Heuristic을 선택했는가?

왜 해당 Heuristic이 **admissible**한가?

---

### Data Structure

왜 해당 자료구조를 선택했는가?

예를 들어 Search 구현에서는 다음과 같은 선택이 생길 수 있다.

```text
Open Set
Closed Set
Priority Queue
State Representation
Parent Tracking
```

각 구조가 Search의 시간 및 메모리 사용에 어떤 영향을 주는지 설명할 수 있어야 한다.

---

### 다양한 크기의 Puzzle

평가 시 하나의 `3 × 3` 예제만 보여주는 것이 아니라
여러 크기의 Puzzle에서 프로그램이 정상적으로 동작하는 예제를 준비해야 한다.

---

# 구현 관점에서 다시 보기

Subject 전체를 실제 개발 작업으로 바꾸면 다음과 같이 나눌 수 있다.

```text
1. Parser
      │
      ▼
2. Validation
      │
      ▼
3. Puzzle Representation
      │
      ▼
4. Snail Goal Generator
      │
      ▼
5. Move Generation
      │
      ▼
6. Solvability
      │
      ▼
7. Heuristics
      │
      ▼
8. A* Search
      │
      ▼
9. Path Reconstruction
      │
      ▼
10. Complexity Measurement
      │
      ▼
11. Output
```

즉 이 과제에서 최종적으로 만드는 것은 단순한 Puzzle UI가 아니다.

핵심은:

> **N-puzzle을 State-space Search 문제로 표현하고, admissible heuristic을 사용하는 A* 계열 알고리즘으로 해결한 뒤 그 탐색 비용과 Solution Path를 보여주는 Solver를 구현하는 것**

이다.