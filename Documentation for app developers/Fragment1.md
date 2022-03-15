# Fragments
## 참고자료
[Android Developers: Fragments](https://developer.android.com/guide/fragments)

## Overview
- Fragment는 app UI의 재사용 가능한 부분을 나타낸다.
- Fragment는 자체 layout을 정의 및 관리하고 자체 lifecycle을 갖고 자체 input events를 처리한다.
- Fragment는 독립적으로 존재할 수 없고 activity나. 다른 Fragment에 hosting되어야 한다.
  - Fragment의 view hierarchy는 host view hierarchy의 일부가 되거나 attach된다.

### [Modularity]
- Fragment는 activity의 UI를 discrete chunk로 나누면서 modularity, reusability를 제공한다.
- activity는 navigation drawer와 같은 global elements를, fragments는 UI의 single screen or portion of a screen을 정의하기에 적합하다.
