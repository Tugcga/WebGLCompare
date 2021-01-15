## Сравнение разных способов сбилдить WebGL в Unity

### Описание

Цель этого небольшого сравнения - выяснить, какой способ создания WebGL-приложений средствами Unity является самым лучшим. Сейчас есть три принципиально разных способа сделать приложение на Unity. Первый способ - это использовать старый (и всё ещё основной) подход, основанный на MonoBehaviour. Второй способ - это использовать активно развивающуюся технологию DOTS вместе с гибридным рендером. Третий способ - это использовать пакет Tiny, который находится ещё на очень ранней стадии разработки. Однако ему пророчатся большие перспективы.

Для теста было сделано три проекта, в каждом из которых была собрана одна простая сцена: камера, источник света с тенями, плоскость и шарики. По клику мышки шариков появляется всё больше и больше, и они постоянно двигаются вверх-вниз, как будто отскакивают от плоскости. Никакой физики, анимаций или чего-то подобного. Вот скрин сцены (а точнее окна Game).

![Test scene](pictures/pic_01.png?raw=true)

Тест состоит в том, чтобы посмотреть, при каком числе двигающихся шариков начнёт проседать FPS. Для подхода MonoBehaviour использовалась версия Unity 2020.2. Для DOTS использовалась версия Unity 2021.1a9 (так как в ней пофикшена критическая ошибка при использовании иерархий Entity в WebGL билде) и пакет Hybrid 0.10. Для Tiny использовалась версия Unity 2020.2 и пакет Tiny 0.31.

Ссылки на тестовые билды:

* [MonoBehaviour](https://tugcga.github.io/test_apps/webgl/webgl_mb/index.html)
* [DOTS](https://tugcga.github.io/test_apps/webgl/webgl_dots/index.html)
* [Tiny](https://tugcga.github.io/test_apps/webgl/webgl_tiny/MainAssembly.html)

Сначала опишем, как именно собирались сцены, а потом обсудим результаты замеров.

### MonoBehaviour

Код скрипта, который порождает и перемещает шарики на сцене.

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public struct ObjectData
{
    public float velocity;
}

public class Spawner : MonoBehaviour
{
    public float spawnWidth;
    public float spawnHeight;
    public float g;
    public GameObject prefab;

    public Text countText;

    List<Transform> objectTfms = new List<Transform>();
    List<ObjectData> objectDatas = new List<ObjectData>();

    void Update()
    {
        if(Input.GetMouseButtonDown(0))
        {
            int newCount = objectTfms.Count / 2 + 1;
            for(int i = 0; i < newCount; i++)
            {
                GameObject newObject = Instantiate(prefab, new Vector3(Random.Range(-spawnWidth, spawnWidth), Random.Range(spawnHeight / 4.0f, spawnHeight), Random.Range(-spawnWidth, spawnWidth)), new Quaternion());
                float scale = Random.Range(0.25f, 1.0f);
                newObject.transform.localScale = new Vector3(scale, scale, scale);
                objectTfms.Add(newObject.transform);
                objectDatas.Add(new ObjectData() { velocity = 0.0f});
            }
            countText.text = objectTfms.Count.ToString();
        }

        for(int i = 0; i < objectTfms.Count; i++)
        {
            ObjectData d = objectDatas[i];
            d.velocity += Time.deltaTime * g;
			
            Vector3 oldPosition = objectTfms[i].position;
            oldPosition.y += d.velocity * Time.deltaTime;

            if(oldPosition.y <= 0.0f)
            {
                d.velocity *= -1;
                oldPosition.y *= -1;
            }

            objectDatas[i] = d;
            objectTfms[i].position = oldPosition;
        }
    }
}
```

Конечно, абсурдно двигать каждый шарик в своём собственном Update-е, поэтому все они двигаются одновременно за один вызов Update. Для этого отдельно в список сохраняются ссылки на компоненты Transform порождённых объектов и данные с текущими скоростями шариков.

Скрипт для вычисления числа кадров в секунду.

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class FPSCounter : MonoBehaviour
{
    public Text fpsText;
    public float lastUpdateTime;

    float accumulator;
    int steps;

    void Start()
    {
        lastUpdateTime = Time.time;
        accumulator = 0.0f;
        steps = 0;
    }

    void Update()
    {
        if(Time.time - lastUpdateTime > 1.0f)
        {
            fpsText.text = ((float)steps / accumulator).ToString();
            accumulator = 0.0f;
            steps = 0;

            lastUpdateTime = Time.time;
        }
        accumulator += Time.deltaTime;
        steps++;
    }
}
```

Тут всё просто. В течении секунды считается, какова общая длительность всех прошедших кадров, потом это делится на число прошедших кадров (получается средняя длительность кадра), и потом к полученному значению берётся обратная величина.

### DOTS

Идея состоит в том, чтобы использовать MonoBehaviour как можно меньше, а всё, что возможно, делать через сущности, компоненты и системы (то есть использовать подход Entity Component System).

Компонент с данными для каждого шарика.

```
using System.Collections;
using Unity.Entities;
using Unity.Mathematics;

[GenerateAuthoringComponent]
public struct ObjectComponent : IComponentData
{
    public float velocity;
    public float g;
}
```

В нём храним и скорость, и значение ускорения свободного падения. Это сделано для того, чтобы каждую следующую координату можно было пересчитывать в не зависимости от данных в каких-нибудь глобальных сущностях-синглтонах.

Компонент для сущности, порождающей шарики.

```
using System.Collections;
using Unity.Entities;
using Unity.Mathematics;

[GenerateAuthoringComponent]
public struct SpawnerComponent : IComponentData
{
    public float spawnWidth;
    public float spawnHeight;
    public float g;
    public Entity prefab;

    public int spawnedCount;
}
```

Система для перемещения шариков.

```
using Unity.Entities;
using Unity.Collections;
using Unity.Transforms;
using Unity.Mathematics;

public class ObjectSystem : SystemBase
{
    protected override void OnCreate()
    {
        base.OnCreate();
    }

    protected override void OnUpdate()
    {
        float dt = Time.DeltaTime;
        Entities.
            ForEach((ref Translation translate, ref ObjectComponent obj) =>
            {
                obj.velocity += dt * obj.g;
                translate.Value.y += obj.velocity * dt;

                if (translate.Value.y <= 0.0f)
                {
                    translate.Value.y *= -1;
                    obj.velocity *= -1;
                }
            }).
            ScheduleParallel();
    }
}
```

Тут всё понятно. Перебираем все сущности, в которых есть и Translation, и ObjectComponent. Для каждой такой сущности вычисляем новые значения скорости (кладём её в ObjectComponent) и координаты по вертикальной оси (кладём её в Translation).

Система для порождения шариков.

```
using Unity.Entities;
using Unity.Collections;
using Unity.Transforms;
using Unity.Mathematics;
using UnityEngine;

public class SpawnerSystem : SystemBase
{
    Unity.Mathematics.Random random;
    EntityManager manager;

    protected override void OnCreate()
    {
        base.OnCreate();
        manager = EntityManager;
        random = new Unity.Mathematics.Random(1);
        RequireSingletonForUpdate<SpawnerComponent>();
    }

    protected override void OnUpdate()
    {
        if(Input.GetMouseButtonDown(0))
        {
            SpawnerComponent spawner = GetSingleton<SpawnerComponent>();

            int newCount = spawner.spawnedCount / 2 + 1;
            NativeArray<Entity> newObjects = manager.Instantiate(spawner.prefab, newCount, Allocator.Temp);
            for(int i = 0; i < newCount; i++)
            {
                float x = random.NextFloat(-spawner.spawnWidth, spawner.spawnWidth);
                float z = random.NextFloat(-spawner.spawnWidth, spawner.spawnWidth);
                float y = random.NextFloat(spawner.spawnHeight / 4.0f, spawner.spawnHeight);

                Entity e = newObjects[i];
                manager.SetComponentData<Translation>(e, new Translation() { Value = new float3(x, y, z)});
                manager.SetComponentData<ObjectComponent>(e, new ObjectComponent() { velocity = 0.0f, g = spawner.g});

                float s = random.NextFloat(0.25f, 1.0f);
                manager.AddComponentData(e, new Scale() {Value = s });
            }
            newObjects.Dispose();

            spawner.spawnedCount += newCount;
            SetSingleton<SpawnerComponent>(spawner);

            GlobalData.SetCount(spawner.spawnedCount);
        }
    }
}
```

Тут в общем тоже всё ясно. Последняя строчка ```GlobalData.SetCount(spawner.spawnedCount);``` нужна для того, чтобы передать в текстовый объект на канвасе значение о числе порождённых шариков. Так как системы - это одно, а интерфейс - он на основе MonoBehaviour, то это как бы два разных мира, и поэтому передачу данных можно сделать вот так вот через глобальный статический класс. Собственно скрипт для UI.

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public static class GlobalData
{
    public static UICount uiCount;
    static bool isInit = false;

    public static void InitUICount(UICount _uiCount)
    {
        uiCount = _uiCount;
        isInit = true;
    }

    public static void SetCount(int value)
    {
        if(isInit)
        {
            uiCount.SetCount(value);
        }
    }
}

public class UICount : MonoBehaviour
{
    public Text countText;

    public void Start()
    {
        SetCount(0);
        GlobalData.InitUICount(this);
    }

    public void SetCount(int value)
    {
        countText.text = value.ToString();
    }
}
```

### Tiny

В этом случае код ровно такой же, как в DOTS случае. Единственное отличи состоит в том, как организован UI. Обычный UI на основе MonoBehaviour там не работает, поэтому надо всё делать на принципах ECS. Но это к тестированию вообще не имеет никакого отношения.

### Результаты тестирования

Дальше в таблице перечислены разные показатели числа шариков и сколько FPS выдаёт каждый из сбилденных проектов. Рядом с показателем FPS в скобках указано два числа - это процент загрузки процессора и видеокарты. Показатели загрузки, конечно, относительные, однако все запуски производились на одной и той же машине. Это позволяет, в том числе, увидеть, во что упирается производительность - в процессор или видеокарту. Ещё в верхней строке таблицы указан размер полученного билда в мегабайтах. Конечно, в реальном проекте основной объём занимают ресурсы графики и звука. Однако указанные объёмы позволяют, например, оценить размер собственно движка для воспроизведения WebGL-приложения.

Число шариков | MonoBehaviour (4.64 Mb) | DOTS (9.32 Mb) | Tiny (2.33 Mb)
--- | --- | --- | ---
709 | 60 (5.4% / 9.4%) | 60 (13% / 18%) |  60 (7% / 5%)
1 064 | 60 (6.7% / 9.4%) | 48 (14% / 18%) |  60 (10% / 6.5%)
1 597 | 60 (8% / 31%) | 30 (13% / 15%) | 60 (12.5% / 8%) 
2 396 | 60 (11% / 33%) | 24 (16% / 15%) | 52 (15% / 25%) 
3 595 | 57 (15% / 33%) | 18 (17% / 30%) | 30 (13% / 30%) 
5 393 | 39 (15% / 34%) | 12 (16% / 14%) | 24 (15% / 33%) 
8 090 | 28 (15% / 35%) | 8 (16% / 26%) | 16 (14% / 30%) 
12 136 | 19 (16% / 30%) | 6 (16% / 13%) | 12 (15% / 35%) 

На материале шариков был активирован параметр GPU Instancing. Из-за этого для MonoBehaviour билда на сцене происходили жуткие фризы в момент порождения новых шариков. И так это было где-то до первой тысячи объектов. Потом при создании новых экземпляров объектов никаких фризов больше не было. Наверное буфферы какие-нибудь перераспределялись, и, наконец, перераспределились достаточно. Поэтому вот ещё таблица с показаниями для билда, в котором у материала для шариков отключён параметр GPU Instancing. В этом случае при порождении новых объектов никаких фризов не происходит, всё идёт гладко. Но и показатели FPS в целом ниже.

Число шариков | MonoBehaviour без инстансов
--- | ---
709 | 60 (10.5% / 8%)
1 064 | 60 (13% / 10%)
1 597 | 45 (14% / 8%)
2 396 | 34 (15% / 8.5%)
3 595 | 26 (16% / 14%)
5 393 | 18 (16% / 14%)
8 090 | 12 (16% / 15%)
12 136 | 8 (16% / 17%)

Несколько замечаний. Для DOTS проекта использовался Universal Render Pipline и Hybrid Renderer V2. Во всех случая загрузка процессора не превышала 15%. Это соответствует примерно одному ядру процессора. В общем получается, что в WebGL всё вычисляется в одном потоке. Загрузка видеокарты во всех случаях ограничена 30-35%. Тоже, наверное, какое-нибудь ограничение WebGL приложений.

### Итого

В целом подход Tiny представляется наиболее оптимальным. Единственное, с помощью Tiny нельзя сделать практически никакую игру. Слишком ограниченные пока у этого пакета возможности. Однако разработка идёт. Так что всё будет. Наверное. Подход, основанный на MonoBehaviour, вроде тоже ничего. А вот DOTS показывает совсем плохие результаты. Столько было разговоров, что за этим подходом будущее, и вот на тебе. По крайней мере в WebGL он работает не очень.

Конечно, делать столь однозначные выводы из одного теста не стоит. Всё-таки этот тест во многом специфический, ведь вряд ли есть игра, где геймплэй основан на взаимодействии с большим числом одинаковых неизменяемых объектов.

### Что насчёт Standalone билдов

Здесь ситуация кардинально другая. Вот таблица с показаниями.

Число шариков | MonoBehaviour (43.2 Mb) | DOTS (61.8 Mb) | Tiny (6.2 Mb)
--- | --- | --- | ---
12 136 | 60 (46.5% / 54.6%) | 60 (6% / 40%) | 60 (41% / 46%)
18 205 | 40 (51% / 53%) | 60 (8.5% / 50%) | 42 (43% / 48%)
27 308 | 29 (64% / 55%) | 60 (10% / 71%) | 30 (44% / 48%)
40 963 | 19 (64% / 55%) | 55 (10% / 71%) | 20 (43% / 46%)
61 445 | 12 (61% / 52%) | 38 (14% / 99%) | 15 (40% / 36%)
92 168 | 7 (60% / 50%) | 26 (14% / 100%) | 10 (36% / 33%)

В общем DOTS с гибридным рендером выигрывает с большим отрывом. Пусть даже у него и самый большой размер дистрибутива. Tiny похуже, MonoBehaviour ещё хуже. Видно, что DOTS тормозится из-за видеокарты, в то время как MonoBehaviour из-за процессора. Tiny хоть и выдаёт небольшой FPS, но при этом систему не сильно нагружает. Почему так происходит - не ясно.

### Сравнение с Three.JS

Three.JS конечно не является игровым движком, но тем не менее интересно посмотреть, какой производительности можно добиться, используя библиотеку, предназначенную исключительно для вывода трёхмерной графики в браузере.

Вот ссылка на приложение для тестирования: [Three.JS test](https://tugcga.github.io/test_apps/webgl/webgl_threejs/index.html)

Все шарики, само собой, сделаны через InstancedMesh. Вот результаты замеров.

Число шариков | Three.JS (1.15 Mb)
--- | ---
709 | 60 (4% / 27%)
1 064 | 60 (4.7% / 31%)
1 597 | 60 (5% / 36%)
2 396 | 60 (5% / 36%)
3 595 | 60 (5% / 36%)
5 393 | 60 (5.3% / 75%)
8 090 | 60 (5.5% / 78%)
12 136 | 60 (6% / 82%)
18 205 | 20 (16% / 98%)
27 308 | 20 (16% / 99%)
40 963 | 19 (16% / 100%)
61 445 | 18 (16% / 100%)
92 168 | 5 (18% / 100%)

Получается, что производительность упирается в видеокарту. Это всё из-за теней. Если их отключить или хотя бы сделать более локальными, то fps будет ещё больше. Нагрузка на процессор сопоставима с DOTS методом в Standalone приложении. Короче, очень хорошая. И вообще, пусть метод Three.JS и не такой же по производительности, как Standalone билды, но очень к ним близок. Гораздо ближе, чем WebGL из Unity.