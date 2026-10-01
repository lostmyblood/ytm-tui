#!/usr/bin/env bash
# ytm-dl.sh - YouTube Music downloader (minimal monochrome TUI)
# Needs: yt-dlp ffmpeg jq   Optional (cover preview): chafa curl
# Env:   MUSIC_DIR=<default folder>  JOBS=<parallel downloads, default 3>
#        COVER=0|text|kitty|iterm|sixel  (default: detect your terminal; real image when supported)

MUSIC_DIR=${MUSIC_DIR:-$HOME/Music}
JOBS=${JOBS:-3}
COVER=${COVER:-1}
FORMATS=(mp3 m4a opus flac vorbis aac wav alac)
CROP='crop=min(iw\,ih):min(iw\,ih)'
[[ $JOBS =~ ^[1-9][0-9]*$ ]] || JOBS=3

# how to draw the cover: kitty / iterm / sixel = real image, text = monochrome block art
case $COVER in
  kitty|iterm|sixel|text) CFMT=$COVER ;;
  *) if [[ -n $TMUX ]]; then CFMT=text
     elif [[ -n $KITTY_WINDOW_ID || $TERM == *kitty* || $TERM == *ghostty* || $TERM_PROGRAM == ghostty ]]; then CFMT=kitty
     elif [[ $TERM_PROGRAM == WezTerm || $TERM_PROGRAM == iTerm.app || $LC_TERMINAL == iTerm2 ]]; then CFMT=iterm
     elif [[ $TERM == foot* || $TERM == mlterm* || $TERM == contour* || $TERM == yaft* ]]; then CFMT=sixel
     else CFMT=text; fi ;;
esac
IMG_SIG=

BOLD=$'\033[1m' DIM=$'\033[2m' UL=$'\033[4m' REV=$'\033[7m' RST=$'\033[0m'
FR=(⠋ ⠙ ⠹ ⠸ ⠼ ⠴ ⠦ ⠧ ⠇ ⠏)

for dep in yt-dlp ffmpeg jq; do
  command -v "$dep" >/dev/null || { echo "missing dependency: $dep"; exit 1; }
done
[[ -t 0 && -t 1 ]] || { echo "ytm-dl.sh needs an interactive terminal"; exit 1; }

JS=()
for r in node bun; do command -v "$r" >/dev/null && JS+=(--js-runtimes "$r"); done

# ---------- state ----------
# one entry per queued link; ST = check | ready | queued | run | ok | err
ITEM_ARRAYS=(LINKS NOPLS ISPL ARTISTS NAMES FOLDERS PIDS UIDS ST MSG ITFMT ITDEST SHOWN)
for a in "${ITEM_ARRAYS[@]}"; do eval "$a=()"; done
UIDN=0 FOCUS=0 SEL=0 FI=0 TICK=0 DIRTY=1 ACTIVE=0 BATCH=0
BUF=("" "$MUSIC_DIR") POS=(0 ${#MUSIC_DIR})
NOTE_SYM= NOTE=
W=$(tput cols); H=$(tput lines)

TMP=$(mktemp -d)
OLD_STTY=$(stty -g)
cleanup() {
  kill "${PIDS[@]}" 2>/dev/null
  stty "$OLD_STTY" 2>/dev/null
  printf '\033[?25h\033[?1049l'
  rm -rf "$TMP"
}
trap cleanup EXIT
trap 'exit 130' INT TERM
trap 'W=$(tput cols); H=$(tput lines); DIRTY=1' WINCH

note() { NOTE_SYM=$1 NOTE=$2; DIRTY=1; }
if ! command -v deno >/dev/null && (( ${#JS[@]} == 0 )); then
  note '!' "no javascript runtime found (install deno) - extraction may be slow"
fi

# ---------- metadata ----------
parse_info() {
  IFS=$'\x1f' read -r IS_PL NAME ARTIST THUMB < <(jq -r '
    (if ._type == "playlist" then (.entries[0] // {}) else . end) as $f
    | ($f.uploader // $f.channel // "") as $u
    | [ (._type == "playlist"),
        (if ._type == "playlist" then (.title // "") | sub("^(Album|Single|EP) - "; "")
         else ($f.track // $f.title // "") end),
        (($f.artists // [] | .[0])
           // (($f.artist // ($u | sub(" - Topic$"; "")) // .uploader // .channel // "") | split(", ") | .[0])),
        ([ (.thumbnails // [] | reverse | .[].url),
           (if ._type == "playlist" then ($f.thumbnails // [] | reverse | .[].url) else empty end),
           $f.thumbnail,
           (if $f.id then "https://i.ytimg.com/vi/\($f.id)/hqdefault.jpg" else empty end)
         ] | map(select(. != null and . != "")) | join(" "))
      ] | join("\u001f")' <<<"$INFO")
  ARTIST=${ARTIST:-Unknown Artist}
  NAME=${NAME:-Unknown}
}

# cover (image or block art, see CFMT) in $TMP/cv.<uid> (needs chafa + curl; silent otherwise)
prep_cover() {
  local uid=$1 urls url ok fmt=symbols opt=--colors=none
  [[ $COVER == 0 ]] && return
  case $CFMT in kitty) fmt=kitty opt= ;; iterm) fmt=iterm opt= ;; sixel) fmt=sixels opt= ;; esac
  command -v chafa >/dev/null && command -v curl >/dev/null || return
  read -ra urls <<<"$2"
  (
    for url in "${urls[@]}"; do
      curl -fsSL --max-time 8 "$url" -o "$TMP/raw.$uid" 2>/dev/null && [[ -s $TMP/raw.$uid ]] && { ok=1; break; }
    done
    [[ $ok ]] || exit
    ffmpeg -v error -y -i "$TMP/raw.$uid" -frames:v 1 -vf "$CROP" "$TMP/img.$uid.jpg" 2>/dev/null ||
      cp "$TMP/raw.$uid" "$TMP/img.$uid.jpg"
    chafa -f "$fmt" $opt --size=24x12 --animate=off "$TMP/img.$uid.jpg" >"$TMP/cv.$uid.tmp" 2>/dev/null </dev/null &&
      mv "$TMP/cv.$uid.tmp" "$TMP/cv.$uid"
  ) &
}

# ---------- queue ----------
remove_item() {
  local i=$1 a
  for a in "${ITEM_ARRAYS[@]}"; do eval "$a=(\"\${$a[@]:0:$i}\" \"\${$a[@]:$((i + 1))}\")"; done
  (( i < SEL )) && ((SEL--))
  (( SEL >= ${#ST[@]} )) && SEL=$(( ${#ST[@]} - 1 ))
  (( SEL < 0 )) && SEL=0
  DIRTY=1
}

add_links() {
  local words w i a
  read -ra words <<<"${1//,/ }"
  (( ${#words[@]} )) || { note '!' "paste a youtube music link first"; return; }
  for w in "${words[@]}"; do
    if [[ ! $w =~ ^https?://((www|m|music)\.)?(youtube\.com|youtu\.be)/ ]]; then
      note '✗' "invalid link, try again"; continue
    fi
    [[ " ${LINKS[*]} " == *" $w "* ]] && continue
    i=${#LINKS[@]}; ((UIDN++))
    for a in "${ITEM_ARRAYS[@]}"; do eval "$a[i]="; done
    LINKS[i]=$w UIDS[i]=$UIDN ST[i]=check
    # a single track (even with a radio/mix id) must not pull the whole mix; album links stay playlists
    [[ ( $w == *"/watch"* || $w == *"youtu.be/"* ) && $w != *"list=OLAK"* ]] && NOPLS[i]=--no-playlist
    yt-dlp -J -q "${JS[@]}" ${NOPLS[i]} --playlist-items 1 "$w" >"$TMP/lk.$UIDN" 2>/dev/null &
    PIDS[i]=$!
    SEL=$i
    note '…' "checking link"
  done
  BUF[0]= POS[0]=0
}

finish_check() {
  local i=$1 uid=${UIDS[$1]}
  wait "${PIDS[i]}" 2>/dev/null
  if jq -e . "$TMP/lk.$uid" >/dev/null 2>&1; then
    INFO=$(<"$TMP/lk.$uid"); parse_info
    ISPL[i]=$IS_PL ARTISTS[i]=$ARTIST NAMES[i]=$NAME ST[i]=ready
    note '✓' "added: $ARTIST – $NAME"
    prep_cover "$uid" "$THUMB"
  else
    note '✗' "invalid link, try again  (${LINKS[i]})"
    remove_item "$i"
  fi
}

commit_dest() {
  local d=${BUF[1]}
  d=${d#"${d%%[![:space:]]*}"}; d=${d%"${d##*[![:space:]]}"}
  d=${d//\\ / }; d=${d#[\"\']}; d=${d%[\"\']}
  [[ $d == "~" || $d == "~/"* ]] && d=$HOME${d#\~}
  [[ -z $d ]] && d=$MUSIC_DIR
  if mkdir -p "$d" 2>/dev/null && [[ -w $d ]]; then
    MUSIC_DIR=$(cd "$d" && pwd) BUF[1]=$MUSIC_DIR POS[1]=${#MUSIC_DIR} NOTE= DIRTY=1
  else
    FOCUS=1; note '✗' "invalid folder, try again"; return 1
  fi
}

do_start() {
  local i n=0
  commit_dest || return
  for i in "${!ST[@]}"; do
    [[ ${ST[i]} == ready ]] || continue
    ST[i]=queued ITFMT[i]=${FORMATS[FI]} ITDEST[i]=$MUSIC_DIR; ((n++))
  done
  (( n )) || { note '!' "nothing to download - add a link first"; return; }
  BATCH=1; note '…' "downloading $n to $MUSIC_DIR"
}

do_remove() {
  (( ${#ST[@]} )) || return
  case ${ST[SEL]} in
    run)   note '!' "can't remove a running download"; return ;;
    check) kill "${PIDS[SEL]}" 2>/dev/null ;;
  esac
  remove_item "$SEL"
}

do_clear() {   # everything except downloads that are running right now
  local i n=0 kept=0
  for ((i = ${#ST[@]} - 1; i >= 0; i--)); do
    if [[ ${ST[i]} == run ]]; then ((kept++)); continue; fi
    [[ ${ST[i]} == check ]] && kill "${PIDS[i]}" 2>/dev/null
    remove_item "$i"; ((n++))
  done
  (( kept )) || BATCH=0
  if (( n == 0 && kept == 0 )); then note '!' "queue is already empty"
  elif (( kept )); then note '✓' "cleared $n, kept $kept running"
  else note '✓' "queue cleared"; fi
}

safe() { printf '%s' "$1" | sed -e 's#[/\\:*?"<>|]#_#g'; }

start_job() {
  local i=$1 base=${ITDEST[i]} folder fname out
  local args=(
    -x --audio-format "${ITFMT[i]}" --audio-quality 0
    --embed-metadata --no-mtime -N 4 -q --ignore-errors
    --print after_move:filepath "${JS[@]}" ${NOPLS[i]}
    --replace-in-metadata artist '(?i)^(.+?)(?:, \1)+$' '\1'   # "boris, boris" -> "boris"
    --convert-thumbnails jpg
    --ppa "ThumbnailsConvertor+FFmpeg_o:-c:v mjpeg -q:v 2 -vf \"$CROP\""   # square cover
  )
  case ${ITFMT[i]} in   # wav/aac can't hold embedded art -> save a jpg next to the file
    wav|aac) args+=(--write-thumbnail) ;;
    *)       args+=(--embed-thumbnail) ;;
  esac
  if [[ ${ISPL[i]} == true ]]; then
    folder="$base/$(safe "${ARTISTS[i]} - ${NAMES[i]}")"
    out="${folder//%/%%}/%(playlist_index)02d - %(title)s.%(ext)s"
    args+=(--parse-metadata "%(playlist_index)s:%(meta_track)s")
  else
    folder=$base; fname=$(safe "${ARTISTS[i]}")
    out="${base//%/%%}/${fname//%/%%} - %(title)s.%(ext)s"
  fi
  FOLDERS[i]=$folder
  yt-dlp "${args[@]}" -o "$out" "${LINKS[i]}" >"$TMP/done.${UIDS[i]}" 2>"$TMP/err.${UIDS[i]}" &
  PIDS[i]=$! ST[i]=run
}

poll() {
  local i rc msg running=0 ok=0 bad=0
  for ((i = ${#ST[@]} - 1; i >= 0; i--)); do
    [[ ${ST[i]} == check ]] && ! kill -0 "${PIDS[i]}" 2>/dev/null && finish_check "$i"
  done
  for i in "${!ST[@]}"; do
    [[ ${ST[i]} == run ]] || continue
    if kill -0 "${PIDS[i]}" 2>/dev/null; then ((running++)); continue; fi
    wait "${PIDS[i]}"; rc=$?; DIRTY=1
    if (( rc == 0 )) && [[ -s $TMP/done.${UIDS[i]} ]]; then
      ST[i]=ok
    else
      msg=$(grep -E '^ERROR' "$TMP/err.${UIDS[i]}" | tail -n 1 |
            sed -E 's/^ERROR: (\[[^]]*\] )?//; s/^https?:\/\/[^ ]+: //')
      ST[i]=err MSG[i]=${msg:-unknown error (exit code $rc)}
    fi
  done
  ACTIVE=0
  for i in "${!ST[@]}"; do
    if [[ ${ST[i]} == queued ]] && (( running < JOBS )); then start_job "$i"; ((running++)); DIRTY=1; fi
    case ${ST[i]} in check|queued|run) ACTIVE=1 ;; ok) ((ok++)) ;; err) ((bad++)) ;; esac
    # a cover finished in the background -> redraw once so it appears without a keypress
    [[ -z ${SHOWN[i]} && -s $TMP/cv.${UIDS[i]} ]] && { SHOWN[i]=1; DIRTY=1; }
  done
  if (( BATCH && ! ACTIVE )); then
    BATCH=0
    if (( bad )); then note '✗' "$ok done, $bad failed"; else note '✓' "done"; fi
  fi
}

# ---------- keyboard ----------
getkey() {
  local c c2 c3 c4 seq
  KEY= CH=
  IFS= read -rsn1 -t "$1" c || { (( $? > 128 )) && return 1; exit 0; }
  case $c in
    '')            KEY=ENTER ;;
    $'\t')         KEY=TAB ;;
    $'\177'|$'\b') KEY=BS ;;
    $'\001')       KEY=HOME ;;
    $'\005')       KEY=END ;;
    $'\025')       KEY=CLEAR ;;
    $'\e')
      IFS= read -rsn1 -t 0.01 c2 && [[ $c2 == '[' || $c2 == O ]] || { KEY=OTHER; return 0; }
      IFS= read -rsn1 -t 0.01 c3
      case $c3 in
        A) KEY=UP ;; B) KEY=DOWN ;; C) KEY=RIGHT ;; D) KEY=LEFT ;;
        H) KEY=HOME ;; F) KEY=END ;; Z) KEY=STAB ;;
        [0-9]) seq=$c3
               while IFS= read -rsn1 -t 0.01 c4 && [[ $c4 != [~A-Za-z] ]]; do seq+=$c4; done
               case $seq in 3) KEY=DEL ;; 1|7) KEY=HOME ;; 4|8) KEY=END ;; *) KEY=OTHER ;; esac ;;
        *) KEY=OTHER ;;
      esac ;;
    [[:cntrl:]])   KEY=OTHER ;;
    *)             KEY=CHAR CH=$c ;;
  esac
}

handle_key() {
  local f=$FOCUS v p n=${#FORMATS[@]} before=${#ST[@]}
  case $KEY in
    TAB)  FOCUS=$(( (f + 1) % 5 )); return ;;
    STAB) FOCUS=$(( (f + 4) % 5 )); return ;;
  esac

  if (( f < 2 )); then   # text fields: s/r/q are ordinary characters here
    v=${BUF[f]} p=${POS[f]}
    case $KEY in
      CHAR)  BUF[f]=${v:0:p}$CH${v:p} POS[f]=$((p + 1)) ;;
      BS)    (( p > 0 )) && { BUF[f]=${v:0:p-1}${v:p}; POS[f]=$((p - 1)); } ;;
      DEL)   BUF[f]=${v:0:p}${v:p+1} ;;
      LEFT)  (( p > 0 )) && POS[f]=$((p - 1)) ;;
      RIGHT) (( p < ${#v} )) && POS[f]=$((p + 1)) ;;
      HOME)  POS[f]=0 ;;
      END)   POS[f]=${#v} ;;
      CLEAR) BUF[f]= POS[f]=0 ;;
      UP)    (( f > 0 )) && FOCUS=$((f - 1)) ;;
      DOWN)  FOCUS=$((f + 1)) ;;
      ENTER) if (( f == 0 )); then
               add_links "$v"; (( ${#ST[@]} > before )) && FOCUS=3
             else commit_dest && FOCUS=2; fi ;;
    esac
    return
  fi

  case $KEY$CH in   # commands only fire on a lone keypress, never inside a paste
    CHARs) (( ISO )) && do_start; return ;;
    CHARr) (( ISO )) && do_remove; return ;;
    CHARc) (( ISO )) && do_clear; return ;;
    CHARq) (( ISO )) && exit 0; return ;;
  esac
  case $f:$KEY in
    2:LEFT)  FI=$(( (FI + n - 1) % n )) ;;
    2:RIGHT) FI=$(( (FI + 1) % n )) ;;
    2:UP)    FOCUS=1 ;;
    2:DOWN)  FOCUS=3 ;;
    2:ENTER) FOCUS=4 ;;
    3:UP)    if (( SEL > 0 )); then ((SEL--)); else FOCUS=2; fi ;;
    3:DOWN)  if (( SEL < before - 1 )); then ((SEL++)); else FOCUS=4; fi ;;
    4:UP)    FOCUS=3 ;;
    4:ENTER) do_start ;;
  esac
}

# ---------- drawing ----------
# put ROW COL STYLE TEXT MAXWIDTH -> appended to the frame buffer
put() { printf -v P '\033[%d;%dH%s%s%s' "$1" "$2" "$3" "${4:0:$5}" "$RST"; OUT+=$P; }

input_field() {   # ROW IDX FOCUSED WIDTH
  local v=${BUF[$2]} p=${POS[$2]} w=$4 s=0 vis pre ch post
  (( p >= w )) && s=$(( p - w + 1 ))
  vis=${v:s:w}
  if (( $3 )); then
    pre=${vis:0:p-s} ch=${vis:p-s:1} post=${vis:p-s+1}
    printf -v P '\033[%d;13H%s%s%s%s%s' "$1" "$pre" "$REV" "${ch:- }" "$RST" "$post"; OUT+=$P
  elif [[ -z $v && $2 == 0 ]]; then
    put "$1" 13 "$DIM" "paste youtube music link(s), enter to add" "$w"
  else
    put "$1" 13 "" "$vis" "$w"
  fi
}

render() {
  local r0=2 r i k n tag sym stat st avail col cv QR top rows=${#ST[@]} f
  local LW=$((W - 6)) PX=0 want= full=1 sig
  (( W >= 90 )) && { PX=$((W - 26)); LW=$((PX - 8)); }
  OUT=

  # real image: drawn once per selection/size change, and left alone by the per-frame clears
  if [[ $CFMT != text && $COVER != 0 ]] && (( PX && rows && H >= 18 )) &&
     [[ ${ST[SEL]} != check && -s $TMP/cv.${UIDS[SEL]} ]]; then
    want=${UIDS[SEL]} sig="$want:$W:$H"
    [[ $sig == "$IMG_SIG" ]] && full=0
  fi
  if (( full )); then
    for ((r = 1; r <= H; r++)); do printf -v P '\033[%d;1H\033[2K' "$r"; OUT+=$P; done
    [[ $CFMT == kitty ]] && OUT+=$'\033_Ga=d,d=A\033\\'
  else
    for ((r = 1; r <= H; r++)); do printf -v P '\033[%d;%dH\033[1K' "$r" $((PX - 1)); OUT+=$P; done
  fi

  # form
  f=(link folder format)
  for i in 0 1 2; do
    if (( FOCUS == i )); then put $((r0 + i)) 2 "$BOLD" "▸" 1; put $((r0 + i)) 4 "$BOLD" "${f[i]}" 8
    else put $((r0 + i)) 4 "$DIM" "${f[i]}" 8; fi
  done
  input_field $r0 0 $((FOCUS == 0)) $((LW - 10))
  input_field $((r0 + 1)) 1 $((FOCUS == 1)) $((LW - 10))
  col=13
  for i in "${!FORMATS[@]}"; do
    if (( i != FI )); then st=$DIM; elif (( FOCUS == 2 )); then st=$REV; else st=$BOLD$UL; fi
    put $((r0 + 2)) $col "$st" "${FORMATS[i]}" 6; col=$(( col + ${#FORMATS[i]} + 2 ))
  done
  [[ -n $NOTE ]] && put $((r0 + 4)) 3 "" "$NOTE_SYM $NOTE" $((LW - 2))

  # queue
  put $((r0 + 6)) 3 "$DIM" "queue $rows" 10
  QR=$(( H - r0 - 9 )); top=0; (( SEL >= QR )) && top=$(( SEL - QR + 1 ))
  for ((i = top; i < rows && i < top + QR; i++)); do
    r=$(( r0 + 7 + i - top )) tag="${ARTISTS[i]} – ${NAMES[i]}"
    case ${ST[i]} in
      check)  sym=${FR[TICK % 10]} stat=checking tag=${LINKS[i]} ;;
      ready)  sym='○' stat=ready ;;
      queued) sym='◌' stat=queued ;;
      run)    sym=${FR[TICK % 10]} stat=downloading ;;
      ok)     sym='✓' stat=done ;;
      err)    sym='✗' stat="failed: ${MSG[i]}"; stat=${stat:0:28} ;;
    esac
    st=; [[ ${ST[i]} == check || ${ST[i]} == queued ]] && st=$DIM
    (( i == SEL )) && { put $r 2 "$BOLD" "▸" 1; (( FOCUS == 3 )) && st=$BOLD; }
    avail=$(( LW - 6 - ${#stat} )); (( avail < 8 )) && avail=8
    put $r 4 "$st" "$sym" 1; put $r 6 "$st" "$tag" $avail
    put $r $(( 3 + LW - ${#stat} )) "$DIM" "$stat" ${#stat}
  done
  (( rows )) || put $((r0 + 7)) 4 "$DIM" "empty" 5

  # start button + hints
  n=0; for i in "${!ST[@]}"; do [[ ${ST[i]} == ready ]] && ((n++)); done
  if (( FOCUS == 4 )); then put $((H - 2)) 2 "$BOLD" "▸" 1; st=$REV; elif (( n )); then st=$BOLD; else st=$DIM; fi
  put $((H - 2)) 4 "$st" " start ($n) " 12
  if (( FOCUS < 2 )); then f="enter confirm · tab move · ctrl-c quit"
  else f="s start · r remove · c clear · q quit · tab move"; (( FOCUS == 2 )) && f+=" · ←/→ format"; fi
  put "$H" 3 "$DIM" "$f" $((W - 4))

  # cover panel (wide terminals)
  if (( PX && rows )) && [[ ${ST[SEL]} != check ]]; then
    i=$SEL
    put $r0 $PX "$BOLD" "${ARTISTS[i]}" 25
    put $((r0 + 1)) $PX "$DIM" "${NAMES[i]}" 25
    if [[ $CFMT == text && -s $TMP/cv.${UIDS[i]} ]]; then
      mapfile -t cv <"$TMP/cv.${UIDS[i]}"
      for ((k = 0; k < ${#cv[@]} && k < 12; k++)); do put $((r0 + 3 + k)) $PX "" "${cv[k]}" 25; done
    fi
    if [[ ${ST[i]} == ok ]]; then
      put $((r0 + 16)) $PX "$DIM" "saved to" 25
      put $((r0 + 17)) $PX "" "${FOLDERS[i]}" 25; (( r0 + 18 < H )) && put $((r0 + 18)) $PX "" "${FOLDERS[i]:25}" 25
    fi
  fi
  printf '%s' "$OUT"
  if [[ -z $want ]]; then IMG_SIG=
  elif (( full )); then
    printf '\033[%d;%dH' $((r0 + 3)) "$PX"; cat "$TMP/cv.$want"; IMG_SIG=$sig
  fi
}

# ---------- main loop ----------
printf '\033[?1049h\033[?25l'
stty -echo -icanon -ixon
while true; do
  if getkey 0.1; then
    ISO=1 handle_key; DIRTY=1
    while getkey 0.003; do ISO=0 handle_key; done
  fi
  poll
  (( ACTIVE )) && { ((TICK++)); DIRTY=1; }
  (( DIRTY )) && { render; DIRTY=0; }
done
