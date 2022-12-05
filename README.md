# Материалы (WIP)

Графическая подсистема является, вероятно, самой сложной и запутанной частью движка. И вместе с тем это именно та часть, в которой нужно очень хорошо ориентироваться. Вы можете легко обрабатывать ввод, воспроизводить звуки и даже не задумываться о том, как оно там внутри устроено. Но редкая игра обойдется без собственных красивых эффектов и тут без определенного набора знаний не обойтись. В одной статье невозможно охватить весь объем информации по данной теме, но я надеюсь, что смогу предоставить вам базу, опираясь на которую вы гораздо легче освоите все нюансы и тонкости рендера Urho3D.

![](images/abaddon.gif)

В папке [demo](demo) находится финальный результат. Двигая мышкой можно вращать модель, колесико поворачивает источника света. Все добавленные ресурсы отделены от файлов движка и находятся в папке [demo/MyData](demo/MyData).

## Используемая версия движка

Батники для скачивания и компиляции движка находятся в папке [engine](engine).

## Импортирование модели

Для экспериментов я решил взять модель Абаддона из игры Dota 2. Здесь есть как текстурные карты, которые Urho3D поддерживает "из коробки", так и специфичные для движка Source 2. Их мы тоже не оставим без внимания. Какая карта за что отвечает можно посмотреть в [этом документе](http://media.steampowered.com/apps/dota2/workshop/Dota2ShaderMaskGuide.pdf). Но прежде чем разбираться с материалами, нам нужно преобразовать модель в понятый для Urho3D формат. Сделать это можно разными способами.

В комплекте с движком поставляется утилита `AssetImporter`, которая позволяет импортировать модели из множества форматов. Ее можно найти в папке `bin/tool`. Команда `AssetImporter.exe model abaddon_econ.fbx abaddon.mdl -t` объединяет все фрагменты модели в единое целое (в fbx-файле модель разбита на части). Обязательно включайте опцию `-t`, если собираетесь использовать карты нормалей (этот параметр добавляет к вершинам модели информацию о касательных). Команда `AssetImporter.exe node abaddon_econ.fbx abaddon.xml -t` сохранит модель в виде префаба, в котором части модели организованы в виде иерархии нод. Так как в данном fbx-файле для модели не назначены текстуры, то материалы придется создавать вручную.

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

Первое, что бросается в глаза при открытии импортированной модели в редакторе Urho3D — белые ноги у коня. Дело в том, что в Доте карта свечения интерпретируется как интенсивность (черно-белое изображение), а в движке Urho3D как цвет. Вы можете исправить саму карту свечения (например в ГИМПе умножить ее на диффузную карту), но мы не ищем легких путей и будем менять шейдер Urho3D, чтобы приспособить его к набору текстур движка Source 2. Но это будет позже, а пока, чтобы посмотреть модель в более приятном виде, вы можете в материале изменить значение множителя свечения MatEmissiveColor на `0 0 0`, либо вообще удалить эту строку (только не забудьте потом все вернуть назад, этот параметр нам еще понадобится). Кстати, когда вы модифицируете и сохраняете файл материала, редактор автоматически подхватывает новую версию и обновляет свой вьюпорт. Более того, это работает даже для шейдеров. Вы можете писать шейдер и наблюдать за результатом своих трудов в реальном времени.

