# Mobx best practices

## Usar uma RootStore

Para acessar uma store através de outra store uma boa prática é usar uma "RootStore". Tal recurso permite que todas as unidades de estado do Mobx compartilhem um escopo comum, possibilitando acesso entre elas:

```
class RootStore {
    constructor() {
        this.userStore = new UserStore(this);
        this.todoStore = new TodoStore(this);
    }
}

class UserStore {
    constructor(rootStore) {
        this.rootStore = rootStore
    }

    getTodos(user) {
        return this.rootStore.todoStore.todos.filter(todo => todo.author === user)
    }
}

class TodoStore {
    @observable todos = []

    constructor(rootStore) {
        this.rootStore = rootStore
    }
}
```

Usar RootStores também é benéfico para testes pois desta forma podemos resetar todas stores de uma única vez.

## Utilizar Models

Models são os elementos tratados por uma store. Por exemplo: se temos uma store chamada `TodoStore` os elementos dela são `Todo` e esse é o model. 

```
import {observable, autorun} from 'mobx';
import uuid from 'node-uuid';

export class TodoStore {
    @observable todos = [];
    @observable isLoading = true;

    loadTodos() {
        this.isLoading = true;
        this.transportLayer.fetchTodos().then(fetchedTodos => {
            fetchedTodos.forEach(json => this.updateTodoFromServer(json));
            this.isLoading = false;
        });
    }

    updateTodoFromServer(json) {
        var todo = this.todos.find(todo => todo.id === json.id);
        if (!todo) {
            todo = new Todo(this, json.id);
            this.todos.push(todo);
        }
        if (json.isDeleted) {
            this.removeTodo(todo);
        } else {
            todo.updateFromJson(json);
        }
    }

    createTodo() {
        var todo = new Todo(this);
        this.todos.push(todo);
        return todo;
    }

    removeTodo(todo) {
        this.todos.splice(this.todos.indexOf(todo), 1);
        todo.dispose();
    }
}

export class Todo {
    id = null;
    @observable completed = false;
    @observable task = "";
    @observable author = null;
    store = null;
    autoSave = true;
    saveHandler = null;

    constructor(store, id=uuid.v4()) {
        this.store = store;
        this.id = id;

        this.saveHandler = reaction(
            () => this.asJson,
            (json) => {
                if (this.autoSave) {
                    this.store.transportLayer.saveTodo(json);
                }
            }
        );
    }

    delete() {
        this.store.transportLayer.deleteTodo(this.id);
        this.store.removeTodo(this);
    }

    @computed get asJson() {
        return {
            id: this.id,
            completed: this.completed,
            task: this.task,
            authorId: this.author ? this.author.id : null
        };
    }

    updateFromJson(json) {
        this.autoSave = false;
        this.completed = json.completed;
        this.task = json.task;
        this.author = this.store.authorStore.resolveAuthor(json.authorId);
        this.autoSave = true;
    }

    dispose() {
        // clean up the observer
        this.saveHandler();
    }
}
```

Como é possível ver no exemplo acima, o autor é uma referência à um objeto "Author" presente em outra store, isso permite que não ocupemos dois espaços de memória pra mesma informação. Permite também que o estado do autor esteja atualizado em toda aplicação caso ele seja modificado na "AuthorStore".

Um outro jeito de fazer essa "serialização" e "deserialização" de dados no Mobx é utilizando: [serializr](https://github.com/mobxjs/serializr), ela gerencia esses "relacionamentos" automaticamente.


## Usar `@observable` ao invés de setState

Se possível, não use `setState`. Toda vez que `setState` é chamado `render` tambpem é chamado, causando uma possível rerenderização desnecessária. 
Quando usamos `@observable` e `@observer` o Mobx só irá causar uma rerenderização caso uma propriedade do Mobx tenha mudado e tenha sido dereferenciada dentro do render (ou `@computed`, `autorun`, etc).

## Não alterar propriedade a partir de componentes

É interessante que não se altere propriedades a partir dos componentes para não se perder controle sobre as mudanças. 

## `@observer`, `@observer` e `@observer`

Usualmente dividimos componentes entre "containers" (smart) e "components" (dumb) e por conta disso podemos querer evitar que componentes "dumbs" sejam "observados". Porém é preferivel que eles sejam!

Isso é importante pois caso toda a `component tree` tenha `@observer` somente o componente onde a propriedade foi dereferenciada será rerenderizada, facilitando desta forma a reconciliação da DOM pelo React e tornando tudo mais rapidinho.

## Toda lógica de negócio deve estar na Store

Tudo que envolva lógica de negócio deve estar na store, o React deve ser usado tão somente para renderizar coisas. Isso facilita testes, pois desta forma você pode testar toda lógica montando tão somente a store. 
Outro ponto é a reutilização de código, já que tudo estará na store.

## `.fromPromise` e `.case` da biblioteca [mobx-utils](https://github.com/mobxjs/mobx-utils)

Uma abordagem interessante é utilizar `.fromPromise` e `.case` para gerenciar requisições à api:

```
@observable users
@action.bound requestUsers() {
    this.users = fromPromise(
        window.fetch("https://randomuser.me/api/?results=3")
        .then(r => r.json())
        .then(data => data.results)
    )
}
```

```
class observablePromise<T> {
    @observable state: string
    @observable value: T
    case<R>({fulfilled, rejected, pending}): R
}
```

```
<div>
    { this.users.case({
        "pending":   () => "loading",
        "rejected":  (e) => "error: " + e,
        "fulfilled": (users) => users.map(user =>
            <div key={user.login.username}>
                <img src={user.picture.thumbnail} />
                {user.email}
            </div>
        )
    }) }
</div>

```

## Utilizar [mobx-react-router](https://github.com/alisd23/mobx-react-router) ou alguma forma de sincronizar URL com o estado

A view deve receber todas informações do seu "gerenciador de estado", isso facilita testes e também renderização no servidor. Por conta disso é importante que o estado da URL esteja presente nas Stores.


##

Referências:

* [https://medium.com/dailyjs/mobx-react-best-practices-17e01cec4140](https://medium.com/dailyjs/mobx-react-best-practices-17e01cec4140)
* [https://mobx.js.org/best/store.html](https://mobx.js.org/best/store.html)
* [http://mobx-patterns.surge.sh/](http://mobx-patterns.surge.sh/)
* [https://mobx.js.org/best/pitfalls.html](https://mobx.js.org/best/pitfalls.html)