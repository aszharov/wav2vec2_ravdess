# Классификация эмоций в аудио

В данном репозитории представлены результаты применения модели wav2vec2.0 к задаче классификации эмоций.
Модель была предложена для применения в области self-supervised learning для работы с речевыми аудиофайлами.
wav2vec2.0 способна кодировать "сырые" файлы в признаковые представления (эмбеддинги), которые в дальнейшем можно использовать
для распознавания речи или в задачах похожих на рассмотренную здесь.

В своей работе я использовал feature_extractor BASE-конфигурации wav2vec2 для получения 12 признаковых наборов 
с размерностью 768 * 263 из промежуточного слоя трансформера в энкодере модели. В работе 
"Emotion Recognition from Speech Using Wav2vec 2.0 Embeddings" by Pepino эмбеддинги от модели взвешенно
усредняются и затем приводятся к виду одномерных векторов, по которым впоследствии и происходит классификация эмоций.
В работе "Emotion Recognition in Sound" by Popova для классификации эмоций используются mel-спектрограммы, которые подаются
на вход нейронной сети с архитектурой VGG. В своей работе я объединил две эти идеи (фичи из первой, архитектура из второй):
аудио-эмбеддинги формата 12 * 768 * 263 без усреднения подаются на вход нейронной сети с VGG-подобной архитектурой, на выходе
модели логиты классов эмоций. В качестве лосса я выбрал стандартную для задачи многоклассовой классификации кросс-энтропию.
На hugging-face я нашёл несколько fine-tuned моделей на датасете RAVDESS, однако они не продемонстрировали заявленных значений
по accuracy (либо я их как-то неправильно использовал). В качестве основной метрики я решил взять accuracy, потому что target 
довольно сбалансирован (за исключением класса neutral), кроме неё вычислялись precision, recall и f1-score для каждого класса.
Кроме того, я исследовал возможность классификации с использованием стандартных алгоритмов (случайный лес, градиентный бустинг)
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
я решил использовать разбиение на обучающую и валидационную выборку по актёрам в таком же виде, как указано в статье.
И очень расстроился (обрадовался), когда увидел, что тренировочный и валидационный лосс расходятся, что довольно явно соответствовало
переобучению под голоса конкретных актёров. В дальнейшем

Результаты обучения модели:

              precision    recall  f1-score   support

           0       0.60      0.75      0.67        20
           1       0.87      0.65      0.74        40
           2       0.80      0.50      0.62        40
           3       0.56      0.35      0.43        40
           4       0.75      0.90      0.82        40
           5       0.68      0.70      0.69        40
           6       0.56      0.82      0.67        40
           7       0.62      0.72      0.67        40

    accuracy                           0.67       300
   macro avg       0.68      0.68      0.66       300
weighted avg       0.68      0.67      0.66       300


На что не хватило времени, ресурсов, опыта:
- Работать с файлами произвольной длины, а не приводить их к единой, насколько я понял, модель Wav2Vec2.0 способна на это. В таком случае придётся видоизменять downstream-архитектуру.
- Сделать k-фолд кроссвалидацию по каждому из 5 фолдов, предложенных в статье by Luna-Jiménez, для более точной оценки полученных метрик качества
- Реализовать аугментацию тренировочных данных с целью увеличения выборки и повышения её многообразия. Мне кажется, что PitchShifting мог бы помочь с переобучением на трейне
- Попробовать другие архитектуры, я реализовал модель похожую на модель из работы by Pepino, где помимо эмбеддингов на отдельный вход подаются наборы eGeMAPS. Однако на удивление она проявила себя хуже
- Разморозить последние слои Wav2Vec2.0 для их подгонки под задачу классификации эмоций. Таким образом, можно было бы получить более информативные эмбеддинги для кокретной задачи.
