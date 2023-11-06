# Coisas que aprend em fazendo um desafio de código em Kotlin (Android)

Alguns dias atrás, eu fiz um desafio de código que consistia em fazer um jogo da velha.

Os requerimentos para o desafio envolviam,  de  maneira opcional, desenvolver para plataformas mobile, usando Kotlin, Swift ou Flutter, mas o desafio poderia ser feito usando o terminal.

Minha principal tecnologia em proficiência em java. A maioria dos meus projetos e dos meus estudos são orientados ao ecossistema Java, e profissionalmente eu trabalhei em sua maioria com React e Typescript. A escolha lógica aqui (e com menor risco) seria fazer o desafio no terminal com Java e Typescript, mas eu decidi ir pelo caminho de desenvolver para plataformas mobile, e aprender, no processo, Jetpack Compose e Desenvolvimento Android.

Eis aqui os meus aprendizados:

## O modelo de programação

O desenvolvimento moderno de Android usa a biblioteca Jetpack Compose para criar UI's. Como é isso?

É principalmente semelhante ao modelo React de fazer UI's.

O principal artefato que você produz para compor uma interface é um Composable. Composables são funções marcadas com a anotação `@Composable`, que podem retornar um `Unit` (que é algo parecido com void) ou podem retornar algum valor, mas a maioria dos composables retorna apenas `Unit`.

Mas como é semelhante ao React se o React retorna objetos (por trás do JSX)?

A DSL é muito clara nesse aspecto. Composables são funções que constroem a interface, onde você usa composição, compondo-a junto com outros composables.

Aqui está um exemplo de um Composable "dummy":

 ```kotlin
@Composable
fun FormSection(
    @StringRes title: Int,
    modifier: Modifier = Modifier,
    children: @Composable (() -> Unit) = {},
) {
    Column(modifier = modifier, horizontalAlignment = Alignment.CenterHorizontally) {
        Text(
            text = stringResource(title),
            fontWeight = FontWeight.Bold,
            fontSize = TextUnit(20.0F, TextUnitType.Sp),
            textAlign = TextAlign.Center
        )
        Spacer(modifier = Modifier.height(5.dp))
        children()
    }
}
```

Este composable é dummy porque ele não tem nenhum estado ou efeito nele. Ele aceita uma função,  similar ao `children` no React que aceita outro composable.

## "Hooks"

No Jetpack Compose, nós não temos nenhum tipo de hook, mas temos construtos que agem de maneira similar aos hooks.

Explicitamente, o modelo de programação é construído com base em observables. Observables são baseados no pattern Observer, onde o objeto Observable tem uma série de subscribers, que consistem em callbacks. Quandoo o observable recebe alguma atualização (ou algum valor), ele vai notificar todos os subscribers, invocando seus respectivos callbacks.

O `useState` do Jetpack age exatamente dessa forma. Usamos delegação para a função `remember`, que possui um callback que recebe um `mutableStateOf<T>`.

```kotlin
var someState by remember { mutableStateOf(0) }
```

Mas, uma coisa importante de se ver aqui é que o subscriber principal desse observer é o próprio Composable. Quando o observer recebe algum valor, o Composable é invocado novamente num processo chamado **recomposition**. É um processo bem parecido com o **reconciliation** do React.

Outra coisa interessante são `LaunchedEffect`s. Eles são de fato o `useEffect` do Jetpack. Eles são muito bons para rodar alguma rotina de maneira assíncrona, como um timeout de uma corrotina para poder fechar algum modal:

```kotlin
LaunchedEffect(key1 = "Popup") {
  // this is a suspend function call 
  viewModel.makeRobotPlay()
}
```

A melhor parte disso é que, diferente do React, `LaunchedEffect`s podem rodar em qualquer parte do lifecycle do Composable. Então, podemos envolver esse componente em um condicional para rodar apenas em determinadas condições:

```kotlin
 if (openRobotPlayingTime) {
   LaunchedEffect(key1 = "Popup") {
     viewModel.makeRobotPlay()
   }
   AlertDialog(
     onDismissRequest = { /*TODO*/ }, 
     confirmButton = { /*TODO*/ }, 
     text = {
       Column(
         verticalArrangement = Arrangement.spacedBy(10.dp)
       ) {
         Text(
           text = stringResource(R.string.robot_time_playing_popup_title),
           style = MaterialTheme.typography.titleLarge
         )
         Text(
           text = stringResource(R.string.robot_time_playing_popup_content))
      }
   })
}
```
Todos os composables aqui podem ser invocados em qualquer ordem.

## Corrotinas

Corrotinas são uma parte central de programção em Kotlin. Aqui eles tem uma utilidade central.

Normalmente você pode usar corroitnas para qualquer necessidade assíncrona, mas lembre-se que corrotinas são funções coloridas (assim como Promises). Uma corrotina pode rodar apenas em:

*  Um `launch` callback de um `CoroutineScope`
*  Um `LaunchedEffect`
*  Uma função marcada com `suspend`

Para lidar com isso em um modelo de programação funcional e síncrono, Jetpack Compose disponibiliza Programação Funcional baseada em observables.
Eu aprendi da pior forma que certos observables não devem ser usados em alguns tipos de `CoroutineScope` (geralmente associados com persistência).
Esses observables podem ser usados para gerenciar o estado na aplicação.

## View Model

O view model é onde nós centralizamos o estado da aplicação para gerenciar estados mais complexos em Data Objects simples. Então, nós vamos envolver esses objetos em um `StateFlow` e fazer atualizações progressivas nesses objetos. Toda função e composable que "ouvir" o `StateFlow` vai ser notificado e atualizado.

É muito simples e pode ser feito em três passos:

1. Crie um `data class` que irá representar o estado da aplicação
2. 
```kotlin
data class GameState(
    val player1Name: String? = null,
    val player2Name: String? = null,
    val isRobotEnabled: Boolean = false,
    val tableSize: Int = 3,
    val gameTable: Array<Array<CellStates>>? = null,
    val playerTime: PlayerTime = PlayerTime.Player1,
    val winner: Winner = Winner.NoWinner
)
```

2. Crie o seu view model baseado nesse `data class`. O objeto deve ser envolvido num observable do tipo `StateFlow` e deve ser exposto com a função `.asStateFlow()`.


```kotlin
class GlobalStateViewModel(db: AppDatabase) : ViewModel() {
    private val _uiState = MutableStateFlow(GameState())
    val uiState = _uiState.asStateFlow()
}
```

3. Subscreva ao view model no Composable usando delegação:

```kotlin
@Composable
fun GameScreen(
    viewModel: GlobalStateViewModel, modifier: Modifier = Modifier,
    onNewGame: () -> Unit
) {
    val globalUiState by viewModel.uiState.collectAsState()
}
```

Para a atualizar os dados nesse view model, Apenas invoque o método `update` no seu `_uiState`. A atualização deve ser exposta como um método do viewModel:

```kotlin
fun makeUpdate{
  _uiState.update {
    it.copy(
     // update the components of the data class here
    )  
  }
}
```

## Acesso e Persistência de Dados

Acesso e Persistência de Dados podem ser feitos com a biblioteca Room.
Eu descreveria essa biblioteca como um Spring Data, porém sem esteróides.
Usamos classes e interfaces anotadas para modelar o acesso aos dados e as entidades do nível de persistência da aplicação.

```kotlin
@Entity 
data class User(@PrimaryKey(autoGenerated = true) val id: Long?, @ColumnInfo(name="name) val name: String?)

@Dao 
interface UserDao {
  @Insert 
  fun create(user: User): Long?
}
```

Por fim, nós definimos nosso banco de dados com uma classe abstrata que expõe a criação do nosso DAO..

```kotlin
@Database(version = 1, entities=[User::class])
abstract class AppDatabase :  RoomDatabase() {
  abstract fun userDao(): UserDao
}
```

E é isso!
Apenas instancia o banco de dados no sue código e você está pronto para usar!

```kotlin
 val db = Room.databaseBuilder(
            applicationContext,
            AppDatabase::class.java, "tic-tac-toe"
).build()
```

Inicialmente, quando usei o banco de dados, minha aplicação travou. Isto é porque o banco de dados não pode ser acessado na thread principal, para prevenir que a UI seja bloqueada.

Por trás dos panos, Room é apenas um invólucro ao redor do SQLite. O banco de dados é apeans um arquivo e a interação com esse arquivo é feito através de algumas funções em C.

Então, como vamos acessar o banco de dados sem bloquear a UI?

Podemos usar uma corrotina com o dispatcher `Dispatchers.IO`, dessa maneira:

```kotlin
viewModelScope.launch(Dispatchers.IO) {
  dao.create(user)
}
```

Eu acho que uma boa maneira de abstrair o acesso ao banco de dados é usando o viewModel.

## Conclusão

Essa foi uma ótima experiência e eu aprendi muito!

O código final para esse desafio pode ser encontrado aqui:

https://github.com/EronAlves1996/TicTacToe

Versão em en-US: https://dev.to/eronalves1996/things-that-ive-learned-doing-a-coding-challenge-in-kotlin-android-4o4m
