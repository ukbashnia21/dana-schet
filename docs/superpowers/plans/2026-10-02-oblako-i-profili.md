# Облако и два профиля — план работ

> **Для исполнителя:** задачи идут по порядку, каждая заканчивается рабочим
> тренажёром и коммитом. Шаги помечены чекбоксами для отслеживания.

**Цель:** прогресс и история перестают быть привязанными к одному браузеру и
делятся на два профиля — Дана и папа — с общей базой команды.

**Подход:** подключаем Firebase Firestore вместо артефактной базы Claude. Код
синхронизации уже написан под это API (`doc().set()`, `collection().orderBy()`,
`onSnapshot()`), поэтому меняется получение объекта `db`, а логика остаётся.
Структуру данных сразу закладываем под двух игроков, чтобы не мигрировать
дважды.

**Технологии:** Firebase Firestore (compat SDK v10), ванильный JS без сборки,
GitHub Pages.

## Общие ограничения

- **Никакой зависимости от Claude.** У Даны его нет. `window.claude` может
  отсутствовать — код обязан работать без него.
- **Один файл приложения.** `index.html` содержит всё; сборки нет и не будет.
- **Офлайн обязателен.** Дана занимается без интернета, записи досылаются позже.
- **`localStorage` остаётся вторым носителем.** Если облако недоступно, тренажёр
  работает как раньше.
- **Версия в двух местах.** При заметных правках поднимать `версия N` в
  `index.html` и `C='ustny-vN'` в `sw.js` — иначе у Даны останется старый кэш.
- **Про тесты.** В проекте нет тестовой инфраструктуры, и заводить её ради
  семейного тренажёра не нужно. Вместо модульных тестов каждая задача
  заканчивается проверкой в браузере с точным ожидаемым результатом. Проверка —
  обязательная часть задачи, а не пожелание.

---

### Задача 1: Проект Firebase и правила доступа

Код не меняется. Это настройка внешнего сервиса, без которой остальное не
взлетит.

**Файлы:**
- Создать: `docs/firebase-setup.md` — куда записать, что и где заведено

- [ ] **Шаг 1: Завести проект**

На `console.firebase.google.com` создать проект `dana-schet`. Аналитику не
включать — она не нужна и тянет лишний код.

- [ ] **Шаг 2: Создать базу Firestore**

Build → Firestore Database → Create database → режим **production**, регион
`eur3` (Европа, ближе по задержке).

- [ ] **Шаг 3: Записать правила доступа**

Firestore → Rules, заменить содержимое на:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /teams/{teamId}/{document=**} {
      allow read, write: if teamId.size() >= 20;
    }
  }
}
```

Смысл: писать можно только в команду, чей идентификатор длиной от 20 символов.
Идентификатор случайный и известен только вам — он и есть ключ.

**Честно о риске:** тот, кто узнает идентификатор команды, сможет читать и
портить данные. Для тренажёра по арифметике это приемлемо; паролей и личных
сведений там нет. Если позже захочется строже — добавим анонимный вход и
правило `request.auth != null`.

- [ ] **Шаг 4: Получить конфигурацию приложения**

Project settings → General → Your apps → Web app (значок `</>`), имя
`dana-schet`. Firebase выдаст объект `firebaseConfig` — скопировать целиком.

Ключ в этом объекте **не секрет**: он определяет, в какой проект писать, а
доступ ограничивают правила. Его нормально держать в публичном репозитории.

- [ ] **Шаг 5: Записать, что завели**

Создать `docs/firebase-setup.md`:

```markdown
# Firebase для тренажёра

Проект: dana-schet, регион eur3.
Правила: писать можно в teams/<id> при длине id от 20 символов.
Идентификатор команды хранится в index.html, константа TEAM.
Консоль: https://console.firebase.google.com/project/dana-schet/firestore
```

- [ ] **Шаг 6: Коммит**

```bash
git add docs/firebase-setup.md
git commit -m "Firebase: проект, база и правила доступа заведены"
```

---

### Задача 2: Подключить Firestore вместо артефактной базы

**Файлы:**
- Изменить: `index.html` — секция `<head>` и функция `initDb()`

**Интерфейсы:**
- Отдаёт дальше: глобальная `DB` — объект Firestore с методами `doc()`,
  `collection()`; глобальная `TEAM` — строка-идентификатор команды.

- [ ] **Шаг 1: Подключить SDK**

В `index.html` перед закрывающим `</head>` добавить:

```html
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-firestore-compat.js"></script>
```

Версия **compat** выбрана намеренно: её API совпадает с тем, под который уже
написан код синхронизации.

- [ ] **Шаг 2: Задать команду и конфигурацию**

В начале `<script>`, сразу после `const LS='ustny-schet-v1'`, добавить:

```js
const TEAM='СЮДА-ВСТАВИТЬ-СЛУЧАЙНУЮ-СТРОКУ-ОТ-20-СИМВОЛОВ';
const FB={apiKey:'…',authDomain:'…',projectId:'dana-schet',storageBucket:'…',messagingSenderId:'…',appId:'…'};
```

Строку команды сгенерировать командой и вставить:

```bash
openssl rand -hex 16
```

Значения `FB` взять из того, что выдал Firebase в задаче 1, шаг 4.

- [ ] **Шаг 3: Переписать получение базы**

Заменить начало `initDb()`:

```js
async function initDb(){
 if(!(window.claude&&window.claude.use)){setSync('local');return;}
 let db=null;try{db=await window.claude.use('db');}catch(e){}
 if(!db){setSync('local');return;}
 DB=db;
```

на:

```js
async function initDb(){
 let db=null;
 try{
  if(window.firebase&&FB.apiKey&&!FB.apiKey.startsWith('…')){
   firebase.initializeApp(FB);
   const fs=firebase.firestore();
   await fs.enablePersistence({synchronizeTabs:true}).catch(()=>{});
   db={doc:p=>fs.doc('teams/'+TEAM+'/'+p),collection:p=>fs.collection('teams/'+TEAM+'/'+p)};
  }
 }catch(e){}
 if(!db){setSync('local');return;}
 DB=db;
```

Обёртка подставляет путь команды, поэтому остальной код продолжает писать
`DB.doc('app/state')`, не зная про команду.

`enablePersistence` включает офлайн-кэш: Дана занимается без интернета, записи
досылаются при подключении.

- [ ] **Шаг 4: Поднять версию**

В `index.html` заменить `версия 3` на `версия 4`; в `sw.js` заменить
`C='ustny-v3'` на `C='ustny-v4'`.

- [ ] **Шаг 5: Проверка в браузере**

```bash
cd /Users/nikolaj/Downloads/dana-schet && python3 -m http.server 8745 --bind 127.0.0.1
```

Открыть `http://127.0.0.1:8745`, пройти одну минуту, затем:

- внизу страницы должно быть **«Синхронизировано»**, а не «Только на этом
  компьютере»;
- в консоли Firebase, Firestore → Data, появился `teams/<ваш id>/app/state`;
- открыть ту же страницу в приватном окне — виден тот же прогресс.

Если внизу «Нет связи с облаком» — проверить правила: идентификатор команды
должен быть не короче 20 символов.

- [ ] **Шаг 6: Коммит и публикация**

```bash
git add index.html sw.js
git commit -m "синхронизация через Firestore вместо артефактной базы"
git push
```

Через минуту проверить `https://ukbashnia21.github.io/dana-schet/` — внизу
должно быть «Синхронизировано».

---

### Задача 3: Структура данных под двух игроков

**Файлы:**
- Изменить: `index.html` — `fresh()`, `pushState()`, `initDb()`, `save()`

**Интерфейсы:**
- Потребляет: `DB`, `TEAM` из задачи 2.
- Отдаёт дальше: глобальная `WHO` — `'dana'` или `'papa'`; все записи игрока
  идут в `players/<WHO>`, общие дни команды — в `team/days`.

- [ ] **Шаг 1: Ввести текущего игрока**

После `const TEAM=…` добавить:

```js
const LS_WHO='ustny-schet-who';
let WHO=null;
try{WHO=localStorage.getItem(LS_WHO);}catch(e){}
const PLAYERS={dana:{name:'Дана',adult:false},papa:{name:'Папа',adult:true}};
```

- [ ] **Шаг 2: Писать профиль в ветку игрока**

В `pushState()` заменить путь:

```js
await DB.doc('app/state').set(JSON.parse(JSON.stringify(S)));
```

на:

```js
await DB.doc('players/'+WHO).set(JSON.parse(JSON.stringify(S)));
```

- [ ] **Шаг 3: Читать оттуда же**

Сначала защитить от запуска без выбранного игрока — в самое начало `initDb()`,
перед `let db=null`, добавить:

```js
 if(!WHO){setSync('local');return;}
```

Без этой строки `initDb()` на старте, когда игрок ещё не выбран, запишет
профиль в `players/null` и засорит базу.

Затем заменить все три обращения к `app/state` на `players/'+WHO`:

```js
const snap=await db.doc('players/'+WHO).get();
…
else if(S.updatedAt)await db.doc('players/'+WHO).set(JSON.parse(JSON.stringify(S)));
…
db.doc('players/'+WHO).onSnapshot(sn=>{…});
```

- [ ] **Шаг 4: Разделить сессии по игрокам**

В `pushSession()` добавить поле игрока:

```js
async function pushSession(s){if(!DB)return;try{await DB.collection('sessions').doc(s.id).set(Object.assign({who:WHO},s));}catch(e){}}
```

В `initDb()` при чтении сессий оставить только свои:

```js
const qs=await db.collection('sessions').orderBy('ts','desc').limit(1000).get();
const have=new Set(SESS.map(s=>s.id)),remote=new Set();
qs.docs.forEach(d=>{const v=d.data();remote.add(d.id);if(v.who&&v.who!==WHO)return;if(!have.has(d.id))SESS.push(JSON.parse(JSON.stringify(v)));});
```

- [ ] **Шаг 5: Публиковать свои дни для команды**

В конец `save()`, после `pushState()`, добавить вызов:

```js
pushDays();
```

И рядом с `pushState()` определить:

```js
async function pushDays(){if(!DB||!WHO)return;
 try{await DB.doc('team/days').set({[WHO]:Object.fromEntries(Object.entries(S.days).map(([k,d])=>[k,taskDone(k)]))},{merge:true});}catch(e){}}
```

Записывается не весь день, а только «выполнена ли норма» по каждой дате —
этого достаточно команде и не раскрывает лишнего.

- [ ] **Шаг 6: Проверка в браузере**

Открыть `http://127.0.0.1:8745`, в консоли браузера выполнить:

```js
localStorage.setItem('ustny-schet-who','dana'); location.reload();
```

Пройти минуту. В Firestore должны появиться:

```
teams/<id>/players/dana      — профиль
teams/<id>/sessions/<id>     — с полем who: "dana"
teams/<id>/team/days         — { dana: { "2026-10-02": false } }
```

Затем переключиться на `papa` той же командой в консоли, перезагрузить,
пройти минуту — профиль `players/papa` появляется отдельно, а `players/dana`
не меняется.

- [ ] **Шаг 7: Коммит**

```bash
git add index.html
git commit -m "данные разделены по игрокам: профили, сессии и дни команды"
git push
```

---

### Задача 4: Экран выбора профиля и защита папиного

**Файлы:**
- Изменить: `index.html` — добавить `vWho()`, изменить `render()`, добавить
  стили `.who`

**Интерфейсы:**
- Потребляет: `WHO`, `PLAYERS` из задачи 3.
- Отдаёт дальше: `setWho(id)` — выбирает игрока и перезагружает интерфейс.

- [ ] **Шаг 1: Экран выбора**

Перед функцией `render()` добавить:

```js
function vWho(){return `<section class="page narrow who">
 <h2>Кто занимается?</h2>
 <p class="muted">Выбор запомнится на этом устройстве. Сменить можно в «Родителям».</p>
 <div class="row">
  <button class="btn primary huge" onclick="setWho('dana')">Дана</button>
  <button class="btn huge" onclick="setWho('papa')">Папа</button>
 </div></section>`;}
function setWho(id){
 if(id==='papa'&&S.pin){const v=prompt('PIN папы');if(v!==S.pin)return;}
 WHO=id;try{localStorage.setItem(LS_WHO,id);}catch(e){}
 S=fresh();S.name=PLAYERS[id].name;
 try{const x=JSON.parse(localStorage.getItem(LS+'-'+id)||'null');if(x&&x.v)S=Object.assign(fresh(),x);}catch(e){}
 render();initDb();}
```

- [ ] **Шаг 2: Показывать его, пока игрок не выбран**

В начало `render()`, сразу после `if(RUN)return;`, добавить:

```js
 if(!WHO){$('#nav').innerHTML='';$('#stats').innerHTML='';$('#main').innerHTML=vWho();return;}
```

- [ ] **Шаг 3: Хранить профили раздельно и локально**

Заменить в `save()`:

```js
localStorage.setItem(LS,JSON.stringify(S));
```

на:

```js
localStorage.setItem(LS+(WHO?'-'+WHO:''),JSON.stringify(S));
```

То же самое в `initDb()` — везде, где встречается `localStorage.setItem(LS,`.

- [ ] **Шаг 4: Кнопка смены игрока**

В `vParent()`, в строку с кнопками раздела «Данные», добавить перед
«Выйти»:

```js
<button class="btn" onclick="if(confirm('Сменить игрока?')){localStorage.removeItem(LS_WHO);WHO=null;render();}">Сменить игрока</button>
```

- [ ] **Шаг 5: Проверка в браузере**

Очистить выбор и перезагрузить:

```js
localStorage.removeItem('ustny-schet-who'); location.reload();
```

Ожидаемое:

- появляется экран «Кто занимается?»;
- нажатие «Дана» открывает тренажёр, имя в шапке — Дана;
- «Родителям» → задать PIN → «Сменить игрока» → «Папа» запрашивает PIN;
- неверный PIN не пускает, верный открывает чистый профиль папы;
- прогресс Даны при этом не тронут — вернуться на неё и убедиться.

- [ ] **Шаг 6: Поднять версию и опубликовать**

`версия 4` → `версия 5` в `index.html`, `C='ustny-v4'` → `C='ustny-v5'` в
`sw.js`.

```bash
git add index.html sw.js
git commit -m "выбор игрока на устройстве, профиль папы под PIN"
git push
```

---

### Задача 5: Личная норма и перенос накопленного

**Файлы:**
- Изменить: `index.html` — `fresh()`, `vParent()`, `saveSet()`, `taskDone()`

- [ ] **Шаг 1: Норма становится личной**

В `fresh()` настройка `perDay` уже лежит внутри `settings` каждого игрока —
после задачи 4 профили раздельные, значит норма уже личная. Проверить это и
убедиться, что в `vParent()` подпись отражает, чья именно норма:

```js
<label class="row">Челленджей в день для ${esc(PLAYERS[WHO].name)} <input id="perDay" class="field sm" type="number" min="1" max="20" value="${st.perDay}"></label>
```

- [ ] **Шаг 2: Перенести прогресс, накопленный до разделения**

Старые данные лежат в `localStorage` под ключом без игрока. Разово перенести их
Дане — добавить в `setWho()` перед чтением профиля:

```js
 try{const old=localStorage.getItem(LS);
  if(old&&id==='dana'&&!localStorage.getItem(LS+'-dana')){localStorage.setItem(LS+'-dana',old);localStorage.removeItem(LS);}
 }catch(e){}
```

- [ ] **Шаг 3: Проверка в браузере**

- Открыть тренажёр, пройти минуту **до** выбора игрока (старый ключ).
- Очистить выбор, перезагрузить, выбрать «Дана».
- Ожидаемое: её монеты, серия и история на месте, ничего не потерялось.
- Выбрать «Папа»: профиль чистый, норма настраивается отдельно.

- [ ] **Шаг 4: Коммит и публикация**

```bash
git add index.html
git commit -m "личная норма у каждого игрока, перенос прежнего прогресса Дане"
git push
```

---

## Что остаётся за рамками этого плана

Слои 4–6 из спецификации — команда, совместный режим и взрослая лестница —
получат отдельные планы, когда этот будет закончен и проверен в деле.

Причина порядка: командные правила опираются на `team/days`, который
появляется здесь, в задаче 3. Строить команду раньше, чем данные разделены по
игрокам, означало бы переделывать её дважды.
