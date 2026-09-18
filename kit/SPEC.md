# 推しオフィス — 作り方の仕様書

この文書は、AI（Claude）が読んで**そのまま1つのアプリを作るための仕様**です。
読んだら、Artifact（claude.ai の中で動く1枚のWebページ）として作ってください。

参考の完成版: <https://yukitchy.github.io/oshi-office-ddaf2ce5/kit/source.html>
（1ファイルで動く実装です。丸ごと写して、名前と写真だけ差し替える作り方でも構いません）

---

## 1. 何を作るか

自分の「推し」4人を、自分の仕事のAI社員として担当に割り振り、
毎朝その推しから今日の報告を受け取り、そのまま相談できる**1画面のWebアプリ**です。

- HTML・CSS・JavaScript の **1ファイル**。外部のライブラリは使わない（Webフォントだけ読み込む）。
- サーバーもデータベースも使わない。設定と写真は `localStorage`（その端末の中）だけに保存する。
- スマートフォンで毎朝開くことが主。パソコンでも見られるようにする。

---

## 2. 画面仕様

### 2-1. スマートフォン（幅700px以下）

- 画面の下に**タブが5つ**。左から `部屋 / 報告 / 相談 / 推し活 / 設定`。
  - アイコンは線画のSVG（家・書類・吹き出し・ハート・歯車）を自分で書く。
  - 「相談」に未対応の要相談があるときは、タブの上に小さい丸い印を出す。
  - タブは `position:fixed` で下に貼り付け、`env(safe-area-inset-bottom)` を足す。
- **部屋タブ**がヒーロー。推しの**大きな写真カードを横スワイプ**で切り替える。
  - カード: 写真は `aspect-ratio:4/3`、`object-fit:cover`、`object-position:50% 22%`（顔が切れない位置）。
  - 写真の下端に白いグラデーションを重ね、その上に**名前（23px・900）と担当（11.5px・900・担当色）**。
  - 要相談があれば写真の右上にバッジ「要相談 N」。
  - 写真の下に**今日の報告3行**（見出し＋本文を26文字で切って「…」）。
  - いちばん下に**「◯◯に相談する」ボタン**（担当色・幅いっぱい）と、その右に 🔊 ボタン。
  - 横スワイプは `overflow-x:auto` ＋ `scroll-snap-type:x mandatory`、スクロールバーは隠す。
  - カードの下に**ドット表示**。いま見ているカードのドットだけ横長（幅22px）にして色を変える。
- **報告タブ**: 4人ぶんの報告を全文で縦に並べる。
- **相談タブ**: 要相談の印が付いた報告だけを集めて出す。
- **推し活タブ**: 上に「TODAY'S OSHI」の枠（今日の1本のメッセージ＋外部リンクのボタン）、下に推し活担当の報告と相談欄。
- **設定タブ**: 4人ぶんの「名前」「担当」の入力欄と「写真を選ぶ」。

### 2-2. パソコン（幅701px以上）

- 下タブと横スワイプは隠し、**2×2の部屋タイル**にする。
- 1タイル＝`grid-template-columns:200px 1fr`。左が写真（縦いっぱい・`object-position:top`）、右が名前・担当・報告3行・会話・相談の入力欄。
- 写真の右下に「大きく見る ↗」。押すと中央にモーダルが開き、報告の全文と会話が1枚で見える。
  - スマートフォンでは同じモーダルを**下から出るシート**にする。
- `Esc` キーと背景クリックで閉じる。

### 2-3. 配色トークン（`:root` にそのまま入れる）

```css
:root{
  --bg:#F7F5FB; --paper:#fff; --ink:#1E1A2E; --mute:#6B6483; --dim:#9C96AF;
  --line:#E4DFF0; --plum:#8E3D8F; --violet:#4B3A8E; --lav:#EFE9F8; --lav2:#DCD2F2;
  color-scheme:light;
}
```

- 推し1人ずつに**担当色** `--c` を持たせ、カード・ボタン・見出しに使う。
  初期値: `#4B3A8E` / `#B0783A` / `#8E3D8F` / `#2F6F8F`。
- 薄い背景は `color-mix(in srgb, var(--c) 22%, #fff)` のように担当色から作る。
- 明るい配色のみ。ダークモードは作らない（`color-scheme:light` で固定）。

### 2-4. 文字

```css
font-family:"Zen Kaku Gothic New","Hiragino Sans","Noto Sans JP",sans-serif;
font-size:15px; line-height:1.8; font-weight:500;
font-feature-settings:"palt"; word-break:normal; line-break:strict; overflow-wrap:anywhere;
```

- Google Fonts から `Zen Kaku Gothic New`（400,500,700,900）を読み込む。
- 見出しは 900、本文は 500。語の途中で改行しない（`word-break:normal`）。
- 角丸は大きめ（カード22〜24px、ボタン14px、バッジ999px）。

---

## 3. データ構造

推しは4人。配列 `OSHI` に持つ。

```js
const OSHI = [
  {
    id:"s1",                 // 変えない識別子
    name:"リク",             // 推しの名前（設定で書き換えられる）
    role:"宿泊担当",         // 担当（設定で書き換えられる）
    c:"#4B3A8E",             // 担当色
    img:"（画像のURLまたはdata URI）",
    greet:"今日の予約と問い合わせをまとめました。急ぎは1件です。",
    report:[                 // [見出し, 本文, "need"（相談したい時だけ）]
      ["今日の予約","撮影での利用が1件（13:00〜18:00）。前後の時間はまだ空いています。"],
      ["問い合わせ","予約サイト経由の2件は自動返信ずみ。自社サイトからの1件は日程の相談なので、返信案を作ってあります。","need"],
      ["泊まれるプランの準備","前の日の夕方から入れる案内文を下書きしました。近くの銭湯とコンビニの場所も入れています。"]
    ],
    sys:"スペースの予約・問い合わせ・宿泊プランの案内を担当。空き状況の確認、返信文の下書き、プランの説明を得意とする。"
  },
  // s2, s3, s4 も同じ形
];
```

初期値のサンプル（レンタルスペース運営の例。**自分の仕事に合わせて書き換える前提の仮の中身**）:

| id | name | role | 担当色 | 報告の中身 |
|---|---|---|---|---|
| s1 | リク | 宿泊担当 | `#4B3A8E` | 今日の予約 ／ 問い合わせ（要相談）／ 泊まれるプランの準備 |
| s2 | ハル | 経理担当 | `#B0783A` | 固定費（要相談）／ サイトの修正費 ／ 入金の確認 |
| s3 | ソラ | 推し活担当 | `#8E3D8F` | 今日の1本 ／ 大きい画面で見る ／ 次の予定 |
| s4 | ナギ | SNS営業担当 | `#2F6F8F` | 営業先の条件（要相談）／ これまでのお客さん ／ レビューのお願い |

ほかに2つ持つ。

```js
// 相談の時に推しが前提として読む、自分の仕事の説明。ここを自分のものに書き換える
const BUSINESS = "オーナーは、撮影などに使えるレンタルスペースを運営している。予約と問い合わせの対応、固定費の見直し、SNSでの営業、これまでの利用者への連絡を手作業でこなしていて、そこを軽くしたい。";

const TODAY = {
  booking:1, inquiry:3,                       // 画面上部に出す今日の数字
  oshiLink:"https://www.youtube.com/",        // 「今日の1本」を開くリンク
  oshiMsg:"今日の推しの1本を置いておきました。報告を読む前に、1本だけどうぞ。"
};
```

- 画面上部に「今日の予約 N件 ・ 問い合わせ N件 ・ **要相談 N件**」を出す。要相談の数は `report` の3番目が `"need"` のものを数える。
- 日付は `new Date().toLocaleDateString("ja-JP",{year:"numeric",month:"2-digit",day:"2-digit",weekday:"short"})`。

---

## 4. 機能仕様

### ① 設定で名前と担当を編集（localStorage保存）

- 設定タブに4行。1行＝担当色の丸・名前の入力欄・担当の入力欄・「写真を選ぶ」。
- `input` のたびに保存する。キーは `oshi-office-v1`、中身は `[{name,role}, ...]` の4件。
- 空にしたら初期値に戻す。読み込みは `try/catch` で囲み、壊れていたら初期値を使う。

```js
function load(){
  try{
    const s=JSON.parse(localStorage.getItem("oshi-office-v1"));
    if(Array.isArray(s)&&s.length===4)
      return OSHI.map((d,i)=>({...d, name:s[i].name||d.name, role:s[i].role||d.role}));
  }catch(e){}
  return OSHI.map(d=>({...d}));
}
```

### ② 推しの写真を自分で入れる（いちばん大事なところ）

- `<input type="file" accept="image/*">` で選ぶ。`<label>` で包んで「写真を選ぶ／写真を変える」と見せ、`input` 自体は視覚的に隠す（`position:absolute;width:1px;height:1px;opacity:0`）。`:focus-within` でふちを出し、キーボードでも選べるようにする。
- 選んだ画像は**そのまま持たない**。`canvas` で**長辺1024px**に縮めてから `toDataURL("image/jpeg", 0.82)` で dataURL にする。元のままだと1枚で保存容量を使い切る。

```js
function shrink(file){
  return new Promise((ok,ng)=>{
    const r=new FileReader(); r.onerror=ng;
    r.onload=()=>{ const im=new Image(); im.onerror=ng;
      im.onload=()=>{
        const k=Math.min(1,1024/Math.max(im.width,im.height)), c=document.createElement("canvas");
        c.width=Math.round(im.width*k); c.height=Math.round(im.height*k);
        c.getContext("2d").drawImage(im,0,0,c.width,c.height);
        ok(c.toDataURL("image/jpeg",0.82));
      }; im.src=r.result; };
    r.readAsDataURL(file);
  });
}
```

- 保存先は `localStorage` の `oshi-office-img`（`{s1:"data:image/jpeg;base64,...", ...}`）。
- **描いている場所すべてに反映する**（スワイプのカード・PCのタイル・モーダル）。`const imgSrc = s => imgs[s.id] || s.img;` を1か所だけ通す。
- **「選んだ写真を元に戻す」ボタン**を置く。押したら `oshi-office-img` を消して初期の画像に戻す。写真が1枚も入っていない時はボタンを隠す。
- 保存に失敗したら（容量オーバー）、画面に「この端末の保存容量がいっぱいで、写真を覚えておけませんでした。」と出す。読めない画像なら「この写真は読み込めませんでした。別の画像で試してください。」。
- 設定タブに「写真はこの端末の中だけに保存され、どこにも送られません。」と**必ず書く**。

### ③ 相談

- 各所に相談の入力欄（丸い入力欄＋「送る」）を置く。送信中はボタンを `disabled`。
- 会話は画面の中の変数（`history[推しのid]`）に持つ。直近4件を吹き出しで出す。自分の発言は右寄せ・薄紫、推しの発言は左寄せ・担当色の縦線。
- 返事は **Claude の中で動いていれば `window.claude.use("sample")` を使う**。使えなければ定型の返事に落とす。

```js
let sampleNs;
async function getSample(){
  if(sampleNs!==undefined) return sampleNs;
  try{ sampleNs = window.claude?.use ? await window.claude.use("sample") : null; }
  catch(e){ sampleNs=null; }
  return sampleNs;
}
async function askOshi(s, history, onText){
  const ns = await getSample();
  if(!ns) return canned(s);                    // 定型の返事
  const turns = [{role:"user",content:sys+"\n\n（以下、会話）"},
                 {role:"assistant",content:"はい、"+s.name+"です。"}].concat(history);
  try{
    const r = await ns(turns,{onText:u=>onText&&onText(u.text), modelTier:"quick", cache:false});
    return r.text;
  }catch(e){ return canned(s); }
}
```

- システムプロンプトの型（`sys`）:

```
あなたはAI社員「{名前}」（{担当}）。実在の人物本人ではなく、持ち主が自分の推しをモデルに設定した「もどき」です。
本人のふりや本人の発言の捏造はしない。口調は落ち着いて優しく、仕事はプロとして具体的に。{その推しの sys}
事業の前提: {BUSINESS}
今日の報告: {見出し＝本文 を「／」でつないだもの}
ルール: 日本語。200字以内。分からない数字は作らず「明細を見せてください」と頼む。最後に一言だけねぎらう。
```

- 定型に落ちている時は、相談欄の下に小さく「この画面では定型の返事だけです。claude.ai で開くと、推しが本当に答えます。」と出す。Claude の中で動いている時はこの文を出さない。

### ④ 🔊 読み上げ

- 報告・推しの返事・カードの横に 🔊 ボタン（背景なし・薄く、押せる大きさを確保）。
- 端末に入っている `speechSynthesis` だけを使う。**実在の人物の声は使わない・作らない**。

```js
const TTS = typeof speechSynthesis !== "undefined";
const PITCH = {s1:0.85, s2:0.90, s3:0.95, s4:1.0};   // 推しごとに声の高さを変える
```

- 声は `ja-JP` の声を選ぶ（`getVoices()` を `/^ja(-|_)?JP/i` で絞る）。`voiceschanged` でも選び直す。
- 読む文は「{名前}です。{本文} 無理しないでくださいね。」。`rate:0.95`、`pitch` は上の表。
- **同じボタンをもう一度押したら止める**（`speechSynthesis.cancel()`）。別のものを押したら前のを止めて新しいものを読む。
- **対応していない端末ではボタンを `disabled`** にし、`title="この端末は読み上げ非対応"` を付ける。エラーは出さない。

### ⑤ ホーム画面に追加できること

- `<title>推しオフィス</title>` と次のメタタグを入れる。

```html
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta name="theme-color" content="#F7F5FB">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="推しオフィス">
```

- 本文の下に `padding-bottom:calc(66px + env(safe-area-inset-bottom))` を入れ、下タブに隠れないようにする。

---

## 5. 守ること

- **実在のアイドル・芸能人の写真や声を、アプリに入れて配らない。似せた画像も作らない・作らせない。**
- 最初から入っている4人の画像は、**AIで描いた架空の人物**です。名前も仮のものです。画面の中に「最初から入っている4人は、AIで描いた架空の人物です」と書いておく。
- 本物の推しは、**持ち主が自分の端末で自分で入れる**。その写真はその端末の `localStorage` の中だけに残り、どこにも送らない（外部へのアップロードは一切書かない）。
- 推しに本人のふりをさせない。相談の答えの中で、本人が言っていない発言を作らない。
- 画面に出す数字（予約件数・金額など）は仮のもの。分からない数字を作って断定しない。

---

## 6. 受け入れ基準（作ったら自分で1つずつ確かめる）

1. 幅390pxで開いて、**横スクロールが出ない**（`document.documentElement.scrollWidth === clientWidth`）。
2. 下タブ5つがすべて切り替わり、それぞれ中身が表示される。
3. 部屋タブでカードを横にスワイプすると次の推しに移り、**ドットの位置が連動する**。
4. 設定で名前と担当を書き換えると、他のタブの表示にも即座に反映される。
5. **写真を入れてページを再読み込みしても、その写真が残っている**。
6. 入れた写真が、スワイプのカード・PCのタイル・モーダルの**すべてに反映されている**。
7. 「選んだ写真を元に戻す」を押すと、初期の画像に戻る。
8. 🔊 を押すと読み上げが始まり、もう一度押すと止まる。**読み上げに対応していない端末でもエラーにならず、ボタンが押せない状態になる**だけ。
9. 相談を送ると返事が返る。Claude の外で開いた時は定型の返事になり、その旨の一文が出る。**どちらの場合も画面が壊れない**。
10. 幅1280pxで開くと2×2のタイルになり、「大きく見る ↗」でモーダルが開き、`Esc` と背景クリックで閉じる。
11. コンソールにエラーが1つも出ていない。
12. 画面のどこかに「最初から入っている4人はAIで描いた架空の人物」「写真は端末の中だけに保存される」の2つが書いてある。
