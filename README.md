# MeTube

> **_NOTE:_**  32-bit ARM builds have been retired (a full year after [other major players](https://www.linuxserver.io/blog/a-farewell-to-arm-hf)), as new Node versions don't support them, and continued security updates and dependencies require new Node versions. Please migrate to a 64-bit OS to continue receiving MeTube upgrades.

![Build Status](https://github.com/alexta69/metube/actions/workflows/main.yml/badge.svg)
![Docker Pulls](https://img.shields.io/docker/pulls/alexta69/metube.svg)

Web GUI for youtube-dl (using the [yt-dlp](https://github.com/yt-dlp/yt-dlp) fork) with playlist support. Allows you to download videos from YouTube and [dozens of other sites](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md).

![screenshot1](https://github.com/alexta69/metube/raw/master/screenshot.gif)

## Run using Docker

```bash
docker run -d -p 8081:8081 -v /path/to/downloads:/downloads ghcr.io/alexta69/metube
```

## Run using docker-compose

```yaml
version: "3"
services:
  metube:
    image: ghcr.io/alexta69/metube
    container_name: metube
    restart: unless-stopped
    ports:
      - "8081:8081"
    volumes:
      - /path/to/downloads:/downloads
```

## Configuration via environment variables

Certain values can be set via environment variables, using the `-e` parameter on the docker command line, or the `environment:` section in docker-compose.

* __UID__: user under which MeTube will run. Defaults to `1000`.
* __GID__: group under which MeTube will run. Defaults to `1000`.
* __UMASK__: umask value used by MeTube. Defaults to `022`.
* __DEFAULT_THEME__: default theme to use for the ui, can be set to `light`, `dark` or `auto`. Defaults to `auto`.
* __DOWNLOAD_DIR__: path to where the downloads will be saved. Defaults to `/downloads` in the docker image, and `.` otherwise.
* __AUDIO_DOWNLOAD_DIR__: path to where audio-only downloads will be saved, if you wish to separate them from the video downloads. Defaults to the value of `DOWNLOAD_DIR`.
* __DOWNLOAD_DIRS_INDEXABLE__: if `true`, the download dirs (__DOWNLOAD_DIR__ and __AUDIO_DOWNLOAD_DIR__) are indexable on the webserver. Defaults to `false`.
* __CUSTOM_DIRS__: whether to enable downloading videos into custom directories within the __DOWNLOAD_DIR__ (or __AUDIO_DOWNLOAD_DIR__). When enabled, a drop-down appears next to the Add button to specify the download directory. Defaults to `true`.
* __CREATE_CUSTOM_DIRS__: whether to support automatically creating directories within the __DOWNLOAD_DIR__ (or __AUDIO_DOWNLOAD_DIR__) if they do not exist. When enabled, the download directory selector becomes supports free-text input, and the specified directory will be created recursively. Defaults to `true`.
* __STATE_DIR__: path to where the queue persistence files will be saved. Defaults to `/downloads/.metube` in the docker image, and `.` otherwise.
* __TEMP_DIR__: path where intermediary download files will be saved. Defaults to `/downloads` in the docker image, and `.` otherwise.
  * Set this to an SSD or RAM filesystem (e.g., `tmpfs`) for better performance
  * __Note__: Using a RAM filesystem may prevent downloads from being resumed
* __DELETE_FILE_ON_TRASHCAN__: if `true`, downloaded files are deleted on the server, when they are trashed from the "Completed" section of the UI. Defaults to `false`.
* __URL_PREFIX__: base path for the web server (for use when hosting behind a reverse proxy). Defaults to `/`.
* __PUBLIC_HOST_URL__: base URL for the download links shown in the UI for completed files. By default MeTube serves them under its own URL. If your download directory is accessible on another URL and you want the download links to be based there, use this variable to set it.
* __PUBLIC_HOST_AUDIO_URL__: same as PUBLIC_HOST_URL but for audio downloads.
* __OUTPUT_TEMPLATE__: the template for the filenames of the downloaded videos, formatted according to [this spec](https://github.com/yt-dlp/yt-dlp/blob/master/README.md#output-template). Defaults to `%(title)s.%(ext)s`.
* __OUTPUT_TEMPLATE_CHAPTER__: the template for the filenames of the downloaded videos, when split into chapters via postprocessors. Defaults to `%(title)s - %(section_number)s %(section_title)s.%(ext)s`.
* __YTDL_OPTIONS__: Additional options to pass to youtube-dl, in JSON format. [See available options here](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/YoutubeDL.py#L183). They roughly correspond to command-line options, though some do not have exact equivalents here, for example `--recode-video` has to be specified via `postprocessors`. Also note that dashes are replaced with underscores.
* __YTDL_OPTIONS_FILE__: A path to a JSON file that will be loaded and used for populating `YTDL_OPTIONS` above. Please note that if both `YTDL_OPTIONS_FILE` and `YTDL_OPTIONS` are specified, the options in `YTDL_OPTIONS` take precedence.

The following example value for `YTDL_OPTIONS` embeds English subtitles and chapter markers (for videos that have them), and also changes the permissions on the downloaded video and sets the file modification timestamp to the date of when it was downloaded:

```yaml
    environment:
      - 'YTDL_OPTIONS={"writesubtitles":true,"subtitleslangs":["en","-live_chat"],"updatetime":false,"postprocessors":[{"key":"Exec","exec_cmd":"chmod 0664","when":"after_move"},{"key":"FFmpegEmbedSubtitle","already_have_subtitle":false},{"key":"FFmpegMetadata","add_chapters":true}]}'
```

The following example value for `OUTPUT_TEMPLATE` sets:
- playlist name and author, if present
- playlist number and count, if present (zero-padded, if needed)
- video author, title and release date in YYYY-MM-DD format, falling back to *UNKNOWN_...* if missing
- sanitises everything for valid UNIX filename

```yaml
    environment:
      - 'OUTPUT_TEMPLATE=%(playlist_title&Playlist |)S%(playlist_title|)S%(playlist_uploader& by |)S%(playlist_uploader|)S%(playlist_autonumber& - |)S%(playlist_autonumber|)S%(playlist_count& of |)S%(playlist_count|)S%(playlist_autonumber& - |)S%(uploader,creator|UNKNOWN_AUTHOR)S - %(title|UNKNOWN_TITLE)S - %(release_date>%Y-%m-%d,upload_date>%Y-%m-%d|UNKNOWN_DATE)S.%(ext)s'
```

## Using browser cookies

In case you need to use your browser's cookies with MeTube, for example to download restricted or private videos:

* Add the following to your docker-compose.yml:

```yaml
    volumes:
      - /path/to/cookies:/cookies
    environment:
      - YTDL_OPTIONS={"cookiefile":"/cookies/cookies.txt"}
```

* Install in your browser an extension to extract cookies:
  * [Firefox](https://addons.mozilla.org/en-US/firefox/addon/export-cookies-txt/)
  * [Chrome](https://chrome.google.com/webstore/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc)
* Extract the cookies you need with the extension and rename the file `cookies.txt`
* Drop the file in the folder you configured in the docker-compose.yml above
* Restart the container

## Browser extensions

Browser extensions allow right-clicking videos and sending them directly to MeTube. Please note that if you're on an HTTPS page, your MeTube instance must be behind an HTTPS reverse proxy (see below) for the extensions to work.

__Chrome:__ contributed by [Rpsl](https://github.com/rpsl). You can install it from [Google Chrome Webstore](https://chrome.google.com/webstore/detail/metube-downloader/fbmkmdnlhacefjljljlbhkodfmfkijdh) or use developer mode and install [from sources](https://github.com/Rpsl/metube-browser-extension).

__Firefox:__ contributed by [nanocortex](https://github.com/nanocortex). You can install it from [Firefox Addons](https://addons.mozilla.org/en-US/firefox/addon/metube-downloader) or get sources from [here](https://github.com/nanocortex/metube-firefox-addon).

## iOS Shortcut

[rithask](https://github.com/rithask) has created an iOS shortcut to send the URL to MeTube from Safari. Initially, you'll need to enter the server address and port, but after that, it will be saved and you can just run the shortcut from the share menu in Safari. The address should include the protocol (http/https) and the port, if it's not the default 80/443. For example: `https://metube.example.com` or `http://192.168.1.1:8081`. The shortcut can be found [here](https://www.icloud.com/shortcuts/f1548df15b734418a77a709103bc1dd5).

## iOS Compatibility

iOS has strict requirements for video files, requiring h264 or h265 video codec and aac audio codec in MP4 container. This can sometimes be a lower quality than the best quality available. To accommodate iOS requirements, when downloading a MP4 format you can choose "Best (iOS)" to get the best quality formats as compatible as possible with iOS requirements.

## Bookmarklet

[kushfest](https://github.com/kushfest) has created a Chrome bookmarklet for sending the currently open webpage to MeTube. Please note that if you're on an HTTPS page, your MeTube instance must be behind an HTTPS reverse proxy (see below) for the bookmarklet to work.

GitHub doesn't allow embedding JavaScript as a link, so the bookmarklet has to be created manually by copying the following code to a new bookmark you create on your bookmarks bar. Change the hostname in the URL below to point to your MeTube instance.

```javascript
javascript:!function(){xhr=new XMLHttpRequest();xhr.open("POST","https://metube.domain.com/add");xhr.withCredentials=true;xhr.send(JSON.stringify({"url":document.location.href,"quality":"best"}));xhr.onload=function(){if(xhr.status==200){alert("Sent to metube!")}else{alert("Send to metube failed. Check the javascript console for clues.")}}}();
```

[shoonya75](https://github.com/shoonya75) has contributed a Firefox version:

```javascript
javascript:(function(){xhr=new XMLHttpRequest();xhr.open("POST","https://metube.domain.com/add");xhr.send(JSON.stringify({"url":document.location.href,"quality":"best"}));xhr.onload=function(){if(xhr.status==200){alert("Sent to metube!")}else{alert("Send to metube failed. Check the javascript console for clues.")}}})();
```

The above bookmarklets use `alert()` as a success/failure notification. The following will show a toast message instead:

Chrome:

```javascript
javascript:!function(){function notify(msg) {var sc = document.scrollingElement.scrollTop; var text = document.createElement('span');text.innerHTML=msg;var ts = text.style;ts.all = 'revert';ts.color = '#000';ts.fontFamily = 'Verdana, sans-serif';ts.fontSize = '15px';ts.backgroundColor = 'white';ts.padding = '15px';ts.border = '1px solid gainsboro';ts.boxShadow = '3px 3px 10px';ts.zIndex = '100';document.body.appendChild(text);ts.position = 'absolute'; ts.top = 50 + sc + 'px'; ts.left = (window.innerWidth / 2)-(text.offsetWidth / 2) + 'px'; setTimeout(function () { text.style.visibility = "hidden"; }, 1500);}xhr=new XMLHttpRequest();xhr.open("POST","https://metube.domain.com/add");xhr.send(JSON.stringify({"url":document.location.href,"quality":"best"}));xhr.onload=function() { if(xhr.status==200){notify("Sent to metube!")}else {notify("Send to metube failed. Check the javascript console for clues.")}}}();
```

Firefox:

```javascript
javascript:(function(){function notify(msg) {var sc = document.scrollingElement.scrollTop; var text = document.createElement('span');text.innerHTML=msg;var ts = text.style;ts.all = 'revert';ts.color = '#000';ts.fontFamily = 'Verdana, sans-serif';ts.fontSize = '15px';ts.backgroundColor = 'white';ts.padding = '15px';ts.border = '1px solid gainsboro';ts.boxShadow = '3px 3px 10px';ts.zIndex = '100';document.body.appendChild(text);ts.position = 'absolute'; ts.top = 50 + sc + 'px'; ts.left = (window.innerWidth / 2)-(text.offsetWidth / 2) + 'px'; setTimeout(function () { text.style.visibility = "hidden"; }, 1500);}xhr=new XMLHttpRequest();xhr.open("POST","https://metube.domain.com/add");xhr.send(JSON.stringify({"url":document.location.href,"quality":"best"}));xhr.onload=function() { if(xhr.status==200){notify("Sent to metube!")}else {notify("Send to metube failed. Check the javascript console for clues.")}}})();
```

## Running behind a reverse proxy

It's advisable to run MeTube behind a reverse proxy, if authentication and/or HTTPS support are required.

When running behind a reverse proxy which remaps the URL (i.e. serves MeTube under a subdirectory and not under root), don't forget to set the URL_PREFIX environment variable to the correct value.

If you're using the [linuxserver/swag](https://docs.linuxserver.io/general/swag) image for your reverse proxying needs (which I can heartily recommend), it already includes ready snippets for proxying MeTube both in [subfolder](https://github.com/linuxserver/reverse-proxy-confs/blob/master/metube.subfolder.conf.sample) and [subdomain](https://github.com/linuxserver/reverse-proxy-confs/blob/master/metube.subdomain.conf.sample) modes under the `nginx/proxy-confs` directory in the configuration volume. It also includes Authelia which can be used for authentication.

### NGINX

```nginx
location /metube/ {
        proxy_pass http://metube:8081;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
}
```

Note: the extra `proxy_set_header` directives are there to make WebSocket work.

### Apache

Contributed by [PIE-yt](https://github.com/PIE-yt). Source [here](https://gist.github.com/PIE-yt/29e7116588379032427f5bd446b2cac4).

```apache
# For putting in your Apache sites site.conf
# Serves MeTube under a /metube/ subdir (http://yourdomain.com/metube/)
<Location /metube/>
    ProxyPass http://localhost:8081/ retry=0 timeout=30
    ProxyPassReverse http://localhost:8081/
</Location>

<Location /metube/socket.io>
    RewriteEngine On
    RewriteCond %{QUERY_STRING} transport=websocket    [NC]
    RewriteRule /(.*) ws://localhost:8081/socket.io/$1 [P,L]
    ProxyPass http://localhost:8081/socket.io retry=0 timeout=30
    ProxyPassReverse http://localhost:8081/socket.io
</Location>
```

### Caddy

The following example Caddyfile gets a reverse proxy going behind [caddy](https://caddyserver.com).

```caddyfile
example.com {
  route /metube/* {
    uri strip_prefix metube
    reverse_proxy metube:8081
  }
}
```

## Updating yt-dlp

The engine which powers the actual video downloads in MeTube is [yt-dlp](https://github.com/yt-dlp/yt-dlp). Since video sites regularly change their layouts, frequent updates of yt-dlp are required to keep up.

There's an automatic nightly build of MeTube which looks for a new version of yt-dlp, and if one exists, the build pulls it and publishes an updated docker image. Therefore, in order to keep up with the changes, it's recommended that you update your MeTube container regularly with the latest image.

I recommend installing and setting up [watchtower](https://github.com/containrrr/watchtower) for this purpose.

## Troubleshooting and submitting issues

Before asking a question or submitting an issue for MeTube, please remember that MeTube is only a UI for [yt-dlp](https://github.com/yt-dlp/yt-dlp). Any issues you might be experiencing with authentication to video websites, postprocessing, permissions, other `YTDL_OPTIONS` configurations which seem not to work, or anything else that concerns the workings of the underlying yt-dlp library, need not be opened on the MeTube project. In order to debug and troubleshoot them, it's advised to try using the yt-dlp binary directly first, bypassing the UI, and once that is working, importing the options that worked for you into `YTDL_OPTIONS`.

In order to test with the yt-dlp command directly, you can either download it and run it locally, or for a better simulation of its actual conditions, you can run it within the MeTube container itself. Assuming your MeTube container is called `metube`, run the following on your Docker host to get a shell inside the container:

```bash
docker exec -ti metube sh
cd /downloads
```

Once there, you can use the yt-dlp command freely.

## Building and running locally

Make sure you have node.js and Python 3.11 installed.

```bash
cd metube/ui
# install Angular and build the UI
npm install
node_modules/.bin/ng build
# install python dependencies
cd ..
pip3 install pipenv
pipenv install
# run
pipenv run python3 app/main.py
```

A Docker image can be built locally (it will build the UI too):

```bash
docker build -t metube .
```

## Development notes

* The above works on Windows and macOS as well as Linux.
* If you're running the server in VSCode, your downloads will go to your user's Downloads folder (this is configured via the environment in .vscode/launch.json).

16.02.26
Ось результати аналізу та стратегія трансформації для проекту **MeTube**, підготовлені у форматі для копіювання в Notion.

---

# 📑 Звіт AI-консультанта: Проект "MeTube"

**MeTube** — це веб-інтерфейс для самостійного хостингу, призначений для завантаження відео з YouTube та інших сайтів за допомогою бібліотек `youtube-dl` або `yt-dlp`.

## 🧬 Частина 1: "ДНК" Проекту

Логіку коду MeTube можна розкласти на такі **атомарні функції**:

*   **Прийом запитів (URL Ingestion):** Обробка вхідних посилань через веб-інтерфейс, розширення браузера або Bookmarklets через POST-запити на ендпоінт `/add`.
*   **Керування чергою (Queue Management):** Послідовна обробка завдань на завантаження з можливістю збереження стану (Persistence) у файловій системі.
*   **Конфігурація середовища (Environment Parsing):** Гнучке налаштування шляхів завантаження, прав доступу (UID/GID) та параметрів інтерфейсу через змінні оточення.
*   **Оркестрація завантаження (Download Orchestration):** Динамічне формування команд для `yt-dlp` з використанням складних шаблонів імен (Output Templates) та параметрів якості.
*   **Пост-обробка (Post-processing):** Використання FFmpeg для вбудовування субтитрів, розділення на глави, додавання метаданих або зміни кодеків (наприклад, для сумісності з iOS).
*   **Управління файлами (File Management):** Організація завантажених файлів у кастомні директорії та можливість їх видалення через інтерфейс.

### 💎 Головна технічна цінність
Головна цінність проекту полягає в **уніфікації та спрощенні складного CLI-інструментарію `yt-dlp`**. MeTube надає користувачеві зручний графічний інтерфейс для керування завантаженнями, який легко розгортається в Docker та підтримує складні сценарії автоматизації (через API та розширення), залишаючись при цьому легким і продуктивним.

---

## 🚀 Частина 2: "Трансформація" (Інтеграція з Gemini LLM)

Інтеграція з Gemini перетворює MeTube із простого завантажувача на **інтелектуальний центр управління медіа-контентом**.

### Як зміниться функціонал?
1.  **Інтелектуальна сортувальник:** Gemini може аналізувати назву та опис відео, автоматично створюючи тематичні папки (наприклад, "Навчання/Програмування" або "Розваги/Музика") замість ручного вибору [Поза межами джерел].
2.  **Автоматичне резюмування (Summarization):** Після завантаження AI може генерувати короткий зміст (TL;DR) відео або транскрибацію, зберігаючи її у текстовому файлі поруч із медіа [Поза межами джерел].
3.  **Smart Template Generator:** Замість складного ручного налаштування `OUTPUT_TEMPLATE`, користувач може просто сказати: *"Хочу назву відео, автора та дату у форматі Рік-Місяць"*, і Gemini згенерує правильний синтаксис для `yt-dlp` [10, Поза межами джерел].

### Сценарій сервісу (MeTube + Gemini + ID_{$})

Створення сервісу **"AI Media Archiver"** на вашому сайті.

**Алгоритм роботи:**
1.  **Ввід (ID_{$}):** Користувач вставляє посилання на вашому сайті. Ваш базовий Python-скрипт `ID_{$}` перехоплює його.
2.  **Аналіз (Gemini):** Скрипт надсилає URL до Gemini. Модель перевіряє тематику відео та визначає, чи відповідає воно правилам сайту, а також пропонує найкращу якість завантаження.
3.  **Передача (MeTube API):** Скрипт `ID_{$}` робить POST-запит на ваш інстанс MeTube: `https://your-metube.com/add` із параметрами, які підказав AI.
4.  **Обробка:** MeTube завантажує відео. Завдяки налаштованим `YTDL_OPTIONS`, воно відразу отримує субтитри та глави.
5.  **Фіналізація:** Після завершення Gemini створює короткий опис для вашої бібліотеки на сайті, а `ID_{$}` оновлює базу даних сайту посиланням на локальний файл.

---

## 📋 План дій для Notion
| Крок | Дія | Результат |
| :--- | :--- | :--- |
| **1** | Розгортання через `docker-compose` | Працюючий бекенд для завантажень |
| **2** | Налаштування `YTDL_OPTIONS` для метаданих | Покращена якість медіа-архіву |
| **3** | Створення Python-містка (`ID_{$}`) | Зв'язок між вашим сайтом та MeTube API |
| **4** | Використання **GitHub Spark** | Швидка побудова UI для вашого нового сервісу |

---

### 💡 Резюме

**Суть:** **Self-hosted веб-інтерфейс для завантаження відео**.

**AI-Роль:** **Інтелектуальна обробка та категоризація медіа-контенту**.

*(Примітка: Сценарії використання Gemini та трансформації функціоналу є аналітичним проектуванням на основі сучасних можливостей AI, оскільки в документації проекту MeTube пряма інтеграція з LLM наразі не описана).*
