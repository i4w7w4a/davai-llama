# Предпроверка FFmpeg и FFprobe

Проверено 29 сентября 2026 года на синтетическом видео без частных материалов. Это проверка установленной пары утилит для следующего малого шага, а не готовность конвейера обработки видео.

## Воспроизведение

Команды PowerShell выполняются из корня репозитория. Все созданные файлы остаются в исключённой из Git папке `.local/media-preflight/`.

```powershell
ffmpeg -version
ffprobe -version
New-Item -ItemType Directory -Force .local/media-preflight | Out-Null
ffmpeg -hide_banner -loglevel error -f lavfi -i "testsrc2=size=160x90:rate=2:duration=3" -c:v libx264 -pix_fmt yuv420p -g 6 -bf 0 -y .local/media-preflight/sample.mp4
ffprobe -v error -show_entries format=duration:stream=index,codec_name,time_base,start_time,duration,nb_frames -of json .local/media-preflight/sample.mp4
ffprobe -v error -select_streams v:0 -show_packets -show_entries packet=pts_time,duration_time -of compact=p=0:nk=1 .local/media-preflight/sample.mp4
ffmpeg -hide_banner -loglevel info -i .local/media-preflight/sample.mp4 -vf "select='eq(n,0)+eq(n,2)+eq(n,4)',showinfo" -fps_mode vfr -frames:v 3 -y .local/media-preflight/frame_%02d.png
```

Для запроса момента 1,25 с сравниваются два способа и отдельно извлечённый кадр с PTS 1,5 с:

```powershell
ffmpeg -hide_banner -loglevel info -i .local/media-preflight/sample.mp4 -vf "select='gte(t,1.25)',showinfo" -fps_mode vfr -frames:v 1 -update 1 -y .local/media-preflight/requested_1_25.png
ffmpeg -hide_banner -loglevel error -ss 1.25 -i .local/media-preflight/sample.mp4 -frames:v 1 -update 1 -y .local/media-preflight/seek_1_25.png
ffmpeg -hide_banner -loglevel error -i .local/media-preflight/sample.mp4 -vf "select='eq(n,3)'" -fps_mode vfr -frames:v 1 -update 1 -y .local/media-preflight/reference_1_5.png
Get-FileHash -Algorithm SHA256 -LiteralPath .local/media-preflight/requested_1_25.png,.local/media-preflight/seek_1_25.png,.local/media-preflight/reference_1_5.png
```

## Наблюдения

- Установлены FFmpeg `2025-08-20-git-4d7c609be3-essentials_build` и FFprobe `8.1.2-full_build`. Это разные сборки; все приведённые команды завершились с кодом 0.
- Получен H.264 MP4 размером 160×90 без звука. FFprobe показал `start_time=0`, `time_base=1/16384`, шесть кадров и длительность потока и контейнера `3.000000` с.
- PTS пакетов: `0`, `0.5`, `1`, `1.5`, `2`, `2.5` с; длительность каждого пакета `0.5` с. Созданы три PNG. `showinfo` для них показал исходные PTS `0`, `1`, `2` с соответственно.
- Фильтр `select='gte(t,1.25)'` выбрал первый кадр с PTS `1.5` с. В этом примере входной `-ss 1.25` сохранил такое же изображение: SHA-256 обоих PNG совпал с отдельно извлечённым кадром `n=3` (PTS `1.5` с). Это наблюдение для данного файла и команд, не общее правило округления времени.

## Граница и три риска следующего среза

1. **Исходное время.** VFR, ненулевое начало потока и перестановка кадров здесь не проверены. Нельзя восстанавливать время только из FPS или номера кадра; следующий опыт должен сверять PTS производных с исходником.
2. **Полнота выборки.** Кадры через секунду не доказывают отсутствие короткого или немого события между ними. Для следующего среза нужна явная граница покрытия и способ углубить выборку.
3. **Частные данные и объём производных.** На реальном видео извлечение может заполнить диск и оставить личные кадры. Следующий срез должен ограничить объём, вести результаты локально и проверить очистку без изменения исходника.

Звук, другие кодеки и реальные материалы не проверялись. Совместимость двух сборок за пределами этого опыта неизвестна.
