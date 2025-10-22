# Editing and Restoring EEG Markers
This script restores and edits EEG event markers from BrainVision recordings.
EN version first - for RU version please read below.

## ENGLISH

### Overview
* The script works with BrainVision marker files.
* I worked with it in GoogleColab, but you can do as you wish.
* **Signals are never modified**; the script reads/writes marker text files (`.vmrk`, `.Markers`).
* The file contains **two executable entry points**:
    * One that processes all `*.vmrk` in the working directory to create `_ConfMarkers.vmrk` using a behavioral CSV (`BaseReport_…_CORR….csv`).
    * One that processes all `*.Markers` into a `three_stages/` subfolder with suffix `_three_stages.Markers`.

---

## Part A — Photodetector-based detection (from `PhotoS`)
* Loads **one EEG channel**: `PhotoS` from a BrainVision recording:
   * `vhdr_path = "Confidence_test000030.vhdr"` (editable variable).
   * `raw = mne.io.read_raw_brainvision(vhdr_path, preload=False)`; then `raw.pick_channels(['PhotoS'])`.
* Detection parameters (**editable**):
   * `chunk_duration = 60` seconds. The data is processed in parts.
   * `threshold = 2.2`.
   * `min_interval_sec = 0.5` → converted to `min_interval_samples = int(min_interval_sec * sfreq)`.
* Detection rule (loop over chunks):
   * When `signal[i-1] < threshold` and `signal[i] >= threshold` and the refractory interval is satisfied ⇒ **emit event** at that time index.
* Output of this section:
   * Builds a table with columns `onset_sec` and `marker_code`, saves to **`photo_markers.csv`** and triggers **Colab download** via `files.download("photo_markers.csv")`.

---

## Part B — Add confidence markers from behavioral CSV → new `.vmrk`
* **Injection of confidence markers** from behavioral logs (CSV) into the marker timeline.  
* Function: `process_type2_markers(vmrk_in: str, base_report_csv: str, vmrk_out: str)`.
* Input marker parsing:
   * Reads the original `.vmrk` header and marker lines under `[Marker Infos]`.
   * Parses entries like `MkN=Stimulus,S  2,38131,1,All` into dictionaries:
      * `parts` (split by `,`), `latency` (int from `parts[2]`), `clean` (marker code with spaces removed, e.g., `"S2"`).
* Behavioral CSV loading:
   * **Autodetects delimiter** by peeking the first line: `sep = ',' if ',' in first else ';'`.
   * Reads with `pd.read_csv(..., sep=sep, encoding='utf-8', engine='python')`.
   * **Cleans BOM** from headers: `df.columns = df.columns.str.strip().str.lstrip('\ufeff')`.
   * **Confidence column** chosen as the **first column whose name contains** `'Увер'`.
   * **Values** are coerced to integers: `1` = confident, `0` = not confident.
* Insertion logic for S6/S7 per trial:
   * Finds indices of all `S4`/`S5` (`correct`/`incorrect`) in the existing marker stream.
   * For each `S4`/`S5`, searches the **next `S1`** after it:
      * If found ⇒ inserts confidence marker **midway** between `S4/5` and that next `S1`.
      * If **not** found ⇒ inserts confidence marker at **`+500 ms`** after the `S4/5` timestamp.
   * Confidence marker label is:
      * `S  6` if confidence value is `1`.
      * `S  7` if confidence value is `0`.
* Output:
   * Merges original markers + injected S6/S7, sorts by latency, and writes a **new `.vmrk`**: `vmrk_out`.

---

## Part C — Batch runner for `.vmrk` → `_ConfMarkers.vmrk` (inside of the second snippet)
* The part that makes the code work with multiple files.
* Helper: `find_base_report(recording_num: str) -> str`
   * Looks for a behavioral CSV in the **current working directory** using pattern:
      * `BaseReport_{recording_num}_*CORR*.csv`
   * Picks exactly one match; otherwise raises an error.
* Entry point: `def main():`
   * Iterates **all `*.vmrk`** in the working directory.
   * **Skips** files that already end with `"_ConfMarkers.vmrk"`.
   * Extracts the **first numeric substring** from the filename and **strips leading zeros** via `str(int(...))`.
   * Calls `find_base_report(recording_num)` to locate the CSV.
   * Calls `process_type2_markers(vmrk_path, base_csv, vmrk_out)` where:
      * `vmrk_out = vmrk_path.replace('.vmrk', '_ConfMarkers.vmrk')`.
* Execution:
   * There is a block `if __name__ == '__main__': main()` which **runs this batch** when the script is executed.

---

## Part D — Modify S1 based on correctness & confidence → `_NewMarkers.vmrk`
* Function: `process_type1_preserve(vmrk_in, vmrk_out)`
   * Reads header and markers from `.vmrk`.
   * Builds an internal list `markers` of dicts with fields:
      * `prefix` (e.g., `"Mk12"`), `parts` (CSV fields), `clean` (like `"S1"`), `latency` (int), and `idx`.
* Re-labels **only `S1` markers** to encode the combo of correctness + confidence that follows:
   * Mapping `combo_map`:
      * `('S4','S6') → '11'` (correct and confident)
      * `('S5','S6') → '12'` (incorrect and confident)
      * `('S4','S7') → '13'` (correct and non-confident)
      * `('S5','S7') → '14'` (incorrect and non-confident)
   * For each `S1`, finds the **next** accuracy (`S4`/`S5`) and the **next** confidence (`S6`/`S7`) **after** that `S1`.  
   * If both exist ⇒ changes `S  1` to `S {11|12|13|14}` using the mapping.
* Output:
   * Writes a new `.vmrk` with updated `S1` labels (header preserved).  
   * The function itself contains a guard to **skip** input files that already end with `'_NewMarkers.vmrk'` (when used in a batch context).

---

## Part E — Transform `.vmrk` into `.Markers` and build the final three-stage version → `three_stages/*_three_stages.Markers`
* Maered converter converts every `.vmrk` in `input_dir` into a BrainVision Analyzer–importable `.Markers` file named `<basename>_Raw Data.Markers`.
* The script reads the original `.vmrk`, finds the `[Marker Infos]` section, parses each `MkN=` line, skips the system entry `New Segment`, restores commas in descriptions by replacing `\1` with `,`, maps channel `0` to `All`, and writes a CSV-like `.Markers` file with a two-line header (`Sampling rate: {sampling_rate}Hz, SamplingInterval: {sampling_interval}ms` and column names).
* Sampling parameters are editable via `sampling_rate` and `sampling_interval`.

* Function: `process_three_stages(markers_in, markers_out)`
   * Reads a **`.Markers`** text file: first **two lines are header**, the rest are marker rows.
   * Parses each row into `records` with `parts` and `clean` (spaces removed in the event code).
   * Finds **stage headers** in the stream: any `clean` in `('S11','S12','S13','S14')`.
      * The **first digit** encodes the **stage number**.
      * The **second digit** encodes the **answer type**.
      * Within a trial, the **last digit must match** across related events.
   * For each stage segment `[S1x … next S1y)`:
      * Re-labels `S2 → S2x` and `S3 → S3x` (i.e., appends the **last digit** of the current stage label).
   * Writes back header + transformed markers to `markers_out`.
* Batch entry point (separate from Part C):
   * When the script runs, it also executes:
      * `out_dir = 'three_stages'`; `os.makedirs(out_dir, exist_ok=True)`.
      * Iterates **all `*.Markers`** in the **current working directory**.
      * **Skips** files ending with `'_three_stages.Markers'`.
      * Writes outputs into `three_stages/` with suffix **`_three_stages`** before the extension.

---

## File structure
* The **only directory created by the code** is `three_stages/` (for Part E outputs).
* All other files are expected in the **current working directory**.
* Output naming seen in code:
   * `photo_markers.csv` (from Part A).
   * `*_ConfMarkers.vmrk` (from Part B/C).
   * `*_three_stages.Markers` placed under `three_stages/` (from Part E).

---

## How to run
* From a folder that contains your `.vhdr/.eeg/.vmrk/.Markers` and behavioral CSVs:
   * `python editing_and_restoring_eeg_markers.py`
* What will happen on run:
   * **Batch 1 (Part C)**: processes every `*.vmrk` into `*_ConfMarkers.vmrk` by locating `BaseReport_{num}_*CORR*.csv` (where `{num}` is extracted from the `.vmrk` name by taking the **first** digit sequence and stripping leading zeros).
   * **Batch 2 (Part E)**: processes every `*.Markers` into `three_stages/*_three_stages.Markers`.

---

## Parameters you can edit
* Photodetector (Part A):
   * `vhdr_path`
   * `chunk_duration`
   * `threshold`
   * `min_interval_sec`
* Behavioral CSV parsing (Part B):
   * Autodetected `sep` (`,` or `;`) based on the first line of the CSV.
   * `conf_col` is the **first** header containing `'Увер'`; `conf` values are treated as integers `1/0`.
* Stage transformation (Part E):
   * No user parameters; behavior fixed by the logic above.

---

## Notes & safeguards
* **Skips**: 
   * `.vmrk` files already ending with `"_ConfMarkers.vmrk"` (Part C) are skipped.
   * Any logic that uses `'_NewMarkers.vmrk'` treats such files as already processed in that step.
   * `.Markers` files already ending with `"_three_stages.Markers"` are skipped in Part E.
* **Delimiter & encoding**:
   * CSV delimiter autodetection `,` vs `;`.
   * Header BOM is stripped (`lstrip('\ufeff')`).
* **Error handling excerpts**:
   * If no or multiple `BaseReport_{num}_*CORR*.csv` matches, the `.vmrk` is skipped with a printed message.
   * If no numeric substring is found in the `.vmrk` name, the file is skipped.

---

## RUSSIAN

### Общее описание
* Скрипт работает с **файлами маркеров** BrainVision.
* Я работала с ним в GoogleColab, но это на ваше усмотрение.
* **Сигналы не меняются**; изменяются только текстовые файлы маркеров (`.vmrk`, `.Markers`).
* В файле есть **две точки запуска**:
   * Обработка всех `*.vmrk` в рабочей папке → создание `_ConfMarkers.vmrk` на основе CSV `BaseReport_…_CORR….csv`.
   * Обработка всех `*.Markers` → папка `three_stages/` с суффиксом `_three_stages.Markers`.

---

## Часть A — Детекция по фотодетектору (`PhotoS`)
* В коде: “Восстановление маркеров на основе данных фотодетектора.”
* Загружается **один канал**: `PhotoS` из BrainVision:
   * `vhdr_path = "Confidence_test000030.vhdr"` (можно менять).
   * `read_raw_brainvision(...); raw.pick_channels(['PhotoS'])`.
* Параметры детекции (редактируемые):
   * `chunk_duration = 60` (сек).
   * `threshold = 2.2`.
   * `min_interval_sec = 0.5` (→ `min_interval_samples`).
* Правило детекции:
   * Пороговый переход снизу вверх + выдержка минимального интервала ⇒ событие.
* Выход:
   * Таблица `onset_sec`/`marker_code` → **`photo_markers.csv`**, загружается через `files.download(...)` в Colab.

---

## Часть B — Добавление маркеров уверенности из CSV → новый `.vmrk`
* Восстановление маркеров уверенности на основании файла с поведенческими данными
* Функция: `process_type2_markers(vmrk_in, base_report_csv, vmrk_out)`.
* Парсинг `.vmrk`:
   * Заголовок + строки под `[Marker Infos]`, затем разбор `MkN=Stimulus,S  2,...`.
   * `clean` — код без пробелов (например, `"S2"`).
* Загрузка CSV:
   * **Автодетект** разделителя: `,` или `;` по первой строке.
   * Чтение в `utf-8`, удаление `BOM` из заголовков.
   * Колонка уверенности — первая, где в имени есть `'Увер'`.
   * Значения приводятся к `int`: `1` — уверенно, `0` — неуверенно.
* Вставка `S6/S7` по каждой пробе:
   * Для каждого `S4`/`S5` ищется следующий `S1`:
      * если найден ⇒ вставка **в середину интервала** `S4/5 → S1`.
      * если **не найден** ⇒ вставка на **`+500 мс`** от `S4/5`.
   * Метка: `S  6` при `1`, `S  7` при `0`.
* Выход:
   * Запись объединённых и отсортированных маркеров в новый `.vmrk` (`vmrk_out`).

---

## Часть C — Пакетная обработка `.vmrk` → `_ConfMarkers.vmrk` (внутри части B)
* Поиск CSV: `find_base_report(recording_num)`
   * Шаблон: `BaseReport_{recording_num}_*CORR*.csv` в **текущей папке**.
   * Требуется ровно одно совпадение.
* Точка запуска `main()`:
   * Обходит все `*.vmrk`.
   * **Пропускает** имена, оканчивающиеся на `"_ConfMarkers.vmrk"`.
   * Извлекает **первую** числовую подпоследовательность из имени и убирает ведущие нули.
   * Находит CSV через `find_base_report(...)`.
   * Вызывает `process_type2_markers(...)` и пишет `*_ConfMarkers.vmrk`.
* Запуск:
   * Блок `if __name__ == '__main__': main()` **автоматически** запускает эту партию при выполнении файла.

---

## Часть D — Изменение `S1` по комбинации точности/уверенности → `_NewMarkers.vmrk`
* Изменение маркера S 1 на основании маркеров правильности и уверенности. 
* Функция: `process_type1_preserve(vmrk_in, vmrk_out)`.
* Логика:
   * Для каждого `S1` ищутся следующие `S4/S5` (точность) и `S6/S7` (уверенность).
   * `('S4','S6')→'11'` (правильный уверенный)
   * `('S5','S6')→'12'` (неправильный уверенный)
   * `('S4','S7')→'13'` (правильный неуверенный)
   * `('S5','S7')→'14'` (неправильный неуверенный)
   * Меняется только метка `S  1` на `S {11|12|13|14}`.
* Выход:
   * Пишет новый `.vmrk`; при пакетной логике файлы `'_NewMarkers.vmrk'` считаются уже обработанными.

---

## Часть E — Преобразование файлов .vmrk в файлы .Markers для импорта в BrainVision Analyzer и создание финальной версии с тремя стадиями → `three_stages/*_three_stages.Markers`
* Конвертер форматов преобразует каждый `.vmrk` в папке `input_dir` в файл `.Markers`, совместимый с BrainVision Analyzer, с именем `<basename>_Raw Data.Markers`.
* Скрипт читает исходный `.vmrk`, находит раздел `[Marker Infos]`, парсит строки `MkN=`, пропускает системную запись `New Segment`, возвращает запятые в описания (`\1` → `,`), отображает канал `0` как `All` и записывает CSV-подобный `.Markers` с двухстрочным заголовком (`Sampling rate: {sampling_rate}Hz, SamplingInterval: {sampling_interval}ms` и названия столбцов).
* Частота дискретизации и интервал редактируются через `sampling_rate` и `sampling_interval`.

* Функция: `process_three_stages(markers_in, markers_out)`.
* Стадии задаются заголовками `S11/S12/S13/S14`:
   * Первая цифра — номер стадии.
   * Вторая — тип ответа.
   * В пределах одной пробы последний разряд совпадает.
* Внутри сегмента стадии:
   * `S2 → S2x`, `S3 → S3x` (где `x` — последний разряд текущего заголовка `S1x`).
* Пакетная обработка:
   * Создаёт папку **`three_stages/`**.
   * Обходит все `*.Markers`, **пропуская** уже оканчивающиеся на `'_three_stages.Markers'`.
   * Пишет результат в `three_stages/` с суффиксом `_three_stages`.

---

## Структура файлов
* Единственная папка, которую **создаёт сам код**, — это **`three_stages/`**.
* Все входные/выходные файлы (`.vhdr/.eeg/.vmrk/.Markers` и `BaseReport_{num}_*CORR*.csv`) ищутся/создаются **в текущей рабочей директории**.
* Имена выходов, встречающиеся в коде:
   * `photo_markers.csv`.
   * `*_ConfMarkers.vmrk`.
   * `three_stages/*_three_stages.Markers`.

---

## Как запускать 
* Из папки, где лежат ваши `.vmrk/.Markers` и соответствующие `BaseReport_{num}_*CORR*.csv`:
   * `python editing_and_restoring_eeg_markers.py`
* При выполнении:
   * Сначала отработает пакет для `*.vmrk` → `*_ConfMarkers.vmrk` (Часть C).
   * Затем пакет для `*.Markers` → `three_stages/*_three_stages.Markers` (Часть E).

---

## Параметры, которые можно менять
* Фотодетектор:
   * `vhdr_path`, `chunk_duration`, `threshold`, `min_interval_sec`.
* Чтение CSV:
   * Автоопределение `sep` по первой строке (`,` или `;`).
   * Поиск `conf_col` по подстроке `'Увер'`; значения — `1/0`.
* Трёхстадийное преобразование:
   * Параметров нет; поведение фиксировано логикой функции.
