# ConcatAdapter
## 1. 정의
- androidx.recyclerview를 implement하면서 사용 가능
- RecyclerView에서 여러 개의 Adapter를 합쳐서 적용시킬 때 사용
- 기본적으로 Header, Content, Footer 등으로 구성
- item loading 중임을 나타낼 때 주로 사용
## 2. 사용 방법
#### 1) implement
```javascript
implementation "androidx.recyclerview:recyclerview:1.2.1"
```
#### 2) Footer, Header를 위한 ViewHolder(Footer, Header 구조 동일)
```javascript
class WordFooterViewHolder(binding: ItemWordfooterBinding): RecyclerView.ViewHolder(binding.root) 
```
#### 3) Footer, Header를 위한 Adapter(Footer, Header 구조 동일)
```javascript
class WordFooterAdapter: RecyclerView.Adapter<WordFooterViewHolder>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): WordFooterViewHolder {
        return WordFooterViewHolder(ItemWordfooterBinding.inflate(LayoutInflater.from(parent.context), parent, false))
    }

    override fun onBindViewHolder(holder: WordFooterViewHolder, position: Int) {

    }

    override fun getItemCount(): Int = 1
}
```
#### 4) Content ViewHolder
```javascript
class WordContentViewHolder(private val binding: ItemWordlistBinding): RecyclerView.ViewHolder(binding.root) {
    fun bind(word:String) {
        if (word.isNotBlank()) binding.word.text = word
    }
}
```
#### 5) Content Adapter
```javascript
class WordContentAdapter: RecyclerView.Adapter<WordContentViewHolder>() {
    val wordlist = arrayListOf<String>()
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): WordContentViewHolder {
        return WordContentViewHolder(ItemWordlistBinding.inflate(LayoutInflater.from(parent.context), parent, false))
    }

    override fun onBindViewHolder(holder: WordContentViewHolder, position: Int) {
        holder.bind(wordlist.get(position))
    }

    override fun getItemCount(): Int = wordlist.size
}
```
#### 6) Activity
```javascript
class MainActivity : AppCompatActivity() {
    lateinit var binding: ActivityMainBinding
    val wordContentAdapter = WordContentAdapter()
    val wordHeaderAdapter = WordHeaderAdapter()
    val wordFooterAdapter = WordFooterAdapter()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        wordContentAdapter.apply {
            wordlist.addAll(resources.getStringArray(R.array.arraylist))
        }

        binding.recyclerview.apply {
            layoutManager = LinearLayoutManager(this@MainActivity2)
            adapter = ConcatAdapter(wordHeaderAdapter, wordContentAdapter, wordFooterAdapter)
        }
    }
}
```
## 3. 추가 정리
- Activity에서처럼 ConcatAdapter() 안에 보이고 싶은 순서대로 Adapter를 설정하고 최종 adapter로 설정
- 현재 code에서는 view binding 사용
- view binding을 사용하지 않는다면 itemView 사용
```javascript
public ViewHolder(@NonNull View itemView) {
    if (itemView == null) {
        throw new IllegalArgumentException("itemView may not be null");
    }
    this.itemView = itemView;
}
```
## 4. view binding 사용하지 않은 Content ViewHolder, Adapter
#### 1-1) Content ViewHolder(no view binding)
```javascript
class WordContentViewHolder(private val view: View): RecyclerView.ViewHolder(view) {
    fun bind(word:String) {
        if (word.isNotBlank()) view.findViewById<TextView>(R.id.word).text = word
    }
}
```
#### 1-2) Content ViewHolder(no view binding)
```javascript
class WordContentViewHolder(view: View): RecyclerView.ViewHolder(view) {
    fun bind(word:String) {
        if (word.isNotBlank()) itemView.findViewById<TextView>(R.id.word).text = word
    }
}
```
#### 2) Content Adapter(no view binding)
```javascript
class WordContentAdapter: RecyclerView.Adapter<WordContentViewHolder>() {
    val wordlist = arrayListOf<String>()
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): WordContentViewHolder {
        return WordContentViewHolder(LayoutInflater.from(parent.context).inflate(R.layout.item_wordlist, parent, false))
    }

    override fun onBindViewHolder(holder: WordContentViewHolder, position: Int) {
        holder.bind(wordlist.get(position))
    }

    override fun getItemCount(): Int = wordlist.size
}
```
## 5. binding.inflate & View.inflate 통한 View 생성
#### 1) binding.inflate
```javascript
val view: View = Binding.flate(LayoutInflater, ViewGroup, Attached)
```
#### 2) View.inflate
```javascript
val view: View = LayoutInflater.inflate(R.layout.id, ViewGroup, Attached)
