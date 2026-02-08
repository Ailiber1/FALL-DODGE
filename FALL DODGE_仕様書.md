FALL DODGE
Game Specification Document
(Platform-Neutral / AI Build Ready / Firebase + Single HTML)
v4 — COMPLETE EDITION


================================
0. 目的 / 対応環境 / 技術スタック
================================
本仕様書は以下の環境でそのまま利用可能とする：
・Claude Code
・Google AI Studio（Build / Apps）
・WebベースAI開発環境全般

技術スタック：
・フロントエンド: 単一HTMLファイル（HTML/CSS/JavaScript 全内包）
・バックエンド: Firebase Realtime Database
・ホスティング: GitHub Pages
・認証: デバイスID方式（localStorage）
・描画: Canvas API（ゲーム部分）

プロジェクト構成：
```
FALL-DODGE/
├── index.html      # 単一ファイルアプリ
└── README.md
```

外部CDN（使用許可）：
・Firebase SDK: https://www.gstatic.com/firebasejs/...


================================
1. ゲーム概要
================================
タイトル：FALL DODGE
ジャンル：リアルタイム回避アクション

・上空から日用品・工具などの3D落下物が降ってくる
・地上の棒人間キャラクターを操作して回避
・落下物を避け続けるゲーム
・回避した回数がスコアになる

プレイ形式：
・一人でプレイ（ソロ）
・マルチプレイ（同画面・リアルタイム同期・最大4人）

対応デバイス：
・PC（Web）最優先
・スマートフォン（Web / 9:16 縦持ち / 16:9 横持ち）

デザインテーマ：
Cyber Terminal / Neon Green / HUD / Matrix Style
※LINKED BLOCKS_ と同一のデザイン体系を継承

キャラクター：
・棒人間（スティックフィギュア）
・頭部: 3Dスフィア（ハイライト＋グラデーション）
・手足: 光沢ライン付きリム
・地面に影を描画

プレイヤーカラー（最大4人）：
・P1: ピンク  (255, 50, 150)
・P2: シアン  (0, 220, 255)
・P3: イエロー (255, 220, 0)
・P4: パープル (180, 80, 255)


================================
2. 起動キャッチコピー（毎回表示）
================================
表示文（固定）：
「一瞬の判断が、生存を決める」

仕様：
・アプリ起動時、毎回必ず表示
・ネオングリーン発光テキスト
・2〜3秒で自動フェードアウト
・クリック／タップで即消去可能
・世界観演出として扱う

同期SE：
・SE_MATCH_READY（インパクト音）


================================
3. UI STATE（共通ステート）
================================
UI_STATE:
- IDLE      : 通常
- HOVER     : ホバー中
- ACTIVE    : 選択中
- DISABLED  : 無効
- ERROR     : エラー

視覚ルール：
・HOVER → 緑発光 + 浮き上がり（translateY: -2px）
・ACTIVE → 常時発光 + 微パルス（pulse animation）
・DISABLED → 低コントラスト（opacity: 0.4）
・ERROR → 短時間赤発光 + 警告SE

CSS変数（LINKED BLOCKS_ 共通）：
```css
:root {
    --neon-green: #00ff41;
    --neon-green-dim: #00aa2a;
    --neon-green-glow: rgba(0, 255, 65, 0.6);
    --neon-cyan: #00ffff;
    --neon-pink: #ff00ff;
    --neon-yellow: #ffff00;
    --neon-red: #ff3366;
    --neon-gold: #ffcc00;
    --neon-purple: #b450ff;
    --bg-dark: #000a04;
    --bg-darker: #050505;
    --bg-panel: rgba(0, 20, 10, 0.85);
    --text-primary: #c0ffc0;
    --text-dim: #4a8a4a;
    --border-glow: 0 0 8px var(--neon-green), inset 0 0 8px rgba(0, 255, 65, 0.1);
}
```


================================
4. タイトル画面構成
================================
ヘッダー帯（上部）：
・「SYSTEM ACCESS // FALL DODGE // CONNECTION: SECURE」
・モノスペースフォント、dim green、小さめ

タイトルロゴ：
・「FALL DODGE」ネオングリーン発光
・text-shadow多重（3層グロー）
・ロゴ下に4色の棒人間イラストが並ぶ

中央カラム（上から縦並び）：
1) USER NAME 入力欄
2) 登録してゲームを始める
3) 登録しないでゲームを始める（ゲスト）
4) 難易度選択: [簡単] [普通] [難しい] ← 3択トグル（デフォルト: 普通）
5) 一人でプレイ
6) マルチプレイ（最大4人）

補助ボタン（下部）：
・ルール説明を見る
・ランキングを見る

フッター帯：
・BGM ON/OFF ボタン
・SE ON/OFF ボタン


================================
5. ユーザーネーム仕様
================================
【登録プレイ】
・ユーザーネーム永続保存（Firebase + localStorage）
・ランキング対象
・同名重複不可
・再ログイン可（deviceIdで照合）

重複時表示：
「このネームは他のユーザーが使用中です」
→ ERROR state + 警告SE

【ゲスト】
・登録不要
・表示名：PLAYER 1 / PLAYER 2 / PLAYER 3 / PLAYER 4
・ランキング対象外

デバイスID生成：
```javascript
function getDeviceId() {
    let id = localStorage.getItem('fall_dodge_device_id');
    if (!id) {
        id = 'dev_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9);
        localStorage.setItem('fall_dodge_device_id', id);
    }
    return id;
}
```


================================
6. ルールブック
================================
・モーダル表示、背景暗転
・スマホ縦スクロール対応
・ネオン枠モーダル

内容：
・落下物（日用品・工具）が上から降ってくる
・左右移動＋ダッシュで回避
・落下物に当たったらゲームオーバー（シールド所持時は1回無効）
・回避成功1回につきスコア+1、ニアミスでボーナス
・難易度は3段階（簡単/普通/難しい）
・時間経過で難易度が徐々に上昇
・ステージ制: スコアに応じて背景と落下物テーマが変化
・パワーアップアイテムが混じって落下（取ると一時効果）
・マルチプレイ: 最大4人、最後まで生き残った人が勝利
・ランキングは登録ユーザーのみ対象
・ゲームモード: ノーマル / サバイバル / タイムアタック / チーム戦 / デイリーチャレンジ / ボスラッシュ / リバース


================================
7. ランキング仕様
================================
表示：
・トップ50表示
・モーダル表示

項目：
・順位 / ユーザーネーム / スコア / 難易度 / ゲームモード

ランキング対象：
・登録ユーザーのみ
・ソロ・マルチ両方のスコアを記録

Firebase取得：
```javascript
database.ref('rankings')
    .orderByChild('score')
    .limitToLast(50)
    .on('value', snap => { /* 降順で表示 */ });
```


================================
8. 操作仕様
================================
【PC】
・← キー長押し：左移動
・→ キー長押し：右移動
・Shift / Space ダブルタップ：ダッシュ（短距離、無敵フレーム付き）
・キーを離す：停止

```javascript
document.addEventListener('keydown', (e) => {
    if (e.key === 'ArrowLeft') GameState.moveLeft = true;
    if (e.key === 'ArrowRight') GameState.moveRight = true;
    if (e.key === ' ' || e.key === 'Shift') GameState.dashRequested = true;
});
document.addEventListener('keyup', (e) => {
    if (e.key === 'ArrowLeft') GameState.moveLeft = false;
    if (e.key === 'ArrowRight') GameState.moveRight = false;
});
```

【スマートフォン】
・画面最下部にネオン枠の [← 左] [右 →] ボタンを配置
・左右ボタンをダブルタップでダッシュ発動
・ゲームCanvas外に配置（重ならないこと）
・touchstart → 移動開始 / touchend → 停止
・e.preventDefault() で誤スクロール防止


================================
9. ゲームルール・ゲームループ
================================
■ コアルール
・画面上空から落下物がランダムX座標に出現
・キャラクターは画面下部に配置、左右移動＋ダッシュ
・落下物に接触 → 即ゲームオーバー（シールド所持時は消費して無効化）
・画面下端を通過した落下物 → スコア+1
・ニアミス（接触ギリギリで回避） → ボーナスポイント

■ ゲームループ（requestAnimationFrame / 60fps）
```
1. 入力状態チェック（moveLeft / moveRight / dash）
2. ダッシュ判定
3. プレイヤー位置更新（移動速度 = 5px/frame）
4. 落下物の位置更新（y += currentSpeed、回転角度更新）
5. バウンド判定（地面衝突時）
6. 当たり判定（矩形衝突 + シールドチェック）
7. ニアミス判定（近接距離チェック）
8. パワーアップアイテム接触判定
9. 画面外落下物の除去 → スコア+1
10. 新規落下物の生成（interval判定 + ステージテーマ反映）
11. 巨大落下物の予告影＋生成
12. 天候エフェクト更新
13. 難易度パラメータの時間経過更新
14. ステージ遷移判定（スコア閾値チェック）
15. コンボ倍率更新
16. Canvas描画（3D風アイテム + 棒人間 + エフェクト）
```

■ 当たり判定
```javascript
function checkCollision(player, obj) {
    // シールド所持時は被弾無効 + シールド消費
    if (player.shield) { player.shield = false; return false; }
    // ダッシュ無敵フレーム中は無効
    if (player.dashInvincible) return false;
    return player.x < obj.x + obj.width &&
           player.x + player.width > obj.x &&
           player.y < obj.y + obj.height &&
           player.y + player.height > obj.y;
}
```

■ ニアミス判定
```javascript
function checkNearMiss(player, obj) {
    const margin = 8; // px
    const expanded = {
        x: obj.x - margin, y: obj.y - margin,
        width: obj.width + margin * 2, height: obj.height + margin * 2
    };
    return !checkCollision(player, obj) &&
           player.x < expanded.x + expanded.width &&
           player.x + player.width > expanded.x &&
           player.y < expanded.y + expanded.height &&
           player.y + player.height > expanded.y;
}
```


================================
10. 難易度仕様
================================
開始前にタイトル画面で選択（3択トグル）：
・簡単
・普通（デフォルト）
・難しい

■ パラメータテーブル

| パラメータ       | 簡単    | 普通    | 難しい   |
|-----------------|---------|---------|---------|
| 初期落下速度     | 2       | 3       | 5       |
| 初期生成間隔(ms) | 1200    | 800     | 500     |
| 同時最大落下数   | 3       | 5       | 8       |
| 速度上昇率(/10秒)| +0.3    | +0.5    | +0.8    |
| 間隔減少率(/10秒)| -50ms   | -80ms   | -100ms  |
| 上昇停止時間     | 90秒後  | 60秒後  | 45秒後  |
| 最大落下速度     | 5       | 8       | 12      |
| 最小生成間隔(ms) | 600     | 300     | 150     |

■ 共通仕様
・時間経過で10秒ごとにパラメータ更新
・上昇停止時間に到達後は固定難易度（サバイバルモード除く）
・シングル／マルチ共通


================================
11. 落下物仕様
================================
■ 落下物カテゴリ（日用品・工具）
以下の10種をランダム生成。すべて3D風描画。

| ID  | アイテム    | 描画色         | サイズ目安 | 特徴              |
|-----|-----------|----------------|----------|-------------------|
| 1   | ハンマー   | オレンジ        | 28x40px  | T字型、金属光沢     |
| 2   | レンチ    | シルバー        | 16x44px  | 顎部+シャフト      |
| 3   | バケツ    | ブルー          | 20x32px  | 台形+楕円リム      |
| 4   | ボトル    | グリーン        | 14x40px  | シリンダー+キャップ  |
| 5   | 靴       | ブラウン        | 24x32px  | L字シルエット      |
| 6   | スマホ    | ダークグレー     | 14x28px  | 角丸+画面反射      |
| 7   | コーヒーカップ | ベージュ    | 16x24px  | 円筒+取っ手+湯気    |
| 8   | ハサミ    | クローム        | 20x40px  | X字+リングハンドル  |
| 9   | 本       | レッド          | 20x28px  | 厚み+ページ断面     |
| 10  | 鍵       | ゴールド        | 12x44px  | 球体頭部+歯       |

■ 3D描画共通仕様
・面ごとのグラデーション（上面=明 / 右面=中 / 底面=暗）
・ハイライト反射（左上に白〜明色の楕円）
・ドロップシャドウ（右下にオフセット暗色）
・落下中に回転アニメーション（rotationAngle += rotationSpeed）

■ 落下物の物理
・回転: 各アイテムに rotationSpeed をランダム付与（-3〜+3 deg/frame）
・バウンド: 地面衝突時に跳ね返り（最大1回、y方向に速度の30%で反射）
  → バウンド中も当たり判定あり（二次被害）
  → バウンド後画面外に消えた時点でスコア+1

■ 巨大ボス落下物
・一定間隔（30秒ごと）で出現
・種類: ピアノ、冷蔵庫、タンス
・サイズ: 画面幅の1/3
・2秒前に予告影（赤い半透明シルエット）を落下地点に表示
・落下速度は通常の70%（遅いが避けにくい）
・通過時スコア+5


================================
12. ステージ制
================================
■ テーマステージ
スコアに応じて背景・BGM・落下物テーマが自動遷移する。

| ステージ     | 遷移スコア | 背景テーマ   | 落下物テーマ            | BGM変化  |
|------------|----------|------------|----------------------|---------|
| STAGE 1    | 0〜      | 工事現場     | ハンマー,レンチ,バケツ    | テンポ低  |
| STAGE 2    | 50〜     | キッチン     | カップ,ボトル,鍵,ハサミ   | テンポ中  |
| STAGE 3    | 120〜    | オフィス     | スマホ,本,鍵            | テンポ高  |
| STAGE 4    | 200〜    | 夜の街      | 靴,全アイテム混合        | テンポ高+ |
| STAGE 5    | 300〜    | 宇宙       | 全アイテム+巨大落下物増加  | テンポ最高 |

■ 遷移演出
・スコア到達時に「STAGE X」テキストが画面中央に2秒表示
・ネオン発光+フラッシュ演出
・背景グリッドの色がステージごとに変化:
  - STAGE 1: グリーン基調
  - STAGE 2: シアン基調
  - STAGE 3: イエロー基調
  - STAGE 4: ピンク基調
  - STAGE 5: パープル基調
・マトリックス背景の文字色もステージに連動

■ 背景描画
```javascript
const STAGE_CONFIG = {
    1: { gridColor: '#001a0a', accentColor: '#00ff41', label: 'CONSTRUCTION SITE' },
    2: { gridColor: '#001a1a', accentColor: '#00ffff', label: 'KITCHEN' },
    3: { gridColor: '#1a1a00', accentColor: '#ffff00', label: 'OFFICE' },
    4: { gridColor: '#1a000a', accentColor: '#ff00ff', label: 'NIGHT CITY' },
    5: { gridColor: '#0a001a', accentColor: '#b450ff', label: 'OUTER SPACE' },
};

function getCurrentStage(score) {
    if (score >= 300) return 5;
    if (score >= 200) return 4;
    if (score >= 120) return 3;
    if (score >= 50) return 2;
    return 1;
}
```


================================
13. 天候エフェクト
================================
難易度とステージに連動して天候が変化する。

| 天候     | 発生条件          | 視覚効果                  | ゲーム効果           |
|---------|-----------------|--------------------------|---------------------|
| 晴れ     | デフォルト        | なし                     | 通常                |
| 雨      | STAGE 2以降ランダム| 斜めの線パーティクル、画面やや暗い | 視認性やや低下         |
| 雷      | STAGE 3以降ランダム| 一瞬の白フラッシュ（0.1秒）  | フラッシュ中に視界喪失  |
| 嵐      | STAGE 4以降ランダム| 強い斜め線+画面揺れ         | 落下物が斜めに飛ぶ     |

■ 天候変化サイクル
・15〜30秒間隔でランダム変化
・天候変化時に画面上部に天候アイコン+名称を1秒表示
・嵐時の落下物角度: x方向に ±2px/frame の横移動を付与

```javascript
const WeatherSystem = {
    current: 'clear',  // 'clear' | 'rain' | 'thunder' | 'storm'
    timer: 0,
    nextChangeAt: 0,

    update(elapsed, stage) {
        if (elapsed < this.nextChangeAt) return;
        const weathers = ['clear'];
        if (stage >= 2) weathers.push('rain');
        if (stage >= 3) weathers.push('thunder');
        if (stage >= 4) weathers.push('storm');
        this.current = weathers[Math.floor(Math.random() * weathers.length)];
        this.nextChangeAt = elapsed + 15000 + Math.random() * 15000;
    },

    applyToObject(obj) {
        if (this.current === 'storm') {
            obj.x += (Math.random() > 0.5 ? 2 : -2);
        }
    }
};
```


================================
14. パワーアップアイテム
================================
金色の星形アイテムが低確率で落下物に混じる。
接触で取得。一時効果が発動。

| アイテム         | アイコン | 効果                    | 持続時間 | 出現率  |
|-----------------|--------|------------------------|---------|--------|
| シールド         | 🛡️     | 被弾1回無効              | 1回使い切り | 8%    |
| スピードアップ    | ⚡      | 移動速度2倍              | 5秒     | 10%    |
| スローモーション  | 🔽      | 落下物速度50%減          | 3秒     | 7%     |
| 縮小            | 🔲      | 当たり判定サイズ50%       | 5秒     | 8%     |

■ 表示仕様
・金色発光の星形（3D風：光沢+影）
・通常落下物より少し遅い落下速度（70%）
・取得時に専用SE再生 + 画面上部にアイコン＋残り秒数をHUD表示
・効果終了時にフェードアウト演出

■ 妨害アイテム（マルチ専用）
・赤い骸骨マークのアイテム（出現率5%）
・取得すると他の全プレイヤーの画面に追加落下物3個を即時追加
・ぷよぷよ式の「おじゃま」演出

```javascript
const PowerUpTypes = {
    shield:   { icon: '🛡️', duration: -1,    color: '#4488ff' },
    speed:    { icon: '⚡',  duration: 5000,  color: '#ffcc00' },
    slow:     { icon: '🔽',  duration: 3000,  color: '#00ff88' },
    shrink:   { icon: '🔲',  duration: 5000,  color: '#88aaff' },
    sabotage: { icon: '💀',  duration: 0,     color: '#ff3333', multiOnly: true },
};
```


================================
15. ダッシュ（回避アクション）
================================
■ 基本仕様
・入力: Shift / Space / 方向ダブルタップ
・効果: 移動方向に高速短距離移動（120px）
・無敵フレーム: ダッシュ中の0.2秒間は当たり判定無効
・クールダウン: 3秒

■ 視覚演出
・ダッシュ中に残像エフェクト（3フレーム分の半透明コピー）
・移動軌跡にネオン発光ライン
・クールダウン中はHUDにゲージ表示（徐々に回復）

```javascript
const DashSystem = {
    cooldown: 3000,
    lastDashTime: 0,
    dashDistance: 120,
    dashDuration: 200,  // ms
    isActive: false,

    canDash() {
        return Date.now() - this.lastDashTime >= this.cooldown;
    },
    execute(direction) {
        if (!this.canDash()) return;
        this.isActive = true;
        this.lastDashTime = Date.now();
        GameState.dashInvincible = true;
        // 120px移動を200msで実行
        setTimeout(() => {
            this.isActive = false;
            GameState.dashInvincible = false;
        }, this.dashDuration);
    }
};
```


================================
17. コンボ・スコアシステム
================================
■ コンボ倍率
・連続回避でコンボカウンター上昇
・被弾（シールド消費含む）でリセット

| 連続回避数 | コンボ倍率 | 表示      |
|----------|----------|----------|
| 0〜9     | x1       | なし      |
| 10〜24   | x2       | "x2 COMBO!" |
| 25〜49   | x3       | "x3 COMBO!" |
| 50〜99   | x5       | "x5 FEVER!" |
| 100〜    | x10      | "x10 MAX!!" |

■ ニアミスボーナス
・落下物と接触ギリギリ（8px以内）で回避
・通常スコアに加え +3 ボーナス（コンボ倍率適用前）
・「NEAR MISS!」テキスト表示（0.5秒フェード）
・専用SE再生

■ スコア計算
```javascript
function addScore(isNearMiss) {
    const base = 1 + (isNearMiss ? 3 : 0);
    const multiplied = base * GameState.comboMultiplier;
    GameState.score += multiplied;
    GameState.comboCount++;
    updateComboMultiplier();
}
```

■ HUD表示
・スコア: 画面左上、常時表示
・コンボ: 画面右上、x2以上で表示（パルスアニメーション）
・ニアミス: 回避地点付近にフロートテキスト


================================
18. スコア／リザルト
================================
■ スコア
・ゲーム中: 画面上部HUDに常時表示
・ネオングリーン数字、モノスペース

■ リザルト画面（#result-screen）

【ソロ】
```
GAME OVER
SCORE: XXX
DIFFICULTY: NORMAL
STAGE: 3 / OFFICE
MAX COMBO: x5 (52)
[ランキング登録完了]    ← 登録ユーザーのみ
[もう一度プレイ]
[タイトルに戻る]
```

【マルチ: 勝利】
```
GAME SET!
WINNER: ユーザーネーム（勝利棒人間が万歳ポーズ）
SCORE: XXX

1st  P1 (YOU)      078
2nd  P2 (GUEST 1)  065
3rd  P3 (GUEST 2)  051
4th  P4 (GUEST 3)  042

[再戦]  [タイトルに戻る]
```


================================
19. ゲームモード
================================
タイトル画面のモード選択でプレイモードを切り替え。

■ ノーマル（デフォルト）
・前述の基本ルール通り
・難易度上昇は上昇停止時間で固定
・ソロ / マルチ両対応

■ サバイバル
・難易度上昇が永久に止まらない
・パラメータ上限なし（速度・間隔が際限なく上昇）
・世界ランキング専用モード（別枠ランキング）
・ソロ専用

■ タイムアタック
・制限時間: 60秒固定
・60秒以内の最高スコアを競う
・被弾してもゲームオーバーにならず、3秒間スコア加算停止のペナルティ
・ソロ / マルチ両対応

■ チーム戦（2v2）
・4人を2チームに分割（P1+P3 vs P2+P4）
・チームメンバーが全滅したら負け
・最後まで1人でも生存しているチームが勝ち
・味方の画面にチームカラーで共有表示
・マルチ4人専用

■ デイリーチャレンジ
・毎日シード固定の落下パターン
・同じ条件で全プレイヤーがスコアを競う
・日替わりランキング（Firebase: dailyRankings/<date>）
・ソロ専用

■ ボスラッシュ
・巨大落下物だけが次々降ってくる特殊モード
・予告影→落下のパターンを読んで回避
・10体クリアで勝利
・ソロ / マルチ両対応

■ リバース
・プレイヤーが画面上部、落下物が下から上昇
・操作感覚の逆転（左右は同じ、上下の認知が反転）
・ソロ / マルチ両対応

```javascript
const GAME_MODES = {
    normal:    { label: 'ノーマル',       solo: true, multi: true,  maxPlayers: 4 },
    survival:  { label: 'サバイバル',     solo: true, multi: false, maxPlayers: 1 },
    timeAttack:{ label: 'タイムアタック',  solo: true, multi: true,  maxPlayers: 4 },
    teamBattle:{ label: 'チーム戦 2v2',   solo: false, multi: true, maxPlayers: 4, minPlayers: 4 },
    daily:     { label: 'デイリーチャレンジ', solo: true, multi: false, maxPlayers: 1 },
    bossRush:  { label: 'ボスラッシュ',    solo: true, multi: true,  maxPlayers: 4 },
    reverse:   { label: 'リバース',       solo: true, multi: true,  maxPlayers: 4 },
};
```


================================
20. マルチプレイ仕様
================================
■ ルーム管理
・ルームID: 6文字ランダム英数字（自動生成）
・最大4人
・1ID = 1ルーム

■ ロビー画面（#lobby-screen）
```
ONLINE ROOM // ルームID: XXXXXX
[ルームを作る]          ← ルーム作成 → 待機
[ルームIDで参加]        ← ID入力 → join
[モード選択]            ← ホストのみ
[難易度選択]            ← ホストのみ

待機中表示:
  P1: YOU          ← ピンク枠
  P2: GUEST 1      ← シアン枠
  P3: 待機中...     ← dim表示
  P4: 待機中...     ← dim表示

[ゲーム開始]  ← ホストのみ、2人以上で有効化（チーム戦は4人必須）
```

■ IDロック条件
・待機中 / 対戦中 / 再戦中
・解除: 全員が「タイトルに戻る」を選択しルーム人数0

■ 同期方式
・ホスト（ルーム作成者）: 落下物の生成・位置計算を担当
・ホストがFirebaseに fallingObjects を書き込み
・ゲスト: fallingObjects をリアルタイム監視して描画
・各プレイヤーは自分の x座標 をFirebaseに書き込み
・当たり判定は各クライアントがローカル実行 → 被弾時 alive: false に更新

■ 位置同期throttle: 50ms（負荷軽減）

■ 勝敗判定
・最後まで alive: true のプレイヤーが勝利
・残り1人になった時点でGAME SET
・全員同フレーム被弾（タイムスタンプ差100ms以内） → ドロー
・チーム戦: チーム単位で全滅判定

■ 妨害アイテム同期
・妨害アイテム取得時、ホストが他プレイヤーのフィールドに追加落下物をFirebase経由で配信


================================
21. マルチ対戦：退席（強制離脱）仕様
================================
■ 退席検知
発生条件：
・アプリ完全離脱
・通信切断
・ブラウザ／タブ終了

■ 退席通知表示
・退席プレイヤー名をネオンレッドで強調表示
・「〇〇さんが退席しました」
・画面中央寄り、約2秒で自動フェードアウト
・警告系SE再生

■ 退席時の処理
・残りプレイヤーが1人 → 自動勝利
・残りプレイヤーが2人以上 → ゲーム続行（退席者は除外）

表示（自動勝利時）：
```
GAME SET!
WINNER: 残ったプレイヤー名
（補足）相手が退席しました
```

■ IDロックとの関係
・ルーム存続中は常にIDロック維持
・全員退出で初めてID解放


================================
22. 落下物衝突演出
================================
■ 地面衝突（スコア加算時）
・衝突地点にパーティクル破片（8〜12個）
・アイテムの色に合わせた破片色
・0.3秒で拡散＋フェードアウト

■ 被弾演出
・衝突地点にネオンレッド発光エフェクト
・爆発パーティクル（赤〜オレンジ系、16個）
・画面全体に赤フラッシュ（0.1秒）
・画面振動（3フレーム、±3px）
・被弾SE再生

■ 巨大落下物の衝突
・通常の3倍のパーティクル量
・画面振動が強い（5フレーム、±6px）
・衝撃波エフェクト（円形に広がるリング）

■ シールド消費時
・衝突地点にブルーのバリア破裂エフェクト
・「SHIELD BREAK!」テキスト表示
・シールドSE再生


================================
23. ゲーム画面HUD構成
================================
■ 画面レイアウト（上から下）

```
┌──────────────────────────────────────────┐
│ SYSTEM ACCESS // FALL DODGE // STAGE 2   │ ← ヘッダー帯
├──────────────────────────────────────────┤
│ SCORE: 078  x3 COMBO!   ☁️雨   🛡️ 5s    │ ← HUDバー
│ [P1:YOU] [P2:GUEST1] [P3:GUEST2] [P4:☠] │ ← マルチ時（☠=死亡済み）
├──────────────────────────────────────────┤
│                                          │
│     🔨     📱          🪣               │ ← 3D落下物
│         📕      ✂️           🔑          │
│    NEAR MISS!                            │ ← ニアミス表示
│                                          │
│       🏃P1   🏃P2   🏃P3               │ ← 棒人間（走行中）
├──────────────────────────────────────────┤
│  [← 左]          [右 →]             │ ← 操作ボタン
│  [DASH: ████░░ 2s]                       │ ← ダッシュゲージ
└──────────────────────────────────────────┘
```

■ HUDバー表示項目
・SCORE: スコア
・コンボ倍率（x2以上で表示、パルスアニメーション）
・天候アイコン（雨/雷/嵐）
・パワーアップ残り時間（アイコン+秒数）
・ダッシュクールダウンゲージ

■ プレイヤー表示（マルチ時）
・P1〜P4の4色バッジ
・死亡済みプレイヤーは☠マーク+グレーアウト
・自分に"YOU"表示


================================
24. Canvas描画仕様
================================
■ 棒人間キャラクター
・頭部: 3Dスフィア（グラデーション+ハイライト楕円）
・胴体/手足: グラデーションライン（太→細のストローク）
・各リムに光沢ハイライト（明色オフセットライン）
・地面に楕円シャドウ
・ポーズ: 走行/立ち/ダッシュ/被弾倒れ/勝利万歳

■ 落下物（3D風）
・面ごとのグラデーション（上面明→下面暗）
・押し出し面（右面・底面）の陰影
・ハイライト反射（左上に小さい楕円）
・ドロップシャドウ（オフセット暗色矩形）
・回転アニメーション

■ パワーアップアイテム
・金色の星形、3D風（ハイライト+影）
・上下にゆっくり浮遊する微小アニメーション
・光の粒子エフェクトが周囲に散る

■ 背景
・暗い格子線（グリッド）：ステージごとに色変化
・マトリックス文字列（Canvas外のDOM層、ステージ連動色）

■ レスポンシブ
```javascript
function resizeCanvas() {
    const canvas = document.getElementById('gameCanvas');
    const container = document.getElementById('game-area');
    const aspectRatio = 9 / 16;
    if (window.innerWidth / window.innerHeight < aspectRatio) {
        canvas.width = container.clientWidth;
        canvas.height = canvas.width / aspectRatio;
    } else {
        canvas.height = container.clientHeight;
        canvas.width = canvas.height * aspectRatio;
    }
}
window.addEventListener('resize', resizeCanvas);
```


================================
25. 状態管理（GameState）
================================
```javascript
const GameState = {
    // 画面
    currentScreen: 'splash',

    // ゲーム
    gameMode: 'normal',       // 'normal'|'survival'|'timeAttack'|'teamBattle'|'daily'|'bossRush'|'reverse'
    gameActive: false,
    difficulty: 'normal',
    score: 0,
    comboCount: 0,
    comboMultiplier: 1,
    maxCombo: 0,

    // ステージ
    currentStage: 1,

    // 天候
    weather: 'clear',

    // プレイヤー
    playerX: 50,
    playerWidth: 40,
    playerHeight: 50,         // 棒人間サイズ
    moveLeft: false,
    moveRight: false,
    moveSpeed: 5,
    dashRequested: false,
    dashInvincible: false,

    // パワーアップ
    shield: false,
    speedBoost: false,
    slowMotion: false,
    shrink: false,
    activePowerUps: [],       // [{type, expiresAt}]

    // 落下物
    fallingObjects: [],
    powerUpItems: [],
    currentFallSpeed: 3,
    currentSpawnInterval: 800,
    lastSpawnTime: 0,
    gameStartTime: 0,
    difficultyFrozen: false,
    nextBossTime: 0,

    // マルチプレイ
    roomId: null,
    playerIndex: null,
    isHost: false,
    players: {},              // {0: {x,y,alive,score,...}, 1: {...}, ...}
    maxPlayers: 4,

    // サウンド
    bgmEnabled: true,
    seEnabled: true,

    // ユーザー
    username: null,
    isGuest: false,
    deviceId: null,

    // タイムアタック
    timeRemaining: 60000,

    // アニメーション
    animationId: null,
};
```


================================
26. Firebase設定
================================
■ データ構造
```javascript
{
  "users": {
    "<username>": {
      "deviceId": "xxx",
      "createdAt": timestamp
    }
  },
  "rankings": {
    "<pushId>": {
      "username": "NAME",
      "score": 150,
      "difficulty": "normal",
      "gameMode": "normal",
      "timestamp": timestamp
    }
  },
  "dailyRankings": {
    "<YYYY-MM-DD>": {
      "<pushId>": {
        "username": "NAME",
        "score": 200,
        "timestamp": timestamp
      }
    }
  },
  "rooms": {
    "<roomId>": {
      "status": "waiting",
      "difficulty": "normal",
      "gameMode": "normal",
      "players": {
        "0": {
          "username": "NAME",
          "deviceId": "xxx",
          "x": 50,
          "alive": true,
          "score": 0,
          "team": null,
          "deathTimestamp": null
        },
        "1": { ... },
        "2": { ... },
        "3": { ... }
      },
      "fallingObjects": [
        { "x": 30, "y": 0, "speed": 3, "type": "hammer", "rotation": 0, "id": "obj_001" }
      ],
      "powerUpItems": [],
      "hostIndex": 0,
      "createdAt": timestamp
    }
  }
}
```

■ セキュリティルール
```json
{
  "rules": {
    "users": {
      "$username": {
        ".read": true,
        ".write": "!data.exists() || data.child('deviceId').val() === newData.child('deviceId').val()"
      }
    },
    "rankings": {
      ".read": true,
      ".write": "newData.child('username').exists()"
    },
    "dailyRankings": {
      ".read": true,
      "$date": { ".write": true }
    },
    "rooms": {
      ".read": true,
      "$roomId": { ".write": true }
    }
  }
}
```


================================
27. 同期レイヤー（SyncLayer）
================================
```javascript
const SyncLayer = {
    roomRef: null,
    listeners: [],

    connect(roomId) {
        this.roomRef = database.ref('rooms/' + roomId);
    },

    updatePlayerPosition(playerIndex, x) {
        this.roomRef.child('players/' + playerIndex + '/x').set(x);
    },

    updateFallingObjects(objects) {
        this.roomRef.child('fallingObjects').set(objects);
    },

    updatePowerUpItems(items) {
        this.roomRef.child('powerUpItems').set(items);
    },

    reportDeath(playerIndex, score) {
        this.roomRef.child('players/' + playerIndex).update({
            alive: false,
            score: score,
            deathTimestamp: firebase.database.ServerValue.TIMESTAMP
        });
    },

    // 妨害アイテム: 他プレイヤーに追加落下物
    sendSabotage(targetIndex, objects) {
        this.roomRef.child('sabotage/' + targetIndex).set(objects);
    },

    onPlayersUpdate(cb) {
        const ref = this.roomRef.child('players');
        ref.on('value', snap => cb(snap.val()));
        this.listeners.push(ref);
    },
    onFallingObjectsUpdate(cb) {
        const ref = this.roomRef.child('fallingObjects');
        ref.on('value', snap => cb(snap.val()));
        this.listeners.push(ref);
    },
    onPowerUpItemsUpdate(cb) {
        const ref = this.roomRef.child('powerUpItems');
        ref.on('value', snap => cb(snap.val()));
        this.listeners.push(ref);
    },
    onSabotageUpdate(playerIndex, cb) {
        const ref = this.roomRef.child('sabotage/' + playerIndex);
        ref.on('value', snap => { if (snap.val()) cb(snap.val()); });
        this.listeners.push(ref);
    },
    onStatusChange(cb) {
        const ref = this.roomRef.child('status');
        ref.on('value', snap => cb(snap.val()));
        this.listeners.push(ref);
    },

    disconnect() {
        this.listeners.forEach(ref => ref.off());
        this.listeners = [];
        this.roomRef = null;
    }
};
```


================================
28. 画面遷移
================================
```
[splash-screen]
    ↓ 2.5秒 or タップ
[title-screen]
    ├→ [rule-modal]           ルール説明
    ├→ [ranking-modal]        ランキング（通常/サバイバル/デイリー切替タブ）
    ├→ [mode-select-modal]    ゲームモード選択
    │     ↓ 選択後
    │   [game-screen]          一人でプレイ
    └→ [lobby-screen]         マルチプレイ
          ↓ 全員揃い＋開始
        [game-screen]
            ↓ ゲームオーバー / GAME SET
        [result-screen]
            ├→ [game-screen]   再戦
            └→ [title-screen]  タイトルへ
```


================================
29. 効果音（SE）
================================
【UI】
・ホバー：短い電子ビープ音
・決定：クリック確定音
・エラー：警告音（低め短め）

【ゲーム】
・被弾：爆発/衝撃音
・シールド破壊：バリア破裂音
・ニアミス：高音の通過音
・パワーアップ取得：上昇キラキラ音
・妨害発動：低音ドゥーン
・ダッシュ：風切り音
・巨大落下物予告：警報サイレン
・巨大落下物衝突：重い衝撃音

【演出】
・ステージ遷移：レベルアップ系SE
・勝利：上昇系ネオンSE
・敗北：下降系ゲームオーバーSE
・起動キャッチコピー：SE_MATCH_READY
・退席通知：低音警告SE
・天候変化：環境音切替


================================
30. BGM仕様
================================
・ループ再生
・ステージごとにテンポ/曲調変化
・ON/OFF 切替
・外部音源対応（mp3 / wav）

```javascript
const SoundManager = {
    bgm: null,
    bgmEnabled: true,
    seEnabled: true,

    playBGM(url) {
        if (this.bgm) this.bgm.pause();
        this.bgm = new Audio(url);
        this.bgm.loop = true;
        if (this.bgmEnabled) this.bgm.play().catch(() => {});
    },
    changeBGMForStage(stage) {
        const urls = {
            1: 'bgm_stage1.mp3',
            2: 'bgm_stage2.mp3',
            3: 'bgm_stage3.mp3',
            4: 'bgm_stage4.mp3',
            5: 'bgm_stage5.mp3',
        };
        this.playBGM(urls[stage] || urls[1]);
    },
    playSE(name) {
        if (!this.seEnabled) return;
        const se = new Audio(`se_${name}.mp3`);
        se.play().catch(() => {});
    },
    toggleBGM() {
        this.bgmEnabled = !this.bgmEnabled;
        if (this.bgm) this.bgmEnabled ? this.bgm.play() : this.bgm.pause();
    },
    toggleSE() { this.seEnabled = !this.seEnabled; }
};
```


================================
31. 実績・称号システム
================================
■ 実績一覧

| 実績ID       | 条件                      | 称号テキスト    |
|-------------|--------------------------|---------------|
| dodge_100   | 累計100回回避              | 見習いドッジャー |
| dodge_1000  | 累計1000回回避             | 熟練ドッジャー  |
| combo_50    | 1ゲームでコンボ50達成       | コンボマスター  |
| near_miss_10| 1ゲームでニアミス10回       | スレスレの達人  |
| boss_clear  | ボスラッシュクリア           | ボスハンター    |
| survival_300| サバイバルでスコア300到達    | サバイバー     |
| all_stage   | STAGE 5到達               | ステージ制覇者  |
| win_10      | マルチ10勝                | 対戦王        |
| daily_top   | デイリーランキング1位       | 日刊チャンプ    |
| no_powerup  | パワーアップ未使用でスコア100| ピュアドッジャー |

■ 保存
・Firebase users/<username>/achievements に保存
・称号はユーザーネームの横に表示（1つ選択）

■ 称号表示
・ネームの右に小さいバッジとして表示
・ランキング画面にも反映


================================
32. 終了処理
================================
■ ソロ
・ゲームオーバー → リザルト表示
・選択肢: [もう一度プレイ] / [タイトルに戻る]

■ マルチ
・勝敗確定 → リザルト表示（全プレイヤー順位表示）
・選択肢: [再戦] / [タイトルに戻る]
・全員が「タイトルに戻る」選択 → ルーム削除 → IDロック解放


================================
33. CPU対戦・AI対戦
================================
■ CPUプレイヤー（3段階AI）
ソロ練習やマルチの空き枠を埋めるCPUプレイヤー。

| AIレベル | 名称   | 行動パターン                                    |
|---------|--------|-----------------------------------------------|
| 弱      | ROOKIE | ランダム移動。反応遅延500ms。ダッシュ不使用                |
| 普通    | NORMAL | 落下物の直下を検知して回避。反応遅延200ms                   |
| 強      | EXPERT | 最適経路計算+フェイント。反応遅延50ms。全アクション使用。妨害アイテム積極取得 |

■ AI行動ロジック
```javascript
const CPUBrain = {
    level: 'normal',  // 'rookie' | 'normal' | 'expert'
    reactionDelay: 200,
    lastDecisionTime: 0,

    decide(cpuPlayer, fallingObjects, powerUps, elapsed) {
        if (elapsed - this.lastDecisionTime < this.reactionDelay) return null;
        this.lastDecisionTime = elapsed;

        switch (this.level) {
            case 'rookie':
                return this.rookieLogic(cpuPlayer);
            case 'normal':
                return this.normalLogic(cpuPlayer, fallingObjects);
            case 'expert':
                return this.expertLogic(cpuPlayer, fallingObjects, powerUps);
        }
    },

    rookieLogic(cpu) {
        // ランダムに左右移動、時々停止
        const r = Math.random();
        if (r < 0.35) return 'left';
        if (r < 0.70) return 'right';
        return 'stop';
    },

    normalLogic(cpu, objects) {
        // 最も近い落下物の直下を避ける
        const threats = objects.filter(o =>
            o.y < cpu.y && o.y > cpu.y - 200 &&
            Math.abs(o.x - cpu.x) < 60
        );
        if (threats.length === 0) return 'stop';
        const nearest = threats.reduce((a, b) => a.y > b.y ? a : b);
        return nearest.x > cpu.x ? 'left' : 'right';
    },

    expertLogic(cpu, objects, powerUps) {
        // 複数落下物の安全地帯を計算
        const dangerZones = objects
            .filter(o => o.y < cpu.y && o.y > cpu.y - 300)
            .map(o => ({ center: o.x, radius: o.width + 20 }));

        // パワーアップが近ければ取りに行く
        const nearPU = powerUps.find(p =>
            Math.abs(p.x - cpu.x) < 80 && p.y > cpu.y - 150 && p.y < cpu.y
        );
        if (nearPU) return nearPU.x > cpu.x ? 'right' : 'left';

        // 安全地帯の中心へ移動
        const safeX = this.findSafeZone(cpu.x, dangerZones);
        if (Math.abs(safeX - cpu.x) < 5) return 'stop';
        return safeX > cpu.x ? 'right' : 'left';
    },

    findSafeZone(currentX, dangerZones) {
        // 画面幅を10分割し、最も安全なゾーンを返す
        let bestX = currentX;
        let bestSafety = -1;
        for (let x = 30; x < 770; x += 80) {
            const safety = dangerZones.reduce((score, dz) => {
                return score + Math.min(200, Math.abs(x - dz.center));
            }, 0);
            if (safety > bestSafety) {
                bestSafety = safety;
                bestX = x;
            }
        }
        return bestX;
    }
};
```

■ CPU性格パラメータ
マルチのBot枠で使用。性格によって行動傾向が変わる。

| 性格     | 説明                              | 特徴                    |
|---------|----------------------------------|------------------------|
| 臆病     | 常に画面端に逃げる                  | 安全だが妨害に弱い         |
| 攻撃的   | 妨害アイテムを積極取得               | スコアより嫌がらせ優先      |
| 堅実     | 中央キープ、最小限の移動             | 安定したスコア            |

■ ゴーストモード
・自分の過去ベストプレイを半透明の棒人間（白色、opacity: 0.3）として再生
・操作ログをlocalStorageに保存（タイムスタンプ + x座標 + アクション）
・ソロプレイ開始時に「ゴースト表示 ON/OFF」を選択

```javascript
const GhostSystem = {
    recording: [],
    playback: [],
    playbackIndex: 0,

    startRecording() {
        this.recording = [];
    },
    recordFrame(elapsed, x) {
        this.recording.push({ t: elapsed, x });
    },
    saveAsGhost() {
        const best = localStorage.getItem('fd_ghost_score') || 0;
        if (GameState.score > best) {
            localStorage.setItem('fd_ghost', JSON.stringify(this.recording));
            localStorage.setItem('fd_ghost_score', GameState.score);
        }
    },
    loadGhost() {
        const data = localStorage.getItem('fd_ghost');
        this.playback = data ? JSON.parse(data) : [];
        this.playbackIndex = 0;
    },
    getGhostPosition(elapsed) {
        while (this.playbackIndex < this.playback.length - 1 &&
               this.playback[this.playbackIndex + 1].t <= elapsed) {
            this.playbackIndex++;
        }
        return this.playback[this.playbackIndex] || null;
    }
};
```

■ チュートリアル対戦
・初回プレイ時（localStorageにフラグなし）に自動発動
・CPU（弱）と1戦
・ゲーム中に操作説明を段階的にオーバーレイ表示:
  1. 「← → で移動」（3秒後に消える）
  2. 「Shift でダッシュ」（10秒経過時）
  3. 「★ アイテムを取ろう」（パワーアップ出現時）
・チュートリアル完了後「fd_tutorial_done」フラグを保存


================================
34. ソーシャル機能
================================
■ リプレイ保存・共有
・ゲーム中の操作ログを記録（入力イベント + タイムスタンプ）
・ゲーム終了時に落下物シード＋操作ログをまとめてリプレイデータとする
・リプレイデータをFirebaseに保存（replays/<replayId>）
・リプレイ再生画面: 操作ログに従って落下物と棒人間の動きを再現
・リプレイURLをコピーしてSNS等で共有可能

```javascript
const ReplaySystem = {
    seed: null,
    inputLog: [],     // [{t, action, value}]
    isRecording: false,

    startRecording(seed) {
        this.seed = seed;
        this.inputLog = [];
        this.isRecording = true;
    },
    logInput(elapsed, action, value) {
        if (!this.isRecording) return;
        this.inputLog.push({ t: elapsed, a: action, v: value });
    },
    stopRecording() {
        this.isRecording = false;
    },
    async save(score, username, difficulty, gameMode) {
        const replayId = 'rp_' + Date.now().toString(36);
        await database.ref('replays/' + replayId).set({
            seed: this.seed,
            log: JSON.stringify(this.inputLog),
            score, username, difficulty, gameMode,
            createdAt: firebase.database.ServerValue.TIMESTAMP
        });
        return replayId;
    },
    async load(replayId) {
        const snap = await database.ref('replays/' + replayId).once('value');
        const data = snap.val();
        data.log = JSON.parse(data.log);
        return data;
    }
};
```

■ 観戦モード
・マルチ対戦中のルームIDを入力して観戦
・読み取り専用（Firebaseへの書き込みゼロ）
・観戦者はルームのplayers/fallingObjectsをon('value')で監視
・観戦者数はrooms/<roomId>/spectatorCount で管理
・最大10人観戦
・画面上部に「SPECTATING」バッジ表示
・観戦者は操作不可、スタンプ送信も不可

```javascript
function joinAsSpectator(roomId) {
    const roomRef = database.ref('rooms/' + roomId);
    roomRef.child('spectatorCount').transaction(count => (count || 0) + 1);

    // 読み取りリスナーのみ
    roomRef.child('players').on('value', snap => {
        renderPlayers(snap.val());
    });
    roomRef.child('fallingObjects').on('value', snap => {
        renderFallingObjects(snap.val());
    });

    // 離脱時にカウント減
    window.addEventListener('beforeunload', () => {
        roomRef.child('spectatorCount').transaction(count => Math.max(0, (count || 0) - 1));
    });
}
```

■ スタンプ/エモート
・ゲーム中にワンタップでスタンプ発動
・棒人間の頭上に吹き出し表示（1.5秒で消える）
・Firebase同期で全プレイヤーに表示

| スタンプID | アイコン | 意味      |
|----------|--------|----------|
| good     | 👍     | ナイス     |
| fire     | 🔥     | アツい     |
| shock    | 😱     | やばい     |
| skull    | 💀     | やられた   |
| laugh    | 😂     | ウケる     |
| gg       | 🤝     | お疲れ     |

・クールダウン: 3秒（スパム防止）
・スマホ: 画面下部にスタンプパレット（6個横並び）
・PC: 数字キー1〜6で発動

```javascript
function sendStamp(stampId) {
    if (Date.now() - lastStampTime < 3000) return;
    lastStampTime = Date.now();
    SyncLayer.roomRef.child('stamps/' + GameState.playerIndex).set({
        id: stampId,
        t: firebase.database.ServerValue.TIMESTAMP
    });
}
```

■ フレンドリスト
・ユーザーネーム検索でフレンド申請
・相手が承認すると双方のフレンドリストに追加
・Firebase: users/<name>/friends/<friendName> = { status: 'accepted', since: timestamp }
・フレンドのオンライン状態: users/<name>/online = true/false（onDisconnectでfalse設定）
・フレンドのルームにワンタップ参加ボタン

```javascript
function setOnlineStatus() {
    const userRef = database.ref('users/' + GameState.username);
    userRef.child('online').set(true);
    userRef.child('online').onDisconnect().set(false);
    // 現在のルームIDも公開（フレンドが参加できるよう）
    userRef.child('currentRoom').onDisconnect().remove();
}

function addFriend(friendName) {
    const myRef = database.ref('users/' + GameState.username + '/friends/' + friendName);
    const theirRef = database.ref('users/' + friendName + '/friends/' + GameState.username);
    myRef.set({ status: 'pending', since: firebase.database.ServerValue.TIMESTAMP });
    // 相手側はincoming
    theirRef.set({ status: 'incoming', since: firebase.database.ServerValue.TIMESTAMP });
}

function acceptFriend(friendName) {
    const myRef = database.ref('users/' + GameState.username + '/friends/' + friendName);
    const theirRef = database.ref('users/' + friendName + '/friends/' + GameState.username);
    myRef.update({ status: 'accepted' });
    theirRef.update({ status: 'accepted' });
}
```

■ 戦績プロフィール
・プロフィール画面（モーダル）に以下を集約表示:
  - ユーザーネーム + 称号バッジ
  - 通算プレイ回数
  - 通算勝率（マルチ）
  - 最高スコア（モード別）
  - 最高コンボ
  - 解除済み実績一覧
  - フレンド数
・他人のユーザーネームタップで閲覧可能（読み取りのみ）
・Firebase: users/<name>/stats に集計データ保存

```javascript
function updateStats(result) {
    const statsRef = database.ref('users/' + GameState.username + '/stats');
    statsRef.transaction(stats => {
        stats = stats || { plays: 0, wins: 0, losses: 0, bestScore: 0, bestCombo: 0 };
        stats.plays++;
        if (result === 'win') stats.wins++;
        if (result === 'lose') stats.losses++;
        if (GameState.score > stats.bestScore) stats.bestScore = GameState.score;
        if (GameState.maxCombo > stats.bestCombo) stats.bestCombo = GameState.maxCombo;
        return stats;
    });
}
```


================================
35. シーズンイベント・隠し要素
================================
■ 季節限定落下物
日付に応じて特別な落下物が通常アイテムに混じる（出現率15%）。

| 期間            | イベント     | 限定落下物                  | 描画特徴           |
|----------------|------------|--------------------------|-------------------|
| 1/1〜1/7       | 正月        | 鏡餅、門松、だるま           | 金赤の和風配色       |
| 2/10〜2/14     | バレンタイン  | チョコボックス、ハート        | ピンク+ブラウン      |
| 3/20〜4/10     | 春          | 桜の花びら（小・当たり判定小） | 淡ピンク透明感       |
| 7/1〜8/31      | 夏          | スイカ、風鈴、花火玉         | 赤緑+涼しげブルー    |
| 10/25〜10/31   | ハロウィン   | かぼちゃ、コウモリ、ゴースト   | オレンジ+パープル    |
| 12/20〜12/25   | クリスマス   | プレゼント箱、ツリー、雪だるま | 赤緑ゴールド        |

```javascript
function getSeasonalItems() {
    const now = new Date();
    const m = now.getMonth() + 1;
    const d = now.getDate();

    if (m === 1 && d <= 7)                    return ['mochi', 'kadomatsu', 'daruma'];
    if (m === 2 && d >= 10 && d <= 14)        return ['chocolate', 'heart'];
    if ((m === 3 && d >= 20) || (m === 4 && d <= 10)) return ['sakura'];
    if (m >= 7 && m <= 8)                     return ['watermelon', 'windchime', 'firework'];
    if (m === 10 && d >= 25)                  return ['pumpkin', 'bat', 'ghost'];
    if (m === 12 && d >= 20 && d <= 25)       return ['present', 'xmas_tree', 'snowman'];
    return [];
}
```

■ 季節限定背景エフェクト
マトリックス背景に重なるパーティクルエフェクト。

| 季節   | エフェクト              |
|-------|----------------------|
| 春     | 桜吹雪（ピンク花びら浮遊） |
| 夏     | 花火パーティクル（上部で散る）|
| 秋     | 紅葉（赤黄の葉が舞う）    |
| 冬     | 雪（白い小粒がゆっくり落下）|

・ゲームプレイ最前面ではなく背景層（z-index低）に配置
・opacity低め、ゲームプレイを阻害しない

■ 隠しコマンド
・タイトル画面で特定キー入力で隠しモード解放

| コマンド                | 効果                                |
|-----------------------|-------------------------------------|
| ↑↑↓↓←→←→BA (コナミ) | シークレットモード「CHAOS」解放        |
| FALL と入力            | 全落下物が巨大化モード「GIANT」         |
| 右キー10回連打          | 高速モード「TURBO」（全体速度3倍）      |

```javascript
const SecretCodes = {
    konamiCode: ['up','up','down','down','left','right','left','right','b','a'],
    inputBuffer: [],
    maxBuffer: 20,

    addInput(key) {
        this.inputBuffer.push(key);
        if (this.inputBuffer.length > this.maxBuffer) this.inputBuffer.shift();
        this.checkCodes();
    },
    checkCodes() {
        const recent = this.inputBuffer.slice(-this.konamiCode.length);
        if (JSON.stringify(recent) === JSON.stringify(this.konamiCode)) {
            unlockSecretMode('chaos');
            showToast('🎮 CHAOS MODE UNLOCKED!');
        }
    }
};
```

■ シークレットモード「CHAOS」
・全落下物がゴム製 → バウンド回数無制限（3回まで跳ねる）
・落下速度ランダム変動（毎フレーム±20%揺れ）
・巨大落下物の出現率3倍
・全プレイヤーのスピード1.5倍
・BGMがチップチューンアレンジに変化
・ランキング対象外

■ レアアイテム「金のハンマー」
・出現率: 0.5%
・見た目: 通常ハンマーの金色版、キラキラパーティクル付き
・回避するとスコア+50（コンボ倍率適用前）
・回避時に「GOLDEN DODGE!」の専用演出（画面中央に金文字2秒）
・実績「幸運のドッジャー」解除トリガー

■ 隠しステージ「STAGE ?」
・条件: STAGE 5でスコア500に到達
・背景: グリッチエフェクト（画面が一瞬ずれる演出が繰り返される）
・グリッドカラー: ランダムに色が変わり続ける
・落下物: 全10種がランダム＋回転速度2倍＋落下速度最大
・巨大落下物が15秒間隔（通常30秒）
・専用BGM（不穏なアンビエント系）
・クリア（スコア600到達）で実績「???を超えし者」解除

```javascript
function checkHiddenStage(score, currentStage) {
    if (currentStage === 5 && score >= 500) {
        return { stage: '?', gridColor: 'random', glitch: true, bossInterval: 15000 };
    }
    return null;
}
```

■ 週末限定イベント
・土曜 00:00 〜 日曜 23:59 に自動発動
・タイトル画面に「WEEKEND EVENT!」バナー表示
・週ごとに異なる特殊ルールをローテーション:

| 週番号（月内） | イベント名        | 特殊ルール                      |
|-------------|-----------------|-------------------------------|
| 第1週        | SPEED WEEKEND   | 全体速度2倍                     |
| 第2週        | GIANT WEEKEND   | 巨大落下物のみ                   |
| 第3週        | REVERSE WEEKEND | リバースモード強制               |
| 第4週        | CHAOS WEEKEND   | CHAOSモードルール適用            |

・イベント中のスコアは通常ランキングとは別枠で保存
・Firebase: weekendRankings/<eventType>/<pushId>

```javascript
function getWeekendEvent() {
    const now = new Date();
    const day = now.getDay(); // 0=日, 6=土
    if (day !== 0 && day !== 6) return null;

    const weekNum = Math.ceil(now.getDate() / 7);
    const events = ['speed', 'giant', 'reverse', 'chaos'];
    return events[(weekNum - 1) % events.length];
}
```

■ 棒人間カラーカスタマイズ（隠し報酬）
通常の4色に加え、特殊カラーを実績解除で獲得。

| カラー名   | 解除条件                   | 見た目                        |
|----------|--------------------------|------------------------------|
| 虹色      | 全実績解除                  | 毎フレーム色相が変化するレインボー  |
| 透明      | サバイバル500スコア          | 半透明（opacity: 0.3）         |
| 炎        | コンボx10達成               | 体の周囲にオレンジの炎パーティクル |
| 氷        | デイリーチャレンジ1位3回      | 水色+体の周囲に氷結エフェクト     |
| グリッチ   | 隠しステージ「?」クリア       | 定期的にノイズが走る演出         |
| ゴールド   | 金のハンマーを10回回避        | 全身金色+キラキラ               |

・設定画面（プロフィール内）で選択
・マルチでは基本カラー（P1〜P4）にエフェクトが重なる形で表示
・Firebase: users/<name>/selectedSkin


================================
36. GameState追加フィールド
================================
v3のGameStateに以下を追加：

```javascript
// v4追加分
Object.assign(GameState, {
    // CPU
    cpuPlayers: [],          // [{level, personality, brain: CPUBrain}]
    showGhost: false,

    // ソーシャル
    friendList: [],
    replayId: null,
    isSpectating: false,
    spectatorCount: 0,

    // スタンプ
    lastStampTime: 0,
    activeStamps: {},        // {playerIndex: {id, expiresAt}}

    // シーズン
    seasonalItems: [],
    weekendEvent: null,
    secretModesUnlocked: [],

    // チュートリアル
    tutorialDone: false,
    tutorialStep: 0,

    // カスタマイズ
    selectedSkin: null,      // 'rainbow' | 'transparent' | 'fire' | 'ice' | 'glitch' | 'gold' | null
});
```


================================
37. Firebase追加データ構造
================================
v3のFirebase構造に以下を追加：

```javascript
{
  // 既存構造に追加
  "users": {
    "<username>": {
      // ...既存フィールド...
      "online": true,
      "currentRoom": "XXXXXX",
      "friends": {
        "<friendName>": { "status": "accepted", "since": timestamp }
      },
      "stats": {
        "plays": 100,
        "wins": 42,
        "losses": 58,
        "bestScore": 350,
        "bestCombo": 78
      },
      "achievements": ["dodge_100", "combo_50"],
      "selectedSkin": "fire"
    }
  },
  "replays": {
    "<replayId>": {
      "seed": 123456,
      "log": "[{\"t\":0,\"a\":\"move\",\"v\":50}...]",
      "score": 200,
      "username": "NAME",
      "difficulty": "normal",
      "gameMode": "normal",
      "createdAt": timestamp
    }
  },
  "weekendRankings": {
    "<eventType>": {
      "<pushId>": {
        "username": "NAME",
        "score": 150,
        "timestamp": timestamp
      }
    }
  },
  "rooms": {
    "<roomId>": {
      // ...既存フィールド...
      "spectatorCount": 3,
      "stamps": {
        "0": { "id": "fire", "t": timestamp }
      }
    }
  }
}
```


================================
38. 不正対策・セキュリティ
================================
■ スコア改ざん防止
・クライアント側スコアをそのまま信用しない
・ホスト（ソロ時は自クライアント）がゲーム中に以下を記録:
  - ゲーム開始時刻
  - 落下物シード
  - 最終スコア
  - ゲーム経過時間
・スコア登録時にサーバー側（Firebase Rules）で基本的な妥当性チェック

```javascript
// ランキング登録時の簡易検証データ付与
function submitScore(score, difficulty, gameMode) {
    const elapsed = Date.now() - GameState.gameStartTime;
    const maxPossibleScore = Math.floor(elapsed / 300); // 理論最大値（300msに1回避）
    if (score > maxPossibleScore) return; // 異常値は送信しない

    database.ref('rankings').push({
        username: GameState.username,
        score: score,
        difficulty: difficulty,
        gameMode: gameMode,
        elapsed: elapsed,
        seed: GameState.currentSeed,
        timestamp: firebase.database.ServerValue.TIMESTAMP
    });
}
```

■ Firebaseセキュリティルール強化
```json
{
  "rules": {
    "rankings": {
      ".read": true,
      "$pushId": {
        ".write": "newData.child('username').exists() && newData.child('score').isNumber() && newData.child('score').val() >= 0 && newData.child('score').val() <= 9999",
        ".validate": "newData.child('elapsed').isNumber() && newData.child('elapsed').val() > 0"
      }
    },
    "rooms": {
      "$roomId": {
        ".write": true,
        "players": {
          "$index": {
            "x": { ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 100" },
            "score": { ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 9999" }
          }
        }
      }
    }
  }
}
```

■ レートリミット
・位置同期: 50ms throttle（既存）
・スタンプ: 3秒クールダウン（既存）
・ランキング登録: 1ゲームにつき1回のみ（重複pushID防止）
・フレンド申請: 1分間に最大5件

■ 退席悪用防止
・負けそうな時にわざと切断する行為への対策
・退席回数をFirebase users/<n>/stats/disconnects に記録
・直近10戦で退席率50%以上のユーザーには「⚠ 切断注意」バッジ表示
・マッチング時にロビーで警告表示（参加拒否はしない）

■ ルーム荒らし防止
・ルーム作成は1デバイスにつき同時1ルームまで
・空ルームは作成から5分経過で自動削除
・ゲーム中のルームは30分で強制終了（無限放置防止）

```javascript
// ルーム自動クリーンアップ（ホスト側で実行）
function setupRoomTimeout(roomId) {
    const roomRef = database.ref('rooms/' + roomId);
    // 5分間誰も参加しなければ削除
    const waitingTimeout = setTimeout(() => {
        roomRef.once('value', snap => {
            const room = snap.val();
            if (room && room.status === 'waiting') {
                const playerCount = Object.keys(room.players || {}).length;
                if (playerCount <= 1) roomRef.remove();
            }
        });
    }, 5 * 60 * 1000);

    // 30分で強制終了
    const gameTimeout = setTimeout(() => {
        roomRef.child('status').set('ended');
    }, 30 * 60 * 1000);
}
```


================================
39. 多言語対応
================================
■ 対応言語
・日本語（デフォルト）
・英語

■ 言語切替
・タイトル画面フッターに [JP / EN] トグルボタン
・選択言語はlocalStorageに保存（fd_language）
・ブラウザのnavigator.language で初回自動判定

■ 実装方式
全テキストをi18nオブジェクトで管理。DOM要素にdata-i18n属性を付与。

```javascript
const i18n = {
    ja: {
        title_sub: 'リアルタイム回避アクション',
        catchcopy: '一瞬の判断が、生存を決める',
        btn_register_play: '登録してゲームを始める',
        btn_guest_play: '登録しないでゲームを始める',
        btn_solo: '一人でプレイ',
        btn_multi: 'マルチプレイ（最大4人）',
        btn_rules: 'ルール説明を見る',
        btn_ranking: 'ランキングを見る',
        difficulty_easy: '簡単',
        difficulty_normal: '普通',
        difficulty_hard: '難しい',
        username_placeholder: 'ユーザーネームを入力',
        username_taken: 'このネームは他のユーザーが使用中です',
        game_over: 'ゲームオーバー',
        game_set: 'GAME SET!',
        winner: '勝者',
        score_label: 'スコア',
        rematch: '再戦',
        back_to_title: 'タイトルに戻る',
        retry: 'もう一度プレイ',
        room_create: 'ルームを作る',
        room_join: 'ルームIDで参加',
        waiting: '待機中...',
        game_start: 'ゲーム開始',
        disconnected: 'さんが退席しました',
        near_miss: 'ニアミス!',
        combo: 'コンボ!',
        fever: 'フィーバー!',
        shield_break: 'シールド破壊!',
        golden_dodge: 'ゴールデンドッジ!',
        stage_label: 'ステージ',
        weather_rain: '雨',
        weather_thunder: '雷',
        weather_storm: '嵐',
        tutorial_move: '← → で移動',
        tutorial_dash: 'Shift でダッシュ',
        tutorial_item: '★ アイテムを取ろう',
        spectating: '観戦中',
        profile: '戦績',
        friends: 'フレンド',
        achievements: '実績',
        weekend_event: '週末イベント開催中!',
        bgm_on: 'BGM: ON',
        bgm_off: 'BGM: OFF',
        se_on: 'SE: ON',
        se_off: 'SE: OFF',
    },
    en: {
        title_sub: 'REAL-TIME DODGE ACTION',
        catchcopy: 'A split-second decision determines survival.',
        btn_register_play: 'Register & Play',
        btn_guest_play: 'Play as Guest',
        btn_solo: 'Solo Play',
        btn_multi: 'Multiplayer (up to 4)',
        btn_rules: 'How to Play',
        btn_ranking: 'Rankings',
        difficulty_easy: 'Easy',
        difficulty_normal: 'Normal',
        difficulty_hard: 'Hard',
        username_placeholder: 'Enter username',
        username_taken: 'This name is already taken',
        game_over: 'GAME OVER',
        game_set: 'GAME SET!',
        winner: 'WINNER',
        score_label: 'SCORE',
        rematch: 'Rematch',
        back_to_title: 'Back to Title',
        retry: 'Play Again',
        room_create: 'Create Room',
        room_join: 'Join by Room ID',
        waiting: 'Waiting...',
        game_start: 'Start Game',
        disconnected: ' has disconnected',
        near_miss: 'NEAR MISS!',
        combo: 'COMBO!',
        fever: 'FEVER!',
        shield_break: 'SHIELD BREAK!',
        golden_dodge: 'GOLDEN DODGE!',
        stage_label: 'STAGE',
        weather_rain: 'Rain',
        weather_thunder: 'Thunder',
        weather_storm: 'Storm',
        tutorial_move: '← → to move',
        tutorial_dash: 'Shift to dash',
        tutorial_item: '★ Grab power-ups!',
        spectating: 'SPECTATING',
        profile: 'Profile',
        friends: 'Friends',
        achievements: 'Achievements',
        weekend_event: 'WEEKEND EVENT!',
        bgm_on: 'BGM: ON',
        bgm_off: 'BGM: OFF',
        se_on: 'SE: ON',
        se_off: 'SE: OFF',
    }
};

let currentLang = localStorage.getItem('fd_language') ||
    (navigator.language.startsWith('ja') ? 'ja' : 'en');

function t(key) {
    return i18n[currentLang][key] || i18n['en'][key] || key;
}

function switchLanguage(lang) {
    currentLang = lang;
    localStorage.setItem('fd_language', lang);
    document.querySelectorAll('[data-i18n]').forEach(el => {
        el.textContent = t(el.dataset.i18n);
    });
}
```

■ Canvas内テキスト
Canvas描画のテキスト（NEAR MISS!, COMBO! 等）も t() 関数経由で出力。

■ 実況テキスト（§41）も言語連動。


================================
40. アクセシビリティ
================================
■ 色覚対応
・デフォルトのネオンカラーに加え、色覚多様性対応モードを設定画面に追加
・3パターン切替: 通常 / 第1色覚（赤緑） / 第2色覚（青黄）

| 要素         | 通常            | 第1色覚対応      | 第2色覚対応      |
|-------------|-----------------|-----------------|-----------------|
| P1カラー     | ピンク(255,50,150)| オレンジ(255,165,0)| オレンジ(255,165,0)|
| P2カラー     | シアン(0,220,255) | ブルー(0,100,255) | ピンク(255,105,180)|
| P3カラー     | イエロー(255,220,0)| イエロー(255,220,0)| グリーン(0,200,0)  |
| P4カラー     | パープル(180,80,255)| ホワイト(220,220,220)| ホワイト(220,220,220)|
| 危険表示      | レッド           | オレンジ+形状マーク | オレンジ+形状マーク |
| パワーアップ   | ゴールド         | ゴールド+星形強調  | ゴールド+星形強調  |

・色だけでなく形状・パターン・テキストラベルでも区別可能にする
・プレイヤーバッジに色＋P番号テキストを常に併記（既存仕様で対応済み）

```javascript
const ColorSchemes = {
    normal: {
        p1: [255,50,150], p2: [0,220,255], p3: [255,220,0], p4: [180,80,255],
        danger: [255,51,102], powerup: [255,204,0]
    },
    protanopia: {
        p1: [255,165,0], p2: [0,100,255], p3: [255,220,0], p4: [220,220,220],
        danger: [255,165,0], powerup: [255,204,0]
    },
    tritanopia: {
        p1: [255,165,0], p2: [255,105,180], p3: [0,200,0], p4: [220,220,220],
        danger: [255,165,0], powerup: [255,204,0]
    }
};

let colorScheme = localStorage.getItem('fd_color_scheme') || 'normal';

function getPlayerColor(index) {
    return ColorSchemes[colorScheme]['p' + (index + 1)];
}
```

■ 操作リマップ
・設定画面でキーバインドを変更可能
・デフォルト: ←→ = 移動、Shift/Space = ダッシュ
・WASD対応（A=左、D=右、W or S=ダッシュ）
・localStorageに保存（fd_keybinds）

```javascript
const defaultKeyBinds = {
    moveLeft:  ['ArrowLeft', 'a', 'A'],
    moveRight: ['ArrowRight', 'd', 'D'],
    dash:      [' ', 'Shift', 'w', 'W'],
    stamp1: ['1'], stamp2: ['2'], stamp3: ['3'],
    stamp4: ['4'], stamp5: ['5'], stamp6: ['6'],
};

let keyBinds = JSON.parse(localStorage.getItem('fd_keybinds')) || defaultKeyBinds;

function isKeyBound(key, action) {
    return keyBinds[action] && keyBinds[action].includes(key);
}

document.addEventListener('keydown', (e) => {
    if (isKeyBound(e.key, 'moveLeft'))  GameState.moveLeft = true;
    if (isKeyBound(e.key, 'moveRight')) GameState.moveRight = true;
    if (isKeyBound(e.key, 'dash'))      GameState.dashRequested = true;
});
```

■ 振動フィードバック（スマートフォン）
・Vibration API対応端末で振動を付与
・ON/OFF切替（設定画面、デフォルトON）

| イベント        | 振動パターン(ms)     |
|---------------|-------------------|
| 被弾           | [100, 30, 100]    |
| シールド破壊    | [50, 20, 50]      |
| ニアミス        | [30]              |
| ダッシュ        | [20]              |
| 巨大落下物衝撃  | [150, 50, 150]    |
| ステージ遷移    | [40, 40, 40]      |

```javascript
const VibrationSystem = {
    enabled: true,

    vibrate(pattern) {
        if (!this.enabled) return;
        if ('vibrate' in navigator) {
            navigator.vibrate(pattern);
        }
    },
    onHit()         { this.vibrate([100, 30, 100]); },
    onShieldBreak() { this.vibrate([50, 20, 50]); },
    onNearMiss()    { this.vibrate([30]); },
    onDash()        { this.vibrate([20]); },
    onBossImpact()  { this.vibrate([150, 50, 150]); },
    onStageChange() { this.vibrate([40, 40, 40]); },
};
```

■ その他
・フォントサイズ: HUDテキストはCanvas内描画のため、全体スケールで自動調整（既存レスポンシブ仕様で対応）
・ハイコントラストモード: 背景を純黒、グリッド非表示で落下物の視認性を最大化
・画面フラッシュ軽減: 雷エフェクトのフラッシュをOFFにできるオプション（光過敏対応）


================================
41. 実況テキストシステム
================================
■ 概要
ゲーム中にリアルタイムで実況テキストが画面下部に流れる。
AIがプレイ状況を判定し、適切なコメントを表示。

■ 表示位置
・ゲームCanvasの下部（操作ボタンの上）
・半透明背景バー上にネオングリーンテキスト
・1行表示、2秒でフェードアウト → 次のテキストに更新
・同時に2つ以上のテキストは表示しない（キューで管理）

■ 実況トリガーとテキスト例

| トリガー               | 日本語例                          | 英語例                          |
|-----------------------|----------------------------------|--------------------------------|
| ゲーム開始             | 「ゲームスタート！生き延びろ！」      | "Game start! Survive!"         |
| 初回回避              | 「最初の回避成功！いい滑り出しだ」    | "First dodge! Good start!"     |
| コンボx2到達          | 「コンボ発動！この調子だ」           | "Combo activated! Keep it up!" |
| コンボx5到達          | 「フィーバー突入！止まらない！」      | "FEVER mode! Unstoppable!"     |
| コンボx10到達         | 「MAX!! 神回避の領域だ！」          | "MAX!! Godlike dodging!"       |
| ニアミス              | 「危ない！紙一重だった！」           | "Close call! That was tight!"  |
| 連続ニアミス3回        | 「攻めすぎだ！スリルを楽しんでるな」  | "Living on the edge!"          |
| シールド取得           | 「シールドGET！保険をかけた」        | "Shield acquired! Safety net!" |
| シールド破壊           | 「シールド消滅！残りは腕だけだ」      | "Shield gone! Skill only now!" |
| スロー発動            | 「スローモーション！チャンスタイム」    | "Slow motion! Seize the chance!"|
| 巨大落下物予告         | 「巨大落下物接近！気をつけろ！」      | "BOSS incoming! Watch out!"    |
| 巨大落下物回避         | 「ナイス！巨大物を回避した！」        | "Nice! Boss dodged!"           |
| 金のハンマー出現       | 「金のハンマー発見！避ければ+50！」   | "Golden Hammer! Dodge for +50!"|
| 金のハンマー回避       | 「ゴールデンドッジ！大量ポイント！」   | "GOLDEN DODGE! Jackpot!"       |
| ステージ遷移           | 「ステージ2突入！キッチンへようこそ」  | "Stage 2! Welcome to Kitchen!" |
| 天候変化（嵐）         | 「嵐が来た！落下物が斜めに飛ぶぞ」   | "Storm! Objects flying sideways!"|
| スコア100到達          | 「100回避達成！まだまだ行ける！」     | "100 dodges! Still going!"     |
| マルチ: 誰か被弾      | 「P2が脱落！残り3人の戦い！」        | "P2 is out! 3 remain!"         |
| マルチ: 残り2人       | 「一騎打ち！最後の勝負だ！」         | "1v1! Final showdown!"         |
| マルチ: 妨害発動      | 「妨害発動！落下物が追加された！」    | "Sabotage! Extra objects incoming!"|
| 被弾直前の膠着        | 「落下物が密集してきた…厳しい！」     | "Objects closing in... Danger!" |
| 長時間生存(60秒)      | 「1分経過！タフな生存者だ」          | "1 minute survived! Tough!"    |
| 長時間生存(120秒)     | 「2分経過！もはや伝説級だ」          | "2 minutes! Legendary!"        |

■ 実装

```javascript
const CommentarySystem = {
    queue: [],
    currentText: null,
    displayUntil: 0,
    lastCommentTime: 0,
    minInterval: 3000,  // 最低3秒間隔（頻度制限）

    // テキスト追加（優先度付き）
    add(key, priority = 0) {
        const text = t('commentary_' + key) || key;
        this.queue.push({ text, priority, time: Date.now() });
        // 優先度順にソート
        this.queue.sort((a, b) => b.priority - a.priority);
    },

    // フレーム更新
    update() {
        const now = Date.now();

        // 現在のテキストが期限切れなら消す
        if (this.currentText && now > this.displayUntil) {
            this.currentText = null;
        }

        // キューにテキストがあり、間隔条件を満たしていれば表示
        if (!this.currentText && this.queue.length > 0 &&
            now - this.lastCommentTime >= this.minInterval) {
            const next = this.queue.shift();
            this.currentText = next.text;
            this.displayUntil = now + 2000;
            this.lastCommentTime = now;
        }
    },

    // 描画
    render(ctx, canvasWidth, y) {
        if (!this.currentText) return;
        const remaining = this.displayUntil - Date.now();
        const alpha = Math.min(1, remaining / 500); // 最後0.5秒でフェードアウト

        ctx.save();
        ctx.globalAlpha = alpha * 0.85;
        ctx.fillStyle = 'rgba(0, 10, 4, 0.7)';
        ctx.fillRect(0, y, canvasWidth, 28);
        ctx.globalAlpha = alpha;
        ctx.fillStyle = '#00ff41';
        ctx.font = '14px "GeistMono", monospace';
        ctx.textAlign = 'center';
        ctx.fillText(this.currentText, canvasWidth / 2, y + 19);
        ctx.restore();
    }
};

// ゲームループ内でのトリガー例
function checkCommentaryTriggers() {
    if (GameState.comboCount === 10) CommentarySystem.add('combo_x2', 1);
    if (GameState.comboCount === 50) CommentarySystem.add('combo_x5', 2);
    if (GameState.comboCount === 100) CommentarySystem.add('combo_x10', 3);
    if (GameState.score === 100) CommentarySystem.add('score_100', 1);
    // ... 各トリガー条件
}
```

■ 実況ON/OFF
・設定画面で切替可能（デフォルトON）
・localStorage: fd_commentary_enabled


================================
42. 実装順序（推奨）
================================
Phase 1: 基盤
 1. HTMLスケルトン + 全screen DOM構造
 2. CSS: ネオンテーマ全体（変数、ボタン、モーダル）
 3. マトリックス背景アニメーション
 4. 画面遷移（showScreen / showModal）
 5. 起動キャッチコピー演出

Phase 2: タイトル + ユーザー管理
 6. タイトル画面UI全配置
 7. Firebase初期化 + ユーザー登録/ゲスト処理
 8. ランキングモーダル（Firebase取得・表示）
 9. ルール説明モーダル
 10. ゲームモード選択UI

Phase 3: ソロモード（ノーマル）
 11. Canvas初期化 + レスポンシブ
 12. 棒人間描画（3Dスフィア頭部 + リムライン）
 13. 左右移動（PC + スマホ）
 14. 3D落下物生成 + 回転 + 落下 + 描画
 15. 当たり判定 + ゲームオーバー
 16. バウンド物理
 17. 難易度システム（時間経過パラメータ更新）
 18. スコア + HUD表示
 19. リザルト画面 + ランキング登録

Phase 4: 新メカニクス
 20. ダッシュシステム + 残像エフェクト + クールダウンHUD
 21. パワーアップアイテム（生成・取得・効果・HUD表示）
 22. ニアミス判定 + コンボシステム
 23. 巨大ボス落下物（予告影 + 落下 + 衝撃演出）

Phase 5: ステージ + 天候
 25. ステージ制（スコア閾値 + 背景色変化 + 遷移演出）
 26. 天候エフェクト（雨/雷/嵐の描画 + ゲーム効果）
 27. 落下物衝突パーティクル + 画面フラッシュ

Phase 6: マルチプレイ
 28. ロビー画面（4人対応）+ ルーム作成/参加
 29. SyncLayer実装
 30. ホスト落下物生成 → Firebase同期
 31. ゲスト落下物受信 → 描画
 32. プレイヤー位置相互同期（x座標）
 33. 勝敗判定（最後の1人） + 退席処理
 34. 妨害アイテム同期
 35. 再戦 / 終了処理
 36. スタンプ/エモート同期

Phase 7: ゲームモード
 37. サバイバルモード（無限加速）
 38. タイムアタックモード（60秒 + ペナルティ）
 39. チーム戦（2v2 チーム分け + チーム全滅判定）
 40. デイリーチャレンジ（シード固定 + 日替わりランキング）
 41. ボスラッシュ（巨大落下物オンリー + 10体クリア）
 42. リバースモード（上下反転）

Phase 8: CPU・AI
 43. CPUBrainロジック（3段階AI）
 44. CPU性格パラメータ
 45. ゴーストモード（録画・再生・保存）
 46. チュートリアル対戦（初回自動発動）

Phase 9: ソーシャル
 47. リプレイ保存・再生・共有
 48. 観戦モード
 49. フレンドリスト（申請・承認・オンライン状態）
 50. 戦績プロフィール

Phase 10: シーズン・隠し要素
 51. 季節限定落下物 + 背景エフェクト
 52. 隠しコマンド（コナミコード等）
 53. シークレットモード「CHAOS」
 54. レアアイテム「金のハンマー」
 55. 隠しステージ「STAGE ?」
 56. 週末限定イベント
 57. 棒人間カラーカスタマイズ（実績報酬）

Phase 11: 仕上げ
 58. サウンド統合（BGM / SE）
 59. 実績・称号システム
 60. レスポンシブ最終調整（縦横両対応）
 61. エッジケーステスト

Phase 12: セキュリティ・アクセシビリティ・i18n
 62. 不正対策（スコア検証・Firebase Rules強化・レートリミット）
 63. 退席悪用防止・ルーム自動クリーンアップ
 64. 多言語対応（i18nオブジェクト・言語切替UI・Canvas内テキスト）
 65. 色覚対応カラースキーム（3パターン切替）
 66. 操作リマップ（キーバインド設定UI）
 67. 振動フィードバック（Vibration API）
 68. ハイコントラストモード・フラッシュ軽減オプション
 69. 実況テキストシステム（CommentarySystem実装・トリガー設定）


================================
43. デザインリファレンス
================================
UIデザインは LINKED BLOCKS_ の以下の要素を継承：

・ヘッダー帯:「SYSTEM ACCESS // ...」のdim green帯
・HUDバー: ネオン枠 + モノスペース + dark panel背景
・プレイヤーバッジ: [P1: YOU] [P2: GUEST 1] ... [P4: GUEST 3] の4色枠表示
・ボタン: ネオンアウトライン + ホバーグロー + 浮き上がり
・モーダル: 暗転背景 + ネオン枠ボックス
・リザルト: 「GAME SET!」大文字ネオン発光 + WINNER棒人間万歳ポーズ + 全員順位表
・枠線: 二重ネオン枠（外枠太/内枠細）

参照画像:
・UIサンプル.jpg（LINKED BLOCKS_ のPC/スマホ実装例）
・FALL_DODGE_UI_Sample.png（FALL DODGE 3D版UIモックアップ）


================================
44. 設計思想まとめ
================================
FALL DODGE は、
・LINKED BLOCKS_ と同一のデザイン体系を継承
・Firebase Realtime Databaseで完結するリアルタイム同期
・単一HTMLファイルで完全動作
・最大4人対戦の棒人間キャラクター（特殊スキン対応）
・3D風の日用品落下物（10種+巨大ボス3種+季節限定）
・ステージ制×天候×パワーアップの重層的なゲーム体験
・ダッシュ・コンボによる奥深い操作性
・7つのゲームモード＋シークレットモードによるリプレイ性
・3段階CPU AI＋ゴーストモードでソロの練習環境
・リプレイ共有・観戦・フレンド・スタンプのソーシャル体験
・季節イベント・隠しコマンド・レアアイテムの発見の楽しさ
・実績・称号・カラーカスタマイズの長期モチベーション
・リアルタイム実況テキストによる臨場感
・日本語/英語の多言語対応
・色覚多様性対応・操作リマップ・振動フィードバックのアクセシビリティ
・スコア検証・退席対策・ルーム管理のセキュリティ
・退席にも破綻しない堅牢なルーム管理
・Canvas + DOM層の分離による高パフォーマンス描画
・AI開発環境でそのまま実装可能な仕様記述

完成度とプレイ体験を重視した
リアルタイム回避アクションゲームである。
