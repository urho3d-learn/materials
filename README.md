# Материалы

Графическая подсистема является, вероятно, самой сложной и запутанной частью движка. И вместе с тем это именно та часть, в которой нужно очень хорошо ориентироваться. Вы можете легко обрабатывать ввод, воспроизводить звуки и даже не задумываться о том, как оно там внутри устроено. Но редкая игра обойдется без собственных красивых эффектов и тут без определенного набора знаний не обойтись. В одной статье невозможно охватить весь объем информации по данной теме, но я надеюсь, что смогу предоставить вам базу, опираясь на которую вы гораздо легче освоите все нюансы и тонкости рендера Urho3D.

![](images/abaddon.gif)

В папке [demo](demo) находится финальный результат. Двигая мышкой можно вращать модель, колёсико поворачивает источника света. Все добавленные ресурсы отделены от файлов движка и находятся в папке [demo/MyData](demo/MyData).

## Используемая версия движка

Батники для скачивания и компиляции движка находятся в папке [engine](engine).

## Импортирование модели

Для экспериментов я решил взять [модель Абаддона](http://www.dota2.com/workshop/requirements/abaddon) из игры Dota 2. Здесь есть как текстурные карты, которые Urho3D поддерживает "из коробки", так и специфичные для движка Source 2. Их мы тоже не оставим без внимания. Какая карта за что отвечает можно посмотреть в [этом документе](http://media.steampowered.com/apps/dota2/workshop/Dota2ShaderMaskGuide.pdf). Но прежде чем разбираться с материалами, нам нужно преобразовать модель в понятый для Urho3D формат. Сделать это можно разными способами.

В комплекте с движком поставляется утилита `AssetImporter`, которая позволяет импортировать модели из множества форматов. После компиляции двжика её можно найти в папке `bin/tool`. Команда `AssetImporter.exe model abaddon_econ.fbx abaddon.mdl -t` объединяет все фрагменты модели в единое целое (в fbx-файле модель разбита на части). Обязательно включайте опцию `-t`, если собираетесь использовать карты нормалей (этот параметр добавляет к вершинам модели информацию о касательных). Команда `AssetImporter.exe node abaddon_econ.fbx abaddon.xml -t` сохранит модель в виде префаба, в котором части модели организованы в виде иерархии нод. Так как в данном fbx-файле для модели не назначены текстуры, то материалы придется создавать вручную.

Можно импортировать модель прямо из редактора (`File` > `Import model...`). В этом случае используется все та же утилита `AssetImporter`. Параметр `-t` включен по умолчанию. Остальные параметры можно указать в окне `View` > `Editor settings`.

Для любителей Блендера есть отдельный экспортер. И он гораздо более функционален, чем `AssetImporter`. Им и воспользуемся. Процесс установки аддона и экспорта модели с помощью Блендера смотрите на видео: <https://youtu.be/96nH6fCbGHk>. В видеозаписи используется [эта](https://github.com/urho3d-tools/blender-exporter/tree/54467a488395d86282ce18e77626a34f61ceb391) версия аддона.

Материал коня после импорта (находится [тут](demo/MyData/Materials/1_abaddon_exported.xml)):

```
<material>
    <technique name="Techniques/DiffNormalSpecEmissive.xml" />
    <texture unit="diffuse" name="Textures/abaddon_mount_color.tga" />
    <parameter name="MatSpecColor" value="1.0 1.0 1.0 32.79999923706055" />
    <texture unit="specular" name="Textures/abaddon_mount_specmask.tga" />
    <parameter name="MatEmissiveColor" value="0.9194928407669067 0.9194928407669067 0.9194928407669067" />
    <texture unit="emissive" name="Textures/abaddon_mount_selfillummask.tga" />
    <texture unit="normal" name="Textures/abaddon_mount_normal.tga" />
</material>
```

Подправленный вручную материал (находится [тут](demo/MyData/Materials/2_abaddon_tuned.xml)):

```
<material>
    <technique name="Techniques/DiffNormalSpecEmissive.xml" />

    <texture unit="diffuse" name="Textures/abaddon_mount_color.tga" />

    <texture unit="specular" name="Textures/abaddon_mount_specmask.tga" />
    <parameter name="MatSpecColor" value="5 5 5 50" />

    <texture unit="emissive" name="Textures/abaddon_mount_selfillummask.tga" />
    <parameter name="MatEmissiveColor" value="1 1 1" />

    <texture unit="normal" name="Textures/abaddon_mount_normal.tga" />
</material>
```

Первое, что бросается в глаза при открытии импортированной модели в редакторе Urho3D — белые ноги у коня. Дело в том, что в Доте карта свечения интерпретируется как интенсивность (черно-белое изображение), а в движке Urho3D как цвет. Вы можете исправить саму карту свечения (например в ГИМПе умножить ее на диффузную карту), но мы не ищем легких путей и будем менять шейдер Urho3D, чтобы приспособить его к набору текстур движка Source 2. Но это будет позже, а пока, чтобы посмотреть модель в более приятном виде, вы можете в материале изменить значение множителя свечения `MatEmissiveColor` на `0 0 0`, либо вообще удалить эту строку (только не забудьте потом все вернуть назад, этот параметр нам еще понадобится). Кстати, когда вы модифицируете и сохраняете файл материала, редактор автоматически подхватывает новую версию и обновляет свой вьюпорт. Более того, это работает даже для шейдеров. Вы можете писать шейдер и наблюдать за результатом своих трудов в реальном времени.

## Процесс рендеринга

Процесс рендеринга в Urho3D описываться с помощью набора текстовых файлов: рендерпасов (CoreData/RenderPaths/\*.xml), техник (CoreData/Techniques/\*.xml) и шейдеров (CoreData/Shaders/GLSL/\*.glsl или CoreData/Shaders/HLSL/\*.hlsl). Как только вы поймете взаимосвязь между ними, в остальном разобраться не составит труда.

Итак, каждый объект сцены в большинстве случаев рендерится несколько раз с применением разных шейдеров (так называемые проходы рендера). Какой именно набор проходов (шейдеров) требуется для отрисовки каждого объекта, описывается в технике. В каком порядке эти проходы (шейдеры) будут выполнены, описано в рендерпасе. Еще раз обращаю ваше внимание: порядок строк в технике не важен.

Также есть материалы (Data/Materials/\*.xml). Материал определяет какая именно техника будет использоваться для объекта, а также наполняет эту технику конкретным содержанием, то есть определяет те данные, которые будут переданы в шейдеры: текстуры, цвета и т. п.

То есть совокупность рендерпас+техника+шейдеры можно представить как алгоритм, а материал — как набор данных для этого алгоритма.

Для понимания порядка вызова шейдеров взгляните на данную сравнительную табличку:

```
// НЕВЕРНО                                       // ВЕРНО
for (i = 0; i < числоОбъектов; i++)              for (i = 0; i < числоПроходов; i++)
{                                                {
    for (j = 0; j < числоПроходов; j++)              for (j = 0; j < числоОбъектов; j++)
        Рендрить(объект[i], проход[j]);              {
}                                                        if (объект[j] имеет проход[i])
                                                             Рендерить(объект[j], проход[i]);
                                                     }
                                                 }
```

Если в технике есть какой-то проход, но в рендерпасе такой проход отсутствует, то он не будет выполняться. И наоборот, если в рендерпасе есть какой-то проход, а в технике он отсутствует, то он тоже выполняться не будет. Однако существуют прописанные в самом движке проходы, которые вы не найдете в рендерпасе, но они все равно будут выполняться, если они есть в технике.

## Шейдеры

Urho3D поддерживает как OpenGL, так и DirectX, поэтому он содержит два аналогичных по функционалу набора шейдеров, которые расположены в папках CoreData/Shaders/GLSL и CoreData/Shaders/HLSL. Рассматривать будем шейдер OpenGL, хотя [hlsl-версия](demo/MyData/Shaders/HLSL/MyLitSolid.hlsl) также имеется.

В папке CoreData/Shaders/GLSL множество файлов, но вам стоит обратить особое внимание на один из них — LitSolid.glsl. Именно этот шейдер используется для подавляющего большинства материалов. Urho3D использует метод убершейдеров, то есть из огромного универсального шейдера LitSolid.glsl посредством дефайнов выбрасываются ненужные куски и получается маленький быстрый специализированный шейдер. Наборы дефайнов указываются в техниках, а некоторые дефайны добавляются самим движком.

## Возвращаемся к нашему коню

На данный момент Urho3D поддерживает 3 метода рендерина: Forward (традиционный), Light Pre-Pass и Deferred. Различия между ними являются темой для отдельного разговора, поэтому просто ограничимся методом Forward, который используется по умолчанию и описан в рендерпасе CoreData/RenderPaths/Forward.xml.

После экспорта из Блендера и ручной доработки мы получили материал [2_abaddon_tuned.xml](demo/MyData/Materials/2_abaddon_tuned.xml):

```
<material>
    <technique name="Techniques/DiffNormalSpecEmissive.xml" />

    <texture unit="diffuse" name="Textures/abaddon_mount_color.tga" />

    <texture unit="specular" name="Textures/abaddon_mount_specmask.tga" />
    <parameter name="MatSpecColor" value="5 5 5 50" />

    <texture unit="emissive" name="Textures/abaddon_mount_selfillummask.tga" />
    <parameter name="MatEmissiveColor" value="1 1 1" />

    <texture unit="normal" name="Textures/abaddon_mount_normal.tga" />
</material>
```

Этот материал использует технику Techniques/DiffNormalSpecEmissive.xml:

```
<technique vs="LitSolid" ps="LitSolid" psdefines="DIFFMAP">
    <pass name="base" psdefines="EMISSIVEMAP" />
    <pass name="light" vsdefines="NORMALMAP" psdefines="NORMALMAP SPECMAP" depthtest="equal" depthwrite="false" blend="add" />
    <pass name="prepass" vsdefines="NORMALMAP" psdefines="PREPASS NORMALMAP SPECMAP" />
    <pass name="material" psdefines="MATERIAL SPECMAP EMISSIVEMAP" depthtest="equal" depthwrite="false" />
    <pass name="deferred" vsdefines="NORMALMAP" psdefines="DEFERRED NORMALMAP SPECMAP EMISSIVEMAP" />
    <pass name="depth" vs="Depth" ps="Depth" />
    <pass name="shadow" vs="Shadow" ps="Shadow" />
</technique>
```

Так как мы будем модифицировать технику, то скопируйте ее в отдельный файл Techniques/MyTechnique.xml, чтобы не затронуть другие материалы, которые могут тоже использовать эту технику. В материале также измените название используемой техники.

Техника использует стандартный для материалов шейдер LitSolid.glsl. Так как мы будем изменять этот шейдер, то скопируйте его в Shaders/GLSL/MyLitSolid.glsl. Также измените имя используемого шейдера в технике. Заодно эту технику можно упростить, выкинув лишнее. Так как проходы `prepass`, `material`, `deferred` и `depth` отсутствуют в рендерпасе Forward (они определены в других рендерпасах), то они никогда не будут использоваться. Однако оставьте проход `shadow`. Хотя он и отсутствует в рендерпасе, но он является встроенным проходом и прописан в самом движке. Без него конь не будет отбрасывать тени.

В результате имеем Techniques/MyTechnique.xml

```
<technique vs="MyLitSolid" ps="MyLitSolid" psdefines="DIFFMAP">
    <pass name="base" psdefines="EMISSIVEMAP" />
    <pass name="light" vsdefines="NORMALMAP" psdefines="NORMALMAP SPECMAP" depthtest="equal" depthwrite="false" blend="add" />
    <pass name="shadow" vs="Shadow" ps="Shadow" />
</technique>
```

Давайте теперь поработаем с нашим шейдером Shaders/GLSL/MyLitSolid.glsl и изменим способ использования карты свечения на нужный нам. Дефайн EMISSIVEMAP (как раз и сигнализирующий, что материал должен определять карту свечения) в шейдере присутствует в нескольких местах, а так как разобрать шейдер построчно в рамках статьи не получится, то мы пойдем другим путем. Дефайны DEFERRED, PREPASS и MATERIAL никогда не будут определены в шейдере, так как проходы, в которых они определяются, мы удалили из техники как неиспользуемые. Поэтому смело удаляем обрамленный ими код. Размер шейдера уменьшился на треть. Дефайн EMISSIVEMAP остался только в одном месте.

Давайте умножим карту свечения на диффузный цвет. Замените строку

```
finalColor += cMatEmissiveColor * texture2D(sEmissiveMap, vTexCoord.xy).rgb;
```

на строку

```
finalColor += cMatEmissiveColor * texture2D(sEmissiveMap, vTexCoord.xy).rgb * diffColor.rgb;
```

Теперь ноги лошади светятся правильным голубым цветом.

Для большей красоты давайте добавим блум-эффект (ореол вокруг ярких частей изображения). Он реализуется двумя строчками в [скрипте](demo/MyData/Scripts/Main.as):

```
// Включаем HDR-рендеринг
renderer.hdrRendering = true;

// Добавляем эффект постобработки
viewport.renderPath.Append(cache.GetResource("XMLFile", "PostProcess/BloomHDR.xml"));
```

Обратите внимание, что эффект постобработки добавляется в конец рендерпаса. Вы можете дописать его прямо в файл.

## Rim Light

Подсветка краев модели — фейковая техника, симулирующая свет, падающий на модель сзади. В движке Source 2 используется отдельная маска для обозначения частей модели, на которые нужно накладывать данный эффект.

Для того, чтобы передать текстуру в шейдер, выберем неиспользуемый текстурный юнит. В нашем материале не используется карта окружения, поэтому будем использовать ее слот. Также в шейдере нам понадобятся два параметра RimColor и RimPower, поэтому сразу добавим и их. Итоговый материал выглядит [так](demo/MyData/Materials/4_abaddon_final.xml). Это может иногда сбивать с толку, но имя текстурного юнита не обязательно должно соответствовать его использованию. Вы вольны передавать через текстурные юниты что угодно и применять это в своих шейдерах как угодно.

Имя юниформа в шейдере будет отличаться от имени параметра в материале на префикс «c» (const): параметр RimColor станет юниформом cRimColor, а параметр RimPower станет юниформом cRimPower.

```
uniform float cRimPower;
uniform vec3 cRimColor;
```

А так выглядит сама реализация эффекта:

```
// RIM LIGHT

// Направление = позиция камеры - позиция фрагмента
vec3 viewDir = normalize(cCameraPosPS - vWorldPos.xyz);

// Скалярное произведение параллельных единичных векторов = 1.
// Скалярное произведение перпендикулярных векторов = 0.
// То есть, если представить шар, то его середина будет белой
// (так как нормаль параллельна линии взгляда), а к краям темнеть.
// Нам же наоборот нужны светлые края, поэтому скалярное произведение вычитается из единицы
float rimFactor = 1.0 - clamp(dot(normal, viewDir), 0.0, 1.0);

// Если cRimPower > 1, то подсветка сжимается к краям.
// Если cRimPower < 1, то подсветка наоборот становится более равномерной.
// При cRimPower = 0 вся модель будет равномерно окрашена, так как любое число в степени ноль = 1
rimFactor = pow(rimFactor, cRimPower);

// Проверяем, нужно ли использовать карту подсветки
#ifdef RIMMAP
    // Учитываем карту и цвет подсветки
    finalColor += texture2D(sEnvMap, vTexCoord.xy).rgb * cRimColor * rimFactor;
#else
    finalColor += cRimColor * rimFactor;
#endif
```

Обратите внимание на дефайн RIMMAP. Возможно вам захочется не использовать карту подсветки, а просто наложить эффект на всю модель. В этом случае вы просто не определяете дефайн RIMMAP в технике. Кстати, не забудьте определить дефайн RIMMAP в технике :) Итоговая техника выглядит [так](demo/MyData/Techniques/MyTechnique.xml), а шейдер — [так](demo/MyData/Shaders/GLSL/MyLitSolid.glsl).

## Литература

* https://urho3d-doxygen.github.io/1_9_0_tutors/_rendering.html
* https://urho3d-doxygen.github.io/1_9_0_tutors/_a_p_i_differences.html
* https://urho3d-doxygen.github.io/1_9_0_tutors/_shaders.html
* https://urho3d-doxygen.github.io/1_9_0_tutors/_render_paths.html
* https://urho3d-doxygen.github.io/1_9_0_tutors/_materials.html
* https://urho3d-doxygen.github.io/1_9_0_tutors/_rendering_modes.html

---

*Следующий урок: <https://github.com/urho3d-learn/post-effects>.*

*Старая версия демки: <https://github.com/1vanK/Urho3DHabrahabr05>.*
