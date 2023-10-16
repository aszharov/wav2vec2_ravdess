# Классификация эмоций в аудио

[Ссылка на Яндекс Диск с весами моделей](https://disk.yandex.ru/d/CbyXVc5dTpG9DA) \
[Ноутбук с результатами](https://github.com/aszharov/wav2vec2_ravdess/wav2vec2.0-ravdess-for-vk-lab-2023.ipynb)

В данном репозитории представлены результаты применения модели wav2vec2.0 к задаче классификации эмоций.
Модель была предложена для применения в области self-supervised learning для работы с речевыми аудиофайлами.
wav2vec2.0 способна кодировать "сырые" файлы в признаковые представления (эмбеддинги), которые в дальнейшем можно использовать
для распознавания речи или в задачах похожих на рассмотренную здесь.

В своей работе я использовал feature_extractor BASE-конфигурации wav2vec2 для получения 12 признаковых наборов 
с размерностью 768 * 263 из промежуточных слоёв трансформера в энкодере модели. В работе 
"Emotion Recognition from Speech Using Wav2vec 2.0 Embeddings" by Pepino эмбеддинги от модели взвешенно
усредняются и затем приводятся к виду одномерных векторов, по которым впоследствии происходит классификация эмоций.
В работе "Emotion Recognition in Sound" by Popova для классификации эмоций используются mel-спектрограммы, которые подаются
на вход нейронной сети с архитектурой VGG. В своей работе я объединил две эти идеи (фичи из первой, архитектура из второй):
аудиоэмбеддинги формата 12 * 768 * 263 без усреднения подаются на вход нейронной сети с VGG-подобной архитектурой, на выходе
модели логиты классов эмоций (модель **fe_vgg_wav2emo**). В качестве лосса выбрана стандартная для задачи многоклассовой классификации кросс-энтропия.
На hugging-face я нашёл несколько fine-tuned моделей на датасете RAVDESS, однако они не продемонстрировали заявленных значений
по accuracy (либо они рассчитаны на немного другие входные данные). В качестве основной метрики я решил взять accuracy, потому что target 
довольно сбалансирован (за исключением класса neutral) и это стандартный выбор в других работах, кроме неё вычислялись precision, recall и f1-score для каждого класса.
Помимо этого, я исследовал возможность классификации с использованием стандартных алгоритмов (случайный лес, градиентный бустинг)
с входными данными в виде коэффициентов MFCC или набора eGeMAPS. 

Несколько слов о самом наборе данных. Ryerson Audio-Visual Database of Emotional Speech and Song (RAVDESS) состоит из видео и аудио 
речи и песен. В работе я использовал audio-speech-only подвыборку, которая состоит из 1440 записей в формате .wav (по 60 для каждого из 
24 актёров). Название файла состоит из 7 значений, которые соответствуют идентификаторам. Для более простой работы я привёл все файлы к
единой длине в 5 секунд (что соответствует самой продолжительной записи).

Filename identifiers
- Modality (01 = full-AV, 02 = video-only, 03 = audio-only).
- Vocal channel (01 = speech, 02 = song).
- Emotion (01 = neutral, 02 = calm, 03 = happy, 04 = sad, 05 = angry, 06 = fearful, 07 = disgust, 08 = surprised).
- Emotional intensity (01 = normal, 02 = strong). NOTE: There is no strong intensity for the 'neutral' emotion.
- Statement (01 = "Kids are talking by the door", 02 = "Dogs are sitting by the door").
- Repetition (01 = 1st repetition, 02 = 2nd repetition).
- Actor (01 to 24. Odd numbered actors are male, even numbered actors are female).

Filename example: 03-01-06-01-02-01-12.wav
Audio-only (03)
Speech (01)
Fearful (06)
Normal intensity (01)
Statement "dogs" (02)
1st Repetition (01)
12th Actor (12)
Female, as the actor ID number is even.

Изначально я решил использовать стадартное `train_test_split` разбиение из sklearn. Однако знакомство с работой
"A proposal for Multimodal Emotion Recognition using aural transformers and Action Units on RAVDESS dataset" by Luna-Jiménez
помогло мне понять, что такой выбор неверен, потому что есть риск переобучения под голос конкретного актёра. В дальнейшем 
я решил использовать разбиение на обучающую и валидационную выборку по актёрам (в таком же виде, как указано в статье).
И очень <del>расстроился</del> обрадовался, когда увидел, что тренировочный и валидационный лосс расходятся, что довольно явно соответствовало
переобучению под голоса конкретных актёров. В дальнейшем я продолжил использовать голоса 19 актёров на трейне и 5 оставшихся актёров на валидации с сохранением
баланса между мужскими и женскими голосами. 

Значение функции потерь и метрики качества accuracy при обучении в течение 100 эпох на трейне и валидации:
![Обучение модели](https://github.com/aszharov/wav2vec2_ravdess/blob/main/tvp_fe_vgg_wav2emo.png?raw=true)

Сильно заметное переобучение (решение: более жёсткие методы регуляризации, аугментация тренировочного датасета, поиск более подходящей архитектуры).

Результаты классификации модели **fe_vgg_wav2emo** (Feature Extractor + VGG) на валидационном датасете:

               precision    recall  f1-score   support

    neutral (0)     0.58      0.55      0.56        20
    calm (1)        0.77      0.85      0.81        40
    happy (2)       0.72      0.45      0.55        40
    sad (3)         0.59      0.47      0.53        40
    angry (4)       0.73      0.88      0.80        40
    fearful (5)     0.81      0.62      0.70        40
    disgust (6)     0.60      0.72      0.66        40
    surprised (7)   0.58      0.78      0.67        40

    accuracy                            0.67       300
    macro avg       0.67      0.67      0.66       300
    weighted avg    0.68      0.67      0.67       300

![Матрица ошибок](https://github.com/aszharov/wav2vec2_ravdess/blob/main/cm_fe_vgg_wav2emo.png?raw=true)

Результат оставляет желать лучшего. Тем не менее эта модель (в первом приближении) преодолевает Human-Perception для Audio-only экземпляров датасета RAVDESS (0.62). 
Необходима дополнительная оценка accuracy на кросс-валидации, чтобы говорить о достижении с большей уверенностью.
Дополнительно была разработана модель, вдохновленная моделью Fusion из статьи by Pepino, которая помимо аудиоэмбеддингов использует аудиофичи eGeMAPS.

Значение функции потерь и метрики качества accuracy при обучении в течение 100 эпох на трейне и валидации для расширенной модели:
![Обучение модели](https://github.com/aszharov/wav2vec2_ravdess/blob/main/tvp_fe_vgg_eGeMAPS_wav2emo.png?raw=true)

Результаты классификации модели **fe_vgg_eGeMAPS_wav2emo** (Feature Extractor + VGG + eGeMAPS features) на валидационном датасете:
               precision    recall  f1-score   support

    neutral (0)     0.58      0.75      0.65        20
    calm (1)        0.79      0.78      0.78        40
    happy (2)       0.76      0.78      0.77        40
    sad (3)         0.70      0.35      0.47        40
    angry (4)       0.79      0.95      0.86        40
    fearful (5)     0.61      0.70      0.65        40
    disgust (6)     0.78      0.70      0.74        40
    surprised (7)   0.70      0.78      0.74        40

    accuracy                            0.72       300
    macro avg       0.71      0.72      0.71       300
    weighted avg    0.72      0.72      0.71       300

![Матрица ошибок](https://github.com/aszharov/wav2vec2_ravdess/blob/main/cm_fe_vgg_eGeMAPS_wav2emo.png?raw=true)

Резульат по accuracy лучше на 0.03, что соответствует дополнительным 9 верным предсказаниям при валидационном датасете в 300 объектов. 
Это не слишком много, чтобы утверждать, что первая модель хуже, чем вторая. Требуется оценка метрик качества на кросс-валидации.

Веса обеих моделей сохранены на Яндекс Диске (ссылка в самом начале).
Среди остальных экспериментов можно выделить, модель градиентного бустинга, использующую фичи eGeMAPS и достигающую `accuracy = 0.56`

На что не хватило времени, ресурсов, опыта:
- Работать с файлами произвольной длины, а не приводить их к единой, насколько я понял, модель Wav2Vec2.0 способна на это. В таком случае придётся видоизменять downstream архитектуру.
- Сделать k-фолд кросс-валидацию по каждому из 5 фолдов, предложенных в статье by Luna-Jiménez, для более точной оценки полученных метрик качества
- Реализовать аугментацию тренировочных данных с целью увеличения выборки и повышения её многообразия. Мне кажется, что PitchShifting мог бы помочь с переобучением на трейне
- Попробовать другие архитектуры, например, заменить слои Conv2D на LSTM.
- Разморозить последние слои Wav2Vec2.0 для их подгонки под задачу классификации эмоций. Таким образом, можно было бы получить более информативные эмбеддинги для конкретной задачи.

Источники:
- https://paperswithcode.com/sota/emotion-recognition-on-ravdess
- [Luna-Jiménez](https://paperswithcode.com/paper/a-proposal-for-multimodal-emotion-recognition)
- [Pepino](https://isca-speech.org/archive/pdfs/interspeech_2021/pepino21_interspeech.pdf)
- [RAVDESS](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0196391)
- [Popova](https://www.researchgate.net/publication/319343259_Emotion_Recognition_in_Sound)
- [Fine-tuned model from HF](https://huggingface.co/Wiam/wav2vec2-base-finetuned-ravdess)
