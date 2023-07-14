+++
title = "使用 go-musicfox 來聽 網易雲"
slug = "musicfox"
date = 2023-07-13T22:55:00-04:00
lastmod = 2023-07-14T16:02:35-04:00
tags = ["music", "netease", "lastfm", "emacs", "hammerspoon", "macos"]
draft = false
weight = 2002
toc = true
comments = true
author = "傅裕夫"
+++

## go-musicfox {#go-musicfox}

[go-musicfox](https://github.com/go-musicfox/go-musicfox) 是一個命令行的網易雲客戶端，支援不少功能例如 [last.fm](https://www.last.fm/home) 上報。

{{< figure src="https://github.com/go-musicfox/go-musicfox/raw/master/previews/main.png" caption="<span class=\"figure-number\">Figure 1: </span>go-musicfox 界面" >}}

本文只做基本介紹，和我踩的一些坑 (還提了幾個 issue)，還有一些自動化工具撰寫。

詳細使用方法，請洽 GitHub。


## 播放引擎選擇 {#播放引擎選擇}

`go-musicfox` 支援 beep, mpd, osx 三種引擎。在本文撰寫時，我實際測試過後的結果。

-   beep: 播放一首歌後 panic 。
-   osx: 歌詞會 delay，因為美國連到網易雲伺服器速度感人 (2MB/s)。
-   mpd: 最穩定，除了下面會提到的坑。


## mpd 踩坑 {#mpd-踩坑}

網易雲對 flac 檔案， HTTP Header仍然使用 `audio/mpeg` ，導致 mpd 誤認為 mp3 而使用錯誤的解碼器。

<details class="code-details"
                 style ="padding: 1em;
                          background-color: #e5f5e5;
                          /* background-color: pink; */
                          border-radius: 15px;
                          color: hsl(157 75%);
                          font-size: 0.9em;
                          box-shadow: 0.05em 0.1em 5px 0.01em  #00000057;">
                  <summary>
                    <strong>
                      <font face="Courier" size="3" color="green">
                         HTTP Header
                      </font>
                    </strong>
                  </summary>

<div class="verse">

HTTP/1.1 200 OK<br />
Server: Tengine<br />
Content-Type: audio/mpeg; charset=UTF-8<br />
Content-Length: 62865627                                                                                                                                                  Connection: keep-alive<br />
Date: Mon, 12 Jun 2023 07:18:15 GMT<br />

</div>


</details>


### 解決辦法 {#解決辦法}

編譯 mpd 時，記得把 FFmpeg support開啟。如果你用 Homebrew 安裝，不用特別處理，FFmpeg 已經被編譯進去了。

如果你和我一樣使用 MacPorts，需要使用以下指令安裝來啟用 FFmpeg decoder。

```sh
sudo port install mpd +ffmpeg
```

然後確定 decoder plugins 裡有 FFmpeg.

```sh
mpd --version
```


### ~/.mpd/mpd.conf {#dot-mpd-mpd-dot-conf}

特別注意的是要停掉 mad，這樣就會 fallback 到 FFmpeg。

```cfg
# musicfox 不會用到，任意設定即可。
music_directory "~/Music"
playlist_directory "~/.mpd/playlists"
db_file "~/.mpd/mpd.db"
log_file "~/.mpd/mpd.log"
pid_file "~/.mpd/mpd.pid"
state_file "~/.mpd/mpdstate"
bind_to_address "~/.mpd/socket"

# 音量平衡
volume_normalization "yes"


audio_output {
    type "osx"
    name "My Mac Device"
    mixer_type "software"
}

# wyy return Content-Type: audio/mpeg; for flac files!!
# just disable mad and fallback to ffmpeg; and ffmpeg will do the correct job
decoder {
        plugin "mad"
        enabled "no"
}
```

記得 `go-musicfox.ini` 要改成對應的值。


## 遠程控制 musicfox {#遠程控制-musicfox}

因為 musicfox 需要按鍵來控制，所以需要一個程式可以開啟 musicfox 並對他發送按鍵的事件。沒錯，就是你我天天用到的 `tmux` !

方法也很簡單，開啟一個 session (假設名字叫 musicfox)，然後 musicfox 在 session裡的第一個 window 且是第一個 pane. 就可以使用一下指令來發送空白鍵 (Space) 給 musicfox.

```sh
tmux send-keys -t musicfox:1 Space
```

之後就能遠程控制 musicfox了~


## 集成 Emacs {#集成-emacs}

我個人是重度 Emacs 用戶，很大一段時間視窗都在 Emacs上，所以能從 Emacs裡控制 musicfox 對我來說很重要。當然不可能有現成的 Package，所以只能自己寫摟。

最後實現的方法是Emacs裡的 [vterm](https://github.com/akermu/emacs-libvterm) package.

方法和 tmux 差不多，不過是改用 `(vterm-insert)` 來發送按鍵。不多做介紹，以下貼 code，歡迎交流討論。


### init-musicfox.el {#init-musicfox-dot-el}

```emacs-lisp
(require 'vterm)

(defvar musicfox-mode-map
  (let ((map (make-sparse-keymap)))
    (define-key map "]" #'musicfox-next)
    map)
  "MusicFox Keymap")

(defvar musicfox-buffer-name "*MusicFox*")

(define-minor-mode musicfox-mode "MusicFox")

(defun musicfox ()
  (interactive)
  (let* ((vterm-buffer-name-string nil)
         (vterm-buffer-name musicfox-buffer-name)
         (b (get-buffer vterm-buffer-name)))
    (if b
        (if (eq (current-buffer) b)
            (bury-buffer)
            (pop-to-buffer-same-window b))
      (vterm)
      (setq-local vterm-buffer-name vterm-buffer-name)
      (setq-local vterm-buffer-name-string vterm-buffer-name-string)
      (local-unset-key (kbd "<f11>")) ;; should match the key in init.el
      (local-unset-key (kbd "C-SPC"))
      (local-set-key "]" #'musicfox-next)
      (vterm-insert "musicfox\n")
      (musicfox-mode))
    ))

(defmacro with-musicfox (body)
  `(with-current-buffer musicfox-buffer-name
                        ,body))

(defun musicfox-next ()
  (interactive)
  (with-musicfox
   (vterm-insert "]")))

(defun musicfox-prev ()
  (interactive)
  (with-musicfox
   (vterm-insert "[")))

(defun musicfox-playpause ()
  (interactive)
  (with-musicfox
   (vterm-insert (kbd "SPC"))))

(defun musicfox-favorite ()
  (interactive)
  (with-musicfox
   (vterm-insert (kbd ","))))

(defun musicfox-download ()
  (interactive)
  (with-musicfox
   (vterm-insert (kbd "d"))))

(defun musicfox-volumn-decrease ()
  (interactive)
  (with-musicfox
   (vterm-insert (kbd "-"))))

(defun musicfox-volumn-increase ()
  (interactive)
  (with-musicfox
   (vterm-insert (kbd "="))))

(spacemacs/declare-prefix
  "am" "music"
  )

;; TODO popup log file with read-only

(defun musixfox-popup-log ()
  (interactive)

  )

(defun musixfox-config ()
  (interactive)
  (find-file "~/Library/Application Support/go-musicfox/go-musicfox.ini"))

;; replicate keybindings and some custom aliases
(spacemacs/set-leader-keys
  "amn" 'musicfox-next
  "am]" 'musicfox-next
  "amb" 'musicfox-prev
  "am[" 'musicfox-prev
  "amp" 'musicfox-playpause
  "am SPC" 'musicfox-playpause
  "amf" 'musicfox-favorite
  "am," 'musicfox-favorite
  "amd" 'musicfox-download
  "am-" 'musicfox-volumn-decrease
  "am=" 'musicfox-volumn-increase
  "aml" 'musixfox-popup-log
  )

(provide 'init-musicfox)
```

之後 `(require 'init-musicfox)` 然後 `M-x musicfox` 。

裡面用到 `spacemacs/set-leader-keys` 是 Spacemacs 專屬函數，其他用戶可以自行綁定其他熱鍵。


## Hammerspoon 實現全局熱鍵控制 musicfox 和 iTunes (Music) {#hammerspoon-實現全局熱鍵控制-musicfox-和-itunes--music}

[Hammerspoon](https://www.hammerspoon.org/) 是一款 macOS 強大的自動化工具，基本上[大部分功能都有](https://www.hammerspoon.org/docs/index.html)。

除了 musicfox，我主要使用 macOS 預設的Music。基本我使用網易雲只是把常聽的音樂下載，然後導入到 Music之後就不用了.


### musicfox-cli {#musicfox-cli}

使用 emasclient 直接調用 Emacs對應的 musicfox 指令 。使用 tmux方法的可以自行撰寫類似的腳本。

```sh
#!/bin/bash -e
/usr/local/bin/emacsclient -e "(musicfox-$1)" > /dev/null
```

之後可以使用 `musicfox-cli playpause` 之類的指令來控制。


### musicfox.lua {#musicfox-dot-lua}

這個檔案是 `musicfox-cli` 的 wrapper，使用 `hs.tasks` 調用。

```lua
local module = {}
CLI_PATH = '/Users/fuyu0425/bin/musicfox-cli'

function callback(exitCode, stdOut, stdErr)
  -- do something when the task is completed
  if exitCode == 0 then
    print('Task succeeded')
    print('Output:', stdOut)
  else
    print('Task failed with exit code', exitCode)
    print('Error:', stdErr)
  end
end

function module.playpause ()
  hs.task.new(CLI_PATH,
              nil,
              {'playpause'}):start()
end

function module.next ()
  hs.task.new(CLI_PATH,
              nil,
              {'next'}):start()
end

function module.prev ()
  hs.task.new(CLI_PATH,
              nil,
              {'prev'}):start()
end

return module
```


### music.lua {#music-dot-lua}

1.  建立一個 Menu bar toggle 來決定目前要控制 iTunes (Apple Music) 還是 musicfox.
2.  綁定全局熱鍵來控制播放器。(Alt + Ctrl + n/b/p)

<!--listend-->

```lua
musicfox = require('musicfox')

ITUNES = 'iTunes'
MUSICFOX = 'MusicFox'

-- Choose your default value to be ITUNES or MUSICFOX
-- CURRENT_MUSIC_APP = ITUNES
CURRENT_MUSIC_APP = MUSICFOX

musicAppMenu = hs.menubar.new()

-- Define state
state = CURRENT_MUSIC_APP

-- Function to toggle state
function toggleMusicAppState()
  if state == ITUNES then
    state = MUSICFOX
    CURRENT_MUSIC_APP = MUSICFOX
  else
    state = ITUNES
    CURRENT_MUSIC_APP = ITUNES
  end
  -- Update the menu title
  musicAppMenu:setTitle(state)
end

-- Set initial state
musicAppMenu:setTitle(state)

-- Set the menu to be our toggle function
musicAppMenu:setClickCallback(toggleMusicAppState)

hs.hotkey.bind({ "alt", "ctrl" }, "p", nil, function()
    if CURRENT_MUSIC_APP == ITUNES then
      hs.itunes.playpause()
    else
      musicfox.playpause()
    end
end)


hs.hotkey.bind({ "alt", "ctrl" }, "n", nil, function()
    if CURRENT_MUSIC_APP == ITUNES then
      hs.itunes.next()
    else
      musicfox.next()
    end
end)

hs.hotkey.bind({ "alt", "ctrl" }, "b", nil, function()
    if CURRENT_MUSIC_APP == ITUNES then
      hs.itunes.previous()
    else
      musicfox.prev()
    end
end)
```


#### 效果 {#效果}

{{< figure src="/images/player-toggle.gif" caption="<span class=\"figure-number\">Figure 2: </span>Menubar toggle" >}}
