# vibe-coding1
import pyxel
import random
from enum import Enum
from typing import Optional


# ゲーム状態を管理する列挙型
class GameState(Enum):
    """ゲーム全体の状態管理"""
    TITLE = 1
    PLAYING = 2
    RESULT = 3


# アクション種別を管理する列挙型
class ActionType(Enum):
    """ダンスアクションの種類"""
    CLAP = "clap"
    STEP_FORWARD = "step_forward"
    STEP_LEFT = "step_left"
    STEP_BACK = "step_back"
    STEP_RIGHT = "step_right"
    POSE = "pose"
    JUMP = "jump"


# 入力キーと対応するアクション
KEY_TO_ACTION = {
    pyxel.KEY_C: ActionType.CLAP,
    pyxel.KEY_W: ActionType.STEP_FORWARD,
    pyxel.KEY_A: ActionType.STEP_LEFT,
    pyxel.KEY_S: ActionType.STEP_BACK,
    pyxel.KEY_D: ActionType.STEP_RIGHT,
    pyxel.KEY_P: ActionType.POSE,
    pyxel.KEY_J: ActionType.JUMP,
}

# アクション表示用テキスト
ACTION_DISPLAY = {
    ActionType.CLAP: "C",
    ActionType.STEP_FORWARD: "W",
    ActionType.STEP_LEFT: "A",
    ActionType.STEP_BACK: "S",
    ActionType.STEP_RIGHT: "D",
    ActionType.POSE: "P",
    ActionType.JUMP: "J",
}


# ダンスシナリオの定義
DANCE_SCENARIOS = [
    [ActionType.CLAP, ActionType.STEP_FORWARD, ActionType.POSE, ActionType.JUMP],
    [ActionType.STEP_LEFT, ActionType.CLAP, ActionType.STEP_RIGHT, ActionType.POSE],
    [ActionType.JUMP, ActionType.STEP_FORWARD, ActionType.CLAP, ActionType.STEP_BACK],
    [ActionType.POSE, ActionType.CLAP, ActionType.STEP_LEFT, ActionType.JUMP],
    [ActionType.STEP_FORWARD, ActionType.POSE, ActionType.STEP_BACK, ActionType.CLAP],
    [ActionType.CLAP, ActionType.CLAP, ActionType.STEP_FORWARD, ActionType.POSE],
    [ActionType.JUMP, ActionType.STEP_LEFT, ActionType.STEP_RIGHT, ActionType.CLAP],
    [ActionType.POSE, ActionType.STEP_FORWARD, ActionType.JUMP, ActionType.CLAP],
    [ActionType.STEP_RIGHT, ActionType.CLAP, ActionType.POSE, ActionType.STEP_LEFT],
    [ActionType.CLAP, ActionType.JUMP, ActionType.POSE, ActionType.STEP_FORWARD],
]


class GameManager:
    """ゲーム全体を管理するクラス"""

    def __init__(self) -> None:
        # ゲーム画面の設定（初期化済みの場合はスキップ）
        if not hasattr(pyxel, 'width'):
            pyxel.init(256, 192, title="Dance Master", fps=30)

        # ゲーム状態の初期化
        self.state = GameState.TITLE
        self.score = 100
        self.time_left = 30.0
        self.frame_count = 0

        # BPM関連
        self.bpm = 100
        self.beat_count = 0

        # ダンス関連
        self.current_scenario: list[ActionType] = DANCE_SCENARIOS[0]
        self.coach_action_index = 0
        self.player_action_index = 0
        self.beat_phase = 0  # 0: コーチ表示, 1: プレイヤー入力待機
        self.beat_timer = 0.0
        self.action_timer = 0.0

        # 入力判定関連
        self.judgments: list[str] = []
        self.expected_actions: list[ActionType] = []
        self.current_action_received = False

    def update(self) -> None:
        """ゲーム状態を毎フレーム更新"""
        if self.state == GameState.TITLE:
            self._update_title()
        elif self.state == GameState.PLAYING:
            self._update_playing()
        elif self.state == GameState.RESULT:
            self._update_result()

    def _update_title(self) -> None:
        """タイトル画面の更新"""
        # スペースキーでゲーム開始
        if pyxel.btnp(pyxel.KEY_SPACE):
            self._start_game()

    def _update_playing(self) -> None:
        """ゲームプレイ中の更新"""
        # 時間をデクリメント
        self.frame_count += 1
        if self.frame_count % 30 == 0:  # 30フレーム=1秒
            self.time_left -= 1

        # ゲーム終了判定
        if self.time_left <= 0:
            self.state = GameState.RESULT
            return

        # ビート処理
        self._update_game_loop()

        # プレイヤー入力処理
        self._handle_player_input()

    def _update_result(self) -> None:
        """リザルト画面の更新"""
        # Rキーでリトライ
        if pyxel.btnp(pyxel.KEY_R):
            self._start_game()

    def _update_game_loop(self) -> None:
        """ゲームループの更新"""
        beat_duration = 4.0 / (self.bpm / 100.0)
        self.beat_timer += 1.0 / 30.0
        self.action_timer += 1.0 / 30.0

        # 各アクションのタイミングチェック
        action_duration = beat_duration / len(self.current_scenario)

        if self.action_timer >= action_duration:
            self.action_timer = 0.0

            if self.beat_phase == 0:
                # コーチのダンス表示フェーズ
                self.coach_action_index += 1

                if self.coach_action_index >= len(self.current_scenario):
                    # コーチの手本終了、プレイヤーフェーズへ
                    self.beat_phase = 1
                    self.coach_action_index = 0
                    self.player_action_index = 0
                    self.expected_actions = list(self.current_scenario)
                    self.current_action_received = False
            else:
                # プレイヤー入力フェーズ
                if self.player_action_index < len(self.expected_actions):
                    self._judge_current_action()
                    self.current_action_received = False
                    self.player_action_index += 1

                    if self.player_action_index >= len(self.expected_actions):
                        # シナリオ完了
                        self._next_scenario()

    def _judge_current_action(self) -> None:
        """現在のアクション判定"""
        expected = self.expected_actions[self.player_action_index]

        if self.current_action_received:
            self.judgments.append("Perfect")
        else:
            self.judgments.append("Bad")
            self.score = max(0, self.score - 5)

    def _handle_player_input(self) -> None:
        """プレイヤーの入力を処理"""
        for key, action in KEY_TO_ACTION.items():
            if pyxel.btnp(key):
                # プレイヤーフェーズで入力があるかチェック
                if self.beat_phase == 1 and self.player_action_index < len(
                    self.expected_actions
                ):
                    expected = self.expected_actions[self.player_action_index]
                    if action == expected:
                        self.current_action_received = True

    def _start_game(self) -> None:
        """ゲームを開始"""
        self.state = GameState.PLAYING
        self.score = 100
        self.time_left = 30.0
        self.frame_count = 0
        self.bpm = 100
        self.beat_count = 0
        self.current_scenario = random.choice(DANCE_SCENARIOS)
        self.coach_action_index = 0
        self.player_action_index = 0
        self.beat_phase = 0
        self.beat_timer = 0.0
        self.action_timer = 0.0
        self.judgments = []
        self.expected_actions = []
        self.current_action_received = False

    def _next_scenario(self) -> None:
        """次のシナリオに進む"""
        self.bpm = min(200, self.bpm + 20)
        self.current_scenario = random.choice(DANCE_SCENARIOS)
        self.beat_phase = 0
        self.coach_action_index = 0
        self.player_action_index = 0
        self.beat_timer = 0.0
        self.action_timer = 0.0
        self.beat_count += 1
        self.expected_actions = []
        self.current_action_received = False

    def draw(self) -> None:
        """画面描画"""
        if self.state == GameState.TITLE:
            self._draw_title()
        elif self.state == GameState.PLAYING:
            self._draw_playing()
        elif self.state == GameState.RESULT:
            self._draw_result()

    def _draw_title(self) -> None:
        """タイトル画面を描画"""
        # 黒背景
        pyxel.cls(0)
        # タイトル表示
        title_text = "DANCE MASTER"
        pyxel.text(
            128 - len(title_text) * 2, 80, title_text, pyxel.COLOR_WHITE
        )
        # スタート指示
        pyxel.text(
            128 - 25, 120, "PRESS SPACE TO START", pyxel.COLOR_YELLOW
        )

    def _draw_playing(self) -> None:
        """ゲームプレイ中の画面を描画"""
        # スポットライト背景描画
        pyxel.cls(0)
        self._draw_stage()

        # コーチ描画
        self._draw_coach()

        # プレイヤー描画
        self._draw_player()

        # UI表示（スコア・時間）
        pyxel.text(5, 5, f"SCORE: {self.score}", pyxel.COLOR_WHITE)
        pyxel.text(5, 15, f"TIME: {max(0, int(self.time_left))}", pyxel.COLOR_WHITE)
        pyxel.text(5, 25, f"BPM: {self.bpm}", pyxel.COLOR_WHITE)

        # コーチのアクション表示（現在実行中のアクション）
        if self.beat_phase == 0 and self.coach_action_index > 0 and \
           self.coach_action_index <= len(self.current_scenario):
            action = self.current_scenario[self.coach_action_index - 1]
            if action in ACTION_DISPLAY:
                pyxel.text(
                    110,
                    150,
                    f"Coach: {ACTION_DISPLAY[action]}",
                    pyxel.COLOR_CYAN
                )

    def _draw_result(self) -> None:
        """リザルト画面を描画"""
        pyxel.cls(0)
        pyxel.text(128 - 30, 30, "GAME OVER", pyxel.COLOR_WHITE)
        pyxel.text(128 - 20, 60, f"SCORE: {self.score}", pyxel.COLOR_YELLOW)
        pyxel.text(
            128 - 40, 90, "JUDGMENTS:", pyxel.COLOR_WHITE
        )
        # 判定結果表示
        y_offset = 105
        for i, judgment in enumerate(self.judgments[-10:]):  # 最後10個表示
            pyxel.text(128 - 20, y_offset + i * 8, judgment, pyxel.COLOR_WHITE)

        pyxel.text(
            128 - 50, 160, "PRESS R TO RETRY", pyxel.COLOR_CYAN
        )

    def _draw_stage(self) -> None:
        """ダンスステージを描画"""
        # スポットライト効果（円形グラデーション）
        pyxel.circfill(128, 96, 80, 1)

    def _draw_coach(self) -> None:
        """コーチキャラクターを描画"""
        coach_x = 60
        coach_y = 100

        # 現在のアクションを取得
        current_action = None
        if self.beat_phase == 0 and self.coach_action_index > 0 and \
           self.coach_action_index <= len(self.current_scenario):
            current_action = self.current_scenario[self.coach_action_index - 1]

        # ヒップホップダンサーのシンプルなスティック図描画
        self._draw_dancer(coach_x, coach_y, pyxel.COLOR_WHITE, current_action)

    def _draw_player(self) -> None:
        """プレイヤーキャラクターを描画"""
        player_x = 196
        player_y = 100

        # プレイヤーは通常ポーズで表示
        self._draw_dancer(player_x, player_y, pyxel.COLOR_CYAN, None)

    def _draw_dancer(
        self, x: int, y: int, color: int, action: Optional[ActionType]
    ) -> None:
        """ダンサーキャラクターを描画"""
        # 頭
        pyxel.circfill(x, y - 25, 4, color)

        # 体
        pyxel.line(x, y - 20, x, y, color)

        # アクションに応じた腕と脚の動き
        if action == ActionType.CLAP:
            # クラップ：両腕を上げる
            pyxel.line(x - 2, y - 12, x - 8, y - 20, color)
            pyxel.line(x + 2, y - 12, x + 8, y - 20, color)
            # 脚は閉じる
            pyxel.line(x - 1, y, x - 2, y + 12, color)
            pyxel.line(x + 1, y, x + 2, y + 12, color)
        elif action == ActionType.STEP_FORWARD:
            # ステップ前：右脚前
            pyxel.line(x - 2, y - 5, x - 4, y + 3, color)
            pyxel.line(x + 2, y, x + 4, y + 10, color)
            # 腕は自然
            pyxel.line(x, y - 10, x - 6, y - 2, color)
            pyxel.line(x, y - 10, x + 6, y - 2, color)
        elif action == ActionType.STEP_LEFT:
            # ステップ左：左脚が出る
            pyxel.line(x - 2, y, x - 8, y + 10, color)
            pyxel.line(x + 2, y - 5, x + 4, y + 3, color)
            # 腕は左に
            pyxel.line(x, y - 10, x - 8, y - 8, color)
            pyxel.line(x, y - 10, x + 4, y - 2, color)
        elif action == ActionType.STEP_BACK:
            # ステップ後：左脚後ろ
            pyxel.line(x - 2, y, x - 2, y + 12, color)
            pyxel.line(x + 2, y - 5, x + 6, y + 3, color)
            # 腕は後ろ
            pyxel.line(x, y - 10, x - 4, y - 2, color)
            pyxel.line(x, y - 10, x + 8, y - 8, color)
        elif action == ActionType.STEP_RIGHT:
            # ステップ右：右脚が出る
            pyxel.line(x - 2, y - 5, x - 4, y + 3, color)
            pyxel.line(x + 2, y, x + 8, y + 10, color)
            # 腕は右に
            pyxel.line(x, y - 10, x - 4, y - 2, color)
            pyxel.line(x, y - 10, x + 8, y - 8, color)
        elif action == ActionType.POSE:
            # ポーズ：決めポーズ（腕L字）
            pyxel.line(x - 2, y - 10, x - 2, y - 18, color)
            pyxel.line(x - 2, y - 18, x - 8, y - 18, color)
            pyxel.line(x + 2, y - 10, x + 6, y, color)
            # 脚は閉じる
            pyxel.line(x - 1, y, x - 1, y + 12, color)
            pyxel.line(x + 1, y, x + 1, y + 12, color)
        elif action == ActionType.JUMP:
            # ジャンプ：両脚上げ、両腕上げ
            pyxel.line(x - 2, y - 5, x - 6, y - 10, color)
            pyxel.line(x + 2, y - 5, x + 6, y - 10, color)
            # 脚は上げる
            pyxel.line(x - 2, y, x - 4, y - 5, color)
            pyxel.line(x + 2, y, x + 4, y - 5, color)
        else:
            # 通常ポーズ：立ち姿勢
            pyxel.line(x, y - 10, x - 6, y, color)
            pyxel.line(x, y - 10, x + 6, y, color)
            # 脚は自然
            pyxel.line(x - 2, y, x - 3, y + 12, color)
            pyxel.line(x + 2, y, x + 3, y + 12, color)


def main() -> None:
    """メイン実行関数"""
    # ゲーム初期化
    pyxel.init(256, 192, title="Dance Master", fps=30)
    game = GameManager()

    # ゲーム更新関数
    def update() -> None:
        """毎フレーム更新"""
        game.update()

    # ゲーム描画関数
    def draw() -> None:
        """毎フレーム描画"""
        game.draw()

    # pyxelのメインループ開始
    pyxel.run(update, draw)


# ゲーム実行
if __name__ == "__main__":
    main()
