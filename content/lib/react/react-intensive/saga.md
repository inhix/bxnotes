---
title: saga
seoDescription: saga | React.
seoKeywords: saga, React
date: 2020-01-19 05:00:00
---
# saga

## push pull

В случае с ```takeEvery```, обработчик просто вызывается каждый раз, при этом нельзя контролировать какие-то исключительные случаи, когда запускать задачу не нужно, или когда следует прекратить наблюдение. 

В случае с ```take```, контроль инвертирован. Вместо того, чтобы запушить action в обработчик, сага самостоятельно обрабатывает action.

Данный инвертированный подход позволяет контролиловать процесс выполнения и решать нетривиальные задачи.

## блокирующие вызовы

Эффект ```call``` блокирующий. Генератор останавливается, пока ```call``` не завершится.

Если нужно обработать конкурентные процессы, то блокирующий вызов не подойдет.

## не блокирующие вызовы

Для не блокирующих вызовов библиотека предлагает эффект ```fork```. Когда задача запускается через ```fork```, она выполняется в фоне, а вызывающая его сага не блокируется и не ждет окончание процесса.

Проблема в том, что так как ```fork``` задача запущена в фоне, она не может вернут в вызывающий код какое-то значение, как это делает ```call```. По этому, форкнутый процесс должен диспатчить события. 

Особенностью ```fork``` процесса является то, что эффект ```cancel``` не отменяет его моментально. Перед удалением процесса будет выполнен блок ```finally```, который может содержать логику, которую необходимо выполнить перед удалением процесса. 

```js
function* sagaWorker(user, password) {
  try {
    //...
  } catch(error) {
    //...
  } finally {
    if (yield cancelled()) {
      // ... обработка cancel логики
    }
  }
}
```

## параллельность

```Yield``` хорош для ожидания асинхронных эффектов в линейном стиле, но для параллельного исполнения не подойдет. 

```js
// код будет выполнен последовательно
const users = yield call(fetch, '/users')
const repos = yield call(fetch, '/repos')
```

Для параллельного выполнения следует использовать эффект ```all```.

```js
import { all, call } from 'redux-saga/effects'

// correct, effects will get executed in parallel
const [users, repos] = yield all([
  call(fetch, '/users'),
  call(fetch, '/repos')
])
```

Когда запускается ```yield all```, генератор блокируется до тех пор, пока все эффекты в переданном массиве не завершатся, либо пока в каком-либо из этих эффектов не возникнет ошибка (как в ```Promise.all```).

## race

Для того, чтобы запустить гонку из нескольких процессов и вернуть результат первого выполненного процесса, есть эффект ```race```.

## fork модель

В ```redux-saga``` фоновые задачи могут выполняться в 2 режимах: 

+ ```fork``` &ndash; создает прикрепленный *fork*
+ ```spawn``` &ndash; создает открепленный *fork* 

### attached fork

Прикрепленные <span lang="en">(Attached)</span> следуют 3 правилам:

+ завершение &ndash; сага будет существовать, пока не завершится она и все прикрепленные дочерние процессы
+ выброс ошибки &ndash; если ошибка происходит в саге или в прикрепленном процессе, прерывается сага и все прикрепленные к ней процессы. **Невозможно обработать ошибку внутри fork процесса**. Блок ```try { } catch(error) { }``` может оборачивать только блокирующий вызов.
+ отмена <span lang="en">сancellation</span> &ndash; текущая блокирующая задача и все прикрепленные процессы будут отменены.

### detached fork

Открепленные (<span lang="en">Detached</span>) форки живут в собственных контекстах исполнения. Родители не ожидают их завершения. Не пойманные ошибки из ```spawn``` задач не всплывают в родительский контекст. Отмена родителя не выполняет автоматическую отмену открепленного форка (их нужно отменять явно).

# каналы

```Redux-saga``` предоставляет способ запускать саги снаружи redux окружения и привязывать их к кастомному *Input/Output*.

*Buffer* отвечает за то, как работать с переполнением каналов, когда события обрабатываются медленнее, чем поступают.

[Каналы в документации](https://redux-saga.js.org/docs/advanced/Channels.html)

## синхронизация по сокетам

Канал может быть из чего угодно, не только из сокетов. Например, каналом может быть движение мыши, ресайз окна, нажатия на кнопку.

Вариант создания канала из *веб-сокетов* *firebase*:

```js
import apiService from '../../services/api'

const createAuthChannel = () =>
  eventChannel((emit) => apiService.onAuthChange((user) => emit({ user })))

export const watchAuthChangeSaga = function*() {
  const authChannel = yield call(createAuthChannel)
  while (true) {
    const { user } = yield take(authChannel)
    if (user) {
      yield put({
        type: SIGN_IN_SUCCESS,
        payload: { user }
      })
    } else {
      yield put({
        type: SIGN_OUT
      })
    }
  }
}

export function* saga() {
  yield all([signUpSaga(), watchAuthChangeSaga()])
}
```

```apiService``` это обертка над *fireBase* API:

```js
import firebase from 'firebase/app'
import 'firebase/auth'
import 'firebase/firestore'
import { firebaseConfig } from '../config'

class ApiService {
  constructor(fbConfig) {
    firebase.initializeApp(fbConfig)
    this.fb = firebase
  }

  //...

  onAuthChange = (callback) => this.fb.auth().onAuthStateChanged(callback)
}
```

## балансировка нагрузки

Допустим, требуется ограничить количество одновременно выполняемых процессов 3 штуками. Если есть свободный процесс, то задача должна немедленно выполниться свободным процессом, иначе задача помещается в очередь и будет обработана первым освободившимся процессом.

С помощью каналов это решается так:

```js
import { channel } from 'redux-saga'
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  // create a channel to queue incoming requests
  const chan = yield call(channel)

  // create 3 worker 'threads'
  for (var i = 0; i < 3; i++) {
    yield fork(handleRequest, chan)
  }

  while (true) {
    const {payload} = yield take('REQUEST')
    yield put(chan, payload)
  }
}

function* handleRequest(chan) {
  while (true) {
    const payload = yield take(chan)
    // process the request
  }
}
```

В примере выше мы создали канал с помощью фабрики каналов. Мы получили канал, который по умолчанию буферизирует все входящие сообщения (конечно, если нет свободного процесса, который может немедленно его обработать).

Сага ```watchRequests``` создает 3 рабочих ```fork``` саги. Созданный канал предоставляется каждой из 3 созданных саг. ```watchRequests``` использует данный канал, чтобы диспатчить в рабочие саги сообщения. На каждый ```REQUEST``` экшен сага наблюдатель отправляет сообщение в канал. Данное сообщение будет принято любой свободной рабочей сагой. Иначе сообщения будут ожидать в очереди канала.

Каждая рабочая сага &ndash; ожидает сообщение в бесконечном цикле. Обработка сообщения блокирует рабочую сагу. После того, как работа сделана, цикл начнется сначала и сага вернется в режим ожидания задачи. 

Данный механизм обеспечивает автоматическую балансировку нагрузки между 3 рабочими сагами &ndash; освободившаяся сага немедленно получит новую задачу.