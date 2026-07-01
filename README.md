# ソフトウェア工学



//ユースケース図

graph TD
    %% アクターの定義（人型の表現）
    User((一般ユーザー))

    %% ユースケースの定義
    subgraph スマート・TODO アプリケーション領域
        UC_Add[タスクを追加する]
        UC_Validate[入力内容を検証する<br>空文字ガード]
        
        UC_View[タスク一覧を表示する]
        UC_Load[LocalStorageから読込]
        
        UC_Manage[ステータスを変更する]
        UC_DnD[ドラッグ&ドロップ移動]
        
        UC_Edit[タスク内容を編集する]
        UC_Delete[タスクを削除する]
        UC_Clear[完了タスクを一括クリアする]
        
        UC_Save[LocalStorageへ自動保存]
    end

    %% アクターからユースケースへの関連
    User --> UC_Add
    User --> UC_View
    User --> UC_Manage
    User --> UC_Edit
    User --> UC_Delete

    %% include / extend の関係性
    %% タスク追加時は、必ず入力検証を行う (extend)
    UC_Validate -.->|<<extend>>| UC_Add
    
    %% 一覧表示時は、必ずLocalStorageからデータを読み込む (include)
    UC_View -.->|<<include>>| UC_Load

    %% ステータス変更の手段として、ドラッグ&ドロップが含まれる (include)
    UC_Manage -.->|<<include>>| UC_DnD

    %% 一括クリアは削除機能の拡張 (extend)
    UC_Clear -.->|<<extend>>| UC_Delete

    %% データの変更（追加・変更・削除）が起きるユースケースは自動保存を内包する (include)
    UC_Add -.->|<<include>>| UC_Save
    UC_Manage -.->|<<include>>| UC_Save
    UC_Edit -.->|<<include>>| UC_Save
    UC_Delete -.->|<<include>>| UC_Save

    %% スタイリング（見やすさのための調整）
    style User fill:#fff,stroke:#333,stroke-width:2px;
    style UC_Save fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px;
    style UC_Load fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px;







//クラス図

classDiagram
    direction TB

    %% --------------------------------------------------
    %% 1. データモデル層 (Data Model)
    %% --------------------------------------------------
    class Task {
        +id: number
        +title: string
        +status: TaskStatus
        +updateStatus(newStatus: TaskStatus): void
        +updateTitle(newTitle: string): void
    }

    class TaskStatus {
        <<enumeration>>
        TODO (未着手)
        IN_PROGRESS (進行中)
        DONE (完了)
    }

    class LocalStorageManager {
        <<utility>>
        +KEY: string$
        +loadTasks(): Task[]$
        +saveTasks(tasks: Task[]): void$
    }

    %% --------------------------------------------------
    %% 2. UI / コンポーネント層 (React Components)
    %% --------------------------------------------------
    class AppComp {
        +tasks: Task[]
        +handleAddTask(title: string): void
        +handleUpdateStatus(id: number, newStatus: TaskStatus): void
        +handleDeleteTask(id: number): void
        +handleClearDoneTasks(): void
        +render(): JSX.Element
    }

    class TaskFormComp {
        -textInput: string
        +onSubmit(title: string): void
        +render(): JSX.Element
    }

    class TaskBoardComp {
        +render(): JSX.Element
    }

    class TaskColumnComp {
        +status: TaskStatus
        +render(): JSX.Element
    }

    class TaskCardComp {
        +task: Task
        +onDelete(id: number): void
        +onStatusChange(id: number, status: TaskStatus): void
        +render(): JSX.Element
    }

    %% --------------------------------------------------
    %% 3. 関係性の定義 (Relationships & Multiplicity)
    %% --------------------------------------------------
    
    %% TaskとStatusの関係（依存・バインド）
    Task --> "1" TaskStatus : 状態を持つ

    %% Appコンポーネントは複数のTaskデータを「集約（所有）」する
    AppComp "1" o-- "0..*" Task : 管理・保持する

    %% Appコンポーネントと下位コンポーネントの「コンポジション（ライフサイクルが同一）」
    AppComp "1" *-- "1" TaskFormComp : 含む
    AppComp "1" *-- "1" TaskBoardComp : 含む
    
    %% ボードは3つの列（未着手・進行中・完了）で構成される
    TaskBoardComp "1" *-- "3" TaskColumnComp : で構成される
    
    %% 各列には、そのステータスに応じたタスクカードが0個以上配置される
    TaskColumnComp "1" *-- "0..*" TaskCardComp : 内包する
    
    %% タスクカードは表示対象のTaskデータを1つ参照する
    TaskCardComp "1" --> "1" Task : 表示・操作対象

    %% Appコンポーネントは、データの保存・復元のためにUtilityクラスを利用（依存関係）
    AppComp ..> LocalStorageManager : 利用する







//ユースケース

sequenceDiagram
    autonumber
    actor User as 一般ユーザー
    participant UI as 画面 (UI)<br>[TaskForm / TaskCard]
    participant Ctrl as コントローラ<br>[AppComp (React State)]
    participant Model as モデル (DB分脈)<br>[LocalStorageManager]

    %% ---- ユースケース1: タスクの追加（条件分岐あり） ----
    note over User, Model: 【ユースケース1】 新しいタスクの追加
    User->>UI: タスク名を入力して「追加」ボタンを押下
    UI->>Ctrl: handleAddTask(title) を呼び出し
    
    alt タイトルが空文字の場合（バリデーション）
        Ctrl-->>UI: エラー状態を通知（入力を拒否）
        UI-->>User: 画面に入力エラーを表示（何も追加されない）
    else 正しい入力の場合
        Ctrl->>Ctrl: 新しい Task オブジェクトを生成 (status: 未着手)
        Ctrl->>Ctrl: State（tasks配列）に新規タスクを追加
        Ctrl->>Model: saveTasks(updatedTasks) を呼び出し
        Model->>Model: ブラウザのLocalStorageへ書き込み
        Ctrl-->>UI: 最新のタスク一覧を再レンダリング (リロードなし)
        UI-->>User: 「未着手」エリアに新しいタスクが滑らかに表示される
    end

    %% ---- ユースケース2: ドラッグ＆ドロップによるステータス変更 ----
    note over User, Model: 【ユースケース2】 ドラッグ＆ドロップによるステータス変更
    User->>UI: タスクカードをドラッグして「進行中」エリアへドロップ
    UI->>Ctrl: handleUpdateStatus(id, "IN_PROGRESS") を呼び出し
    
    note over Ctrl: 【ループ処理】 該当タスクの探索と更新
    loop tasks配列の全件走査 (Array.map)
        Ctrl->>Ctrl: 一致する id のタスクオブジェクトを探索
    end
    
    Ctrl->>Ctrl: 対象タスクの status を "IN_PROGRESS" に更新
    Ctrl->>Ctrl: 更新後の配列で State を更新
    Ctrl->>Model: saveTasks(updatedTasks) を呼び出し
    Model->>Model: ブラウザのLocalStorageへ上書き保存
    
    Ctrl-->>UI: 最新の状態を通知
    UI-->>User: タスクカードが「進行中」エリアへと滑らかに移動する (0.1秒以内)








//状態遷移図

stateDiagram-v2
    %% 初期状態からの遷移
    [*] --> 登録前 : アプリ起動 / 入力フォーム表示

    %% 状態の定義
    state 登録前 {
        [*] --> 入力中 : ユーザーがテキストを入力
        入力中 --> 入力中 : 編集・クリア
    }

    state 作成完了_永続化領域 {
        state 未着手 {
            [*] --> 初期表示
        }
        state 進行中
        state 完了
    }

    %% 状態遷移とトリガーの定義
    登録前 --> 未着手 : 追加ボタン押下 / handleAddTask()<br>【LocalStorageへ保存】
    
    %% エリア間のドラッグ＆ドロップ（状態遷移）
    未着手 --> 進行中 : 「進行中」へドラッグ / handleUpdateStatus()<br>【LocalStorageへ保存】
    進行中 --> 未着手 : 「未着手」へドラッグ / handleUpdateStatus()<br>【LocalStorageへ保存】
    
    進行中 --> 完了 : 「完了」へドラッグ / handleUpdateStatus()<br>【LocalStorageへ保存】
    完了 --> 進行中 : 「進行中」へドラッグ / handleUpdateStatus()<br>【LocalStorageへ保存】
    
    未着手 --> 完了 : 「完了」へドラッグ / handleUpdateStatus()<br>【LocalStorageへ保存】
    完了 --> 未着手 : 「未着手」へドラッグ / handleUpdateStatus()<br>【LocalStorageへ保存】

    %% インライン編集（自己遷移）
    未着手 --> 未着手 : 内容を編集 / handleUpdateTitle()<br>【LocalStorageへ保存】
    進行中 --> 進行中 : 内容を編集 / handleUpdateTitle()<br>【LocalStorageへ保存】

    %% 終了状態（削除）への遷移
    未着手 --> [*] : 削除ボタン押下 / handleDeleteTask()<br>【LocalStorageから消去】
    進行中 --> [*] : 削除ボタン押下 / handleDeleteTask()<br>【LocalStorageから消去】
    完了 --> [*] : 削除ボタン押下 or 一括クリア / handleDeleteTask()<br>【LocalStorageから消去】



## 現在動く機能
- [x] タスク追加機能 ＆ 空文字バリデーション
- [x] ドラッグ＆ドロップによるステータス変更
- [x] タスク内容の編集機能
## 起動方法

javascript app.jsx
## 動作確認
1. ブラウザで (http://localhost:5173/) を開く
2. タスクを入力して登録
3. 一覧に表示されることを確認
