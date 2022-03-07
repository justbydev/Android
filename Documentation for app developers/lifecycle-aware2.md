# Lifecycle-aware components 2
## Save UI states
- user expectation과 system behavior간의 gap을 줄이기 위해 ViewModel, onSaveInstanceState() method, local storage를 사용하여 UI state를 persist해야 한다.
  - 어떤 것을 사용할 지는 UI data의 complexity, app use cases, 메모리 사용량 대비 검색 속도를 고려한다.
  - 어떤 방법을 사용하든 user expectation을 만족해야 하며 매끄럽고 원활한 UI를 제공해야 한다.
  - 대부분의 경우 ViewModel과 onSaveInstanceState()를 모두 사용해야 한다.

### [User expectations and system behavior]
#### User-initiated UI state dismissal
- 다음과 같은 경우 activity르 완전히 dismiss하고 user 또한 clean state와 함께 activity가 시작되길 expect한다.
  - back button pressed
  - Recents screen에서 swiping
  - activity에서 상위 항목으로 이동
  - 설정 화면에서 app 종료
  - Activity.finish()에 의해 finishing

- 다음과 같은 상황에서는 activity instance가 destroy되고 memory에서 제거되며 저장된 state들도 remove 된다.
  - 이는 user expectation과 system behavior가 match된다.

#### System-initiated UI state dismissal
- configuration change(rotation or switching into multi-window mode)와 같은 상황에서 user는 activity's UI state가 유지되길 expect하지만 system은 activity를 destroy하고 저장된 UI state도 없앤다.
- 
