# Activities
## Tasks and the back stack
- task는 user가 interact하는 activities의 collection
- activities들은 back stack에 arrange된다.
  - 새로운 activity는 back stack에 추가되고 finish되면 stack으로부터 pop된다.

### [Lifecycle of a task and its back stack]
- user가 app launcher의 icon이나 홈 화면의 바로가기를 클릭하면 app의 task가 foreground로 나오게 된다.
  - 최근에 app을 사용한 적이 없어 task가 없다면 task를 생성하고 main activity가 stack의 root activity로써 app이 실행된다.
- new activity는 stack의 top에 push되고 이미 있던 activity는 stopped된 채로 stack에 남아 있는다.
- back action을 취하면 activity가 destroy되면서 stack에서 pop되고 이전 activity가 resume된다.
- stack의 activities는 절대 재정렬되지 않고 오직 push, pop만 된다.
- 계속 back action을 취하면 홈 스크린이 나올 때까지 pop 되고 stack에서 모든 activity가 remove되면 task는 남아있지 않다.

#### Back press behavior for root launcher activities
- Intent filter에서 ACTION_MAIN, CATEGORY_LAUNCHER로 정의한 activity가 Root launcher activity다.
  - app launcher에서 app을 시작할 때 entry point가 되는 unique한 activity로 task를 시작할 때 사용된다.
