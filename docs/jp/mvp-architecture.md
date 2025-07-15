# MVP Architecture（Model-View-Presenter）

## 概要
MVPアーキテクチャ（Model-View-Presenter）は、アプリケーションのロジックとプレゼンテーション層を分離するUIアーキテクチャパターンです。MVCパターンの改良版として、ViewとModelの完全な分離を実現し、特にテスタビリティと保守性を重視したパターンです。

## 基本構造

```
┌─────────────────────────────────────────────────────┐
│                    View                             │
│                 (UI Layer)                          │
├─────────────────────────────────────────────────────┤
│                      ↕                              │
│                 IView Interface                     │
│                      ↕                              │
├─────────────────────────────────────────────────────┤
│                  Presenter                          │
│              (Presentation Logic)                   │
├─────────────────────────────────────────────────────┤
│                      ↕                              │
│                 IModel Interface                    │
│                      ↕                              │
├─────────────────────────────────────────────────────┤
│                    Model                            │
│              (Business Logic & Data)                │
└─────────────────────────────────────────────────────┘
```

## 核心コンポーネント

### 1. Model（モデル）
```go
package main

package mvp

    // context
    // sync
    // time
)

// User ユーザーデータモデル
type User struct {
    ID        *int       `json:// id,omitempty`
    Name      string     `json:// name`
    Email     string     `json:// email`
    CreatedAt *time.Time `json:// created_at,omitempty`
    IsActive  bool       `json:// is_active`
}

// NewUser ユーザー作成
func NewUser(name, email string) *User {
    now := time.Now()
    return &User{
        Name:      name,
        Email:     email,
        CreatedAt: &now,
        IsActive:  true,
    }
}

// UserModelListener ユーザーモデルリスナーインターフェース
type UserModelListener interface {
    OnUserLoaded(users []*User)
    OnUserSaved(user *User)
    OnError(error string)
}

// UserModel ユーザーモデルインターフェース
type UserModel interface {
    AddListener(listener UserModelListener)
    RemoveListener(listener UserModelListener)
    LoadUsers(ctx context.Context) error
    SaveUser(ctx context.Context, user *User) error
    DeleteUser(ctx context.Context, userID int) error
}

// UserModelImpl ユーザーモデル実装
type UserModelImpl struct {
    userRepository UserRepository
    listeners      []UserModelListener
    users          []*User
    mu             sync.RWMutex
}
        // TODO: implement
        
        async func (receiver) save_user(user: User) {
        // ユーザー保存
        // TODO: implement
        
        async func (receiver) delete_user(user_id int) {
        // ユーザー削除
        // TODO: implement

type UserModel struct {
    // ユーザーモデル実装
    
    func (receiver) __init__(user_repository: 'UserRepository') {
        receiver.user_repository = user_repository
        receiver.listeners []IUserModelListener = []
        receiver.users []User = []
        
    func (receiver) add_listener(listener: IUserModelListener) {
        // リスナー追加
        if listener not in receiver.listeners:
            receiver.listeners.append(listener)
            
    func (receiver) remove_listener(listener: IUserModelListener) {
        // リスナー削除
        if listener in receiver.listeners:
            receiver.listeners.remove(listener)
            
    async func (receiver) load_users() {
        // ユーザー一覧読み込み
        try:
            receiver.users = await receiver.user_repository.get_all_users()
            receiver._notify_users_loaded(receiver.users)
        except Exception as e:
            receiver._notify_error(f// ユーザー読み込みエラー: {str(e)})
            
    async func (receiver) save_user(user: User) {
        // ユーザー保存
        try:
            # バリデーション
            if not user.name.strip():
                raise ValueError(// ユーザー名は必須です)
            if not user.email.strip():
                raise ValueError(// メールアドレスは必須です)
                
            saved_user = await receiver.user_repository.save_user(user)
            receiver._notify_user_saved(saved_user)
            
        except Exception as e:
            receiver._notify_error(f// ユーザー保存エラー: {str(e)})
            
    async func (receiver) delete_user(user_id int) {
        // ユーザー削除
        try:
            await receiver.user_repository.delete_user(user_id)
            # ローカルリストからも削除
            receiver.users = [u for u in receiver.users if u.id != user_id]
            receiver._notify_users_loaded(receiver.users)
            
        except Exception as e:
            receiver._notify_error(f// ユーザー削除エラー: {str(e)})
            
    func (receiver) _notify_users_loaded(users []User) {
        // ユーザーロード通知
        for listener in receiver.listeners:
            listener.on_user_loaded(users)
            
    func (receiver) _notify_user_saved(user: User) {
        // ユーザー保存通知
        for listener in receiver.listeners:
            listener.on_user_saved(user)
            
    func (receiver) _notify_error(error string) {
        // エラー通知
        for listener in receiver.listeners:
            listener.on_error(error)
```

### 2. View（ビュー）
```go
package main

type IUserView struct {
    // ユーザービューインターフェース
    
        func (receiver) show_users(users []User) {
        // ユーザー一覧表示
        // TODO: implement
        
        func (receiver) show_loading() {
        // ローディング表示
        // TODO: implement
        
        func (receiver) hide_loading() {
        // ローディング非表示
        // TODO: implement
        
        func (receiver) show_error(message string) {
        // エラーメッセージ表示
        // TODO: implement
        
        func (receiver) show_success(message string) {
        // 成功メッセージ表示
        // TODO: implement
        
        func (receiver) clear_form() {
        // フォームクリア
        // TODO: implement
        
        func (receiver) get_user_form_data() {
        // フォームデータ取得
        // TODO: implement
        
        func (receiver) set_presenter(presenter: 'IUserPresenter') {
        // プレゼンター設定
        // TODO: implement

# Web実装例（Flask）

type WebUserView struct {
    // Webベースのユーザービュー
    
    func (receiver) __init__(app: Flask) {
        receiver.app = app
        receiver.presenter *'IUserPresenter' = nil
        receiver.users_data []User = []
        receiver.is_loading = false
        receiver.setup_routes()
        
    func (receiver) set_presenter(presenter: 'IUserPresenter') {
        // プレゼンター設定
        receiver.presenter = presenter
        
    func (receiver) setup_routes() {
        // ルート設定
                def users_page():
            if receiver.presenter:
                asyncio.create_task(receiver.presenter.load_users())
            return render_template('users.html', 
                                 users=receiver.users_data, 
                                 loading=receiver.is_loading)
                                 
                def new_user():
            if request.method == 'POST':
                if receiver.presenter:
                    asyncio.create_task(receiver.presenter.save_user())
                return redirect(url_for('users_page'))
            return render_template('user_form.html')
            
                func delete_user(user_id) {
            if receiver.presenter:
                asyncio.create_task(receiver.presenter.delete_user(user_id))
            return redirect(url_for('users_page'))
            
                def api_users():
            return jsonify([{
                'id': u.id,
                'name': u.name,
                'email': u.email,
                'created_at': u.created_at.isoformat() if u.created_at else nil,
                'is_active': u.is_active
            } for u in receiver.users_data])
            
    func (receiver) show_users(users []User) {
        // ユーザー一覧表示
        receiver.users_data = users
        receiver.hide_loading()
        
    func (receiver) show_loading() {
        // ローディング表示
        receiver.is_loading = true
        
    func (receiver) hide_loading() {
        // ローディング非表示
        receiver.is_loading = false
        
    func (receiver) show_error(message string) {
        // エラーメッセージ表示
        flash(message, 'error')
        receiver.hide_loading()
        
    func (receiver) show_success(message string) {
        // 成功メッセージ表示
        flash(message, 'success')
        receiver.hide_loading()
        
    func (receiver) clear_form() {
        // フォームクリア
        # Webの場合はリダイレクトで処理
        // TODO: implement
        
    func (receiver) get_user_form_data() {
        // フォームデータ取得
        return User(
            name=request.form.get('name', ''),
            email=request.form.get('email', ''),
            is_active=request.form.get('is_active') == 'on'
        )

# GUI実装例（tkinter）

type TkinterUserView struct {
    // tkinterベースのユーザービュー
    
    func (receiver) __init__() {
        receiver.root = tk.Tk()
        receiver.root.title(// ユーザー管理)
        receiver.root.geometry(// 800x600)
        
        receiver.presenter *'IUserPresenter' = nil
        receiver.users_data []User = []
        
        receiver.setup_ui()
        
    func (receiver) set_presenter(presenter: 'IUserPresenter') {
        // プレゼンター設定
        receiver.presenter = presenter
        
    func (receiver) setup_ui() {
        // UI設定
        # メインフレーム
        main_frame = ttk.Frame(receiver.root)
        main_frame.pack(fill=tk.BOTH, expand=true, padx=10, pady=10)
        
        # ボタンフレーム
        button_frame = ttk.Frame(main_frame)
        button_frame.pack(fill=tk.X, pady=(0, 10))
        
        ttk.Button(button_frame, text=// 読み込み, 
                  command=receiver._load_users).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(button_frame, text=// 新規作成, 
                  command=receiver._create_user).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(button_frame, text=// 削除, 
                  command=receiver._delete_selected_user).pack(side=tk.LEFT)
                  
        # ユーザーリスト
        receiver.tree = ttk.Treeview(main_frame, columns=('ID', 'Name', 'Email', 'Active'), show='headings')
        receiver.tree.heading('#1', text='ID')
        receiver.tree.heading('#2', text='名前')
        receiver.tree.heading('#3', text='メール')
        receiver.tree.heading('#4', text='アクティブ')
        
        # スクロールバー
        scrollbar = ttk.Scrollbar(main_frame, orient=tk.VERTICAL, command=receiver.tree.yview)
        receiver.tree.configure(yscrollcommand=scrollbar.set)
        
        receiver.tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=true)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # プログレスバー
        receiver.progress = ttk.Progressbar(main_frame, mode='indeterminate')
        
    func (receiver) show_users(users []User) {
        // ユーザー一覧表示
        receiver.users_data = users
        
        # ツリーをクリア
        for item in receiver.tree.get_children():
            receiver.tree.delete(item)
            
        # ユーザーを追加
        for user in users:
            receiver.tree.insert('', tk.END, values=(
                user.id,
                user.name,
                user.email,
                'はい' if user.is_active else 'いいえ'
            ))
            
        receiver.hide_loading()
        
    func (receiver) show_loading() {
        // ローディング表示
        receiver.progress.pack(fill=tk.X, pady=5)
        receiver.progress.start()
        
    func (receiver) hide_loading() {
        // ローディング非表示
        receiver.progress.stop()
        receiver.progress.pack_forget()
        
    func (receiver) show_error(message string) {
        // エラーメッセージ表示
        messagebox.showerror(// エラー, message)
        receiver.hide_loading()
        
    func (receiver) show_success(message string) {
        // 成功メッセージ表示
        messagebox.showinfo(// 成功, message)
        receiver.hide_loading()
        
    func (receiver) clear_form() {
        // フォームクリア
        # 選択をクリア
        for item in receiver.tree.selection():
            receiver.tree.selection_remove(item)
            
    func (receiver) get_user_form_data() {
        // フォームデータ取得
        # ダイアログから入力を取得
        name = simpledialog.askstring(// ユーザー作成, // 名前を入力してください:)
        if not name:
            raise ValueError(// 名前が入力されていません)
            
        email = simpledialog.askstring(// ユーザー作成, // メールアドレスを入力してください:)
        if not email:
            raise ValueError(// メールアドレスが入力されていません)
            
        return User(name=name, email=email)
        
    func (receiver) _load_users() {
        // ユーザー読み込み
        if receiver.presenter:
            asyncio.create_task(receiver.presenter.load_users())
            
    func (receiver) _create_user() {
        // ユーザー作成
        if receiver.presenter:
            asyncio.create_task(receiver.presenter.save_user())
            
    func (receiver) _delete_selected_user() {
        // 選択されたユーザーを削除
        selection = receiver.tree.selection()
        if not selection:
            messagebox.showwarning(// 警告, // 削除するユーザーを選択してください)
            return
            
        item = selection[0]
        user_id = receiver.tree.item(item)['values'][0]
        
        if messagebox.askyesno(// 確認, // 選択されたユーザーを削除しますか？):
            if receiver.presenter:
                asyncio.create_task(receiver.presenter.delete_user(user_id))
                
    func (receiver) run() {
        // アプリケーション開始
        receiver.root.mainloop()
```

### 3. Presenter（プレゼンター）
```go
package main

type IUserPresenter struct {
    // ユーザープレゼンターインターフェース
    
        async func (receiver) load_users() {
        // ユーザー読み込み
        // TODO: implement
        
        async func (receiver) save_user() {
        // ユーザー保存
        // TODO: implement
        
        async func (receiver) delete_user(user_id int) {
        // ユーザー削除
        // TODO: implement

type UserPresenter struct {
    // ユーザープレゼンター実装
    
    func (receiver) __init__(view: IUserView, model: IUserModel) {
        receiver.view = view
        receiver.model = model
        
        # ビューにプレゼンターを設定
        receiver.view.set_presenter(self)
        
        # モデルのリスナーとして登録
        receiver.model.add_listener(self)
        
    async func (receiver) load_users() {
        // ユーザー読み込み
        receiver.view.show_loading()
        await receiver.model.load_users()
        
    async func (receiver) save_user() {
        // ユーザー保存
        try:
            user_data = receiver.view.get_user_form_data()
            receiver.view.show_loading()
            await receiver.model.save_user(user_data)
            
        except ValueError as e:
            receiver.view.show_error(str(e))
        except Exception as e:
            receiver.view.show_error(f// 予期しないエラー: {str(e)})
            
    async func (receiver) delete_user(user_id int) {
        // ユーザー削除
        receiver.view.show_loading()
        await receiver.model.delete_user(user_id)
        
    # IUserModelListener の実装
    func (receiver) on_user_loaded(users []User) {
        // ユーザーロード完了通知
        receiver.view.show_users(users)
        
    func (receiver) on_user_saved(user: User) {
        // ユーザー保存完了通知
        receiver.view.show_success(// ユーザーを保存しました)
        receiver.view.clear_form()
        # 一覧を再読み込み
        asyncio.create_task(receiver.model.load_users())
        
    func (receiver) on_error(error string) {
        // エラー通知
        receiver.view.show_error(error)

# 高度なプレゼンター実装例
type AdvancedUserPresenter struct {
    // 高度なユーザープレゼンター
    
    def __init__(self, view: IUserView, model: IUserModel, 
                 validator: 'UserValidator', logger: 'Logger'):
        super().__init__(view, model)
        receiver.validator = validator
        receiver.logger = logger
        receiver.operation_history []str = []
        
    async func (receiver) save_user() {
        // バリデーション付きユーザー保存
        try:
            user_data = receiver.view.get_user_form_data()
            
            # バリデーション
            validation_errors = receiver.validator.validate_user(user_data)
            if validation_errors:
                error_message = // \\n.join(validation_errors)
                receiver.view.show_error(f// 入力エラー:\\n{error_message})
                return
                
            receiver.view.show_loading()
            
            # 操作ログ
            receiver.logger.info(f// User save attempt: {user_data.email})
            receiver.operation_history.append(f// 保存試行: {user_data.name})
            
            await receiver.model.save_user(user_data)
            
        except ValueError as e:
            receiver.logger.warning(f// Validation error: {str(e)})
            receiver.view.show_error(str(e))
        except Exception as e:
            receiver.logger.error(f// Unexpected error during user save: {str(e)})
            receiver.view.show_error(f// 予期しないエラー: {str(e)})
            
    async func (receiver) delete_user(user_id int) {
        // 確認付きユーザー削除
        try:
            receiver.view.show_loading()
            
            # 操作ログ
            receiver.logger.info(f// User delete attempt: ID {user_id})
            receiver.operation_history.append(f// 削除試行: ID {user_id})
            
            await receiver.model.delete_user(user_id)
            
        except Exception as e:
            receiver.logger.error(f// Error during user delete: {str(e)})
            receiver.view.show_error(f// 削除エラー: {str(e)})
            
    func (receiver) get_operation_history() {
        // 操作履歴取得
        return receiver.operation_history.copy()

# バリデーター
type UserValidator struct {
    // ユーザーバリデーター
    
    func (receiver) validate_user(user: User) {
        // ユーザーバリデーション
        errors = []
        
        # 名前チェック
        if not user.name or not user.name.strip():
            errors.append(// 名前は必須です)
        elif len(user.name.strip()) < 2:
            errors.append(// 名前は2文字以上で入力してください)
            
        # メールアドレスチェック
        if not user.email or not user.email.strip():
            errors.append(// メールアドレスは必須です)
        elif not receiver._is_valid_email(user.email):
            errors.append(// 有効なメールアドレスを入力してください)
            
        return errors
        
    func (receiver) _is_valid_email(email string) {
        // メールアドレス形式チェック
                pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not nil
```

## MVPパターンの特徴

### 1. 完全な分離
```go
package main

# ビューはモデルを直接知らない
type OrderView struct {
    func (receiver) __init__() {
        receiver.presenter *'IOrderPresenter' = nil
        # モデルへの直接的な参照はない
        
    func (receiver) set_presenter(presenter: 'IOrderPresenter') {
        receiver.presenter = presenter
        
    func (receiver) on_submit_button_clicked() {
        # プレゼンターを通してのみ操作
        if receiver.presenter:
            receiver.presenter.submit_order()

# モデルもビューを直接知らない
type OrderModel struct {
    func (receiver) __init__() {
        receiver.listeners []IOrderModelListener = []
        # ビューへの直接的な参照はない
        
    func (receiver) process_order(order: Order) {
        # ビジネスロジック実行
        result = receiver._validate_and_process(order)
        
        # リスナー（プレゼンター）に通知
        if result.success:
            receiver._notify_order_processed(result.order)
        else:
            receiver._notify_error(result.error_message)
```

### 2. テスタビリティ
```go
package main


type TestUserPresenter struct {
    // ユーザープレゼンターのテスト
    
    func (receiver) setUp() {
        receiver.mock_view = Mock(spec=IUserView)
        receiver.mock_model = Mock(spec=IUserModel)
        receiver.presenter = UserPresenter(receiver.mock_view, receiver.mock_model)
        
    async func (receiver) test_load_users_shows_loading() {
        // ユーザー読み込み時にローディング表示
        # Arrange
        receiver.mock_model.load_users = AsyncMock()
        
        # Act
        await receiver.presenter.load_users()
        
        # Assert
        receiver.mock_view.show_loading.assert_called_once()
        receiver.mock_model.load_users.assert_called_once()
        
    async func (receiver) test_save_user_with_valid_data() {
        // 有効なデータでのユーザー保存
        # Arrange
        test_user = User(name=// Test User, email="test        receiver.mock_view.get_user_form_data.return_value = test_user
        receiver.mock_model.save_user = AsyncMock()
        
        # Act
        await receiver.presenter.save_user()
        
        # Assert
        receiver.mock_view.show_loading.assert_called_once()
        receiver.mock_model.save_user.assert_called_once_with(test_user)
        
    async func (receiver) test_save_user_with_invalid_data() {
        // 無効なデータでのユーザー保存
        # Arrange
        receiver.mock_view.get_user_form_data.side_effect = ValueError(// 名前は必須です)
        
        # Act
        await receiver.presenter.save_user()
        
        # Assert
        receiver.mock_view.show_error.assert_called_once_with(// 名前は必須です)
        receiver.mock_model.save_user.assert_not_called()
        
    func (receiver) test_on_user_loaded_updates_view() {
        // ユーザーロード完了時のビュー更新
        # Arrange
        test_users = [
            User(id=1, name=// User 1, email=// user1            User(id=2, name=User 2// , email=user2        ]
        
        # Act
        receiver.presenter.on_user_loaded(test_users)
        
        # Assert
        receiver.mock_view.show_users.assert_called_once_with(test_users)
        
    func (receiver) test_on_error_shows_error_message() {
        // エラー時のエラーメッセージ表示
        # Arrange
        error_message = // データベース接続エラー
        
        # Act
        receiver.presenter.on_error(error_message)
        
        # Assert
        receiver.mock_view.show_error.assert_called_once_with(error_message)

type TestUserModel struct {
    // ユーザーモデルのテスト
    
    func (receiver) setUp() {
        receiver.mock_repository = Mock()
        receiver.mock_listener = Mock(spec=IUserModelListener)
        receiver.model = UserModel(receiver.mock_repository)
        receiver.model.add_listener(receiver.mock_listener)
        
    async func (receiver) test_load_users_success() {
        // ユーザー読み込み成功
        # Arrange
        test_users = [User(id=1, name=// Test, email="test        receiver.mock_repository.get_all_users = AsyncMock(return_value=test_users)
        
        # Act
        await receiver.model.load_users()
        
        # Assert
        receiver.mock_listener.on_user_loaded.assert_called_once_with(test_users)
        
    async func (receiver) test_load_users_failure() {
        // ユーザー読み込み失敗
        # Arrange
        receiver.mock_repository.get_all_users = AsyncMock(
            side_effect=Exception(// DB Error)
        )
        
        # Act
        await receiver.model.load_users()
        
        # Assert
        receiver.mock_listener.on_error.assert_called_once()
        
    async func (receiver) test_save_user_validation_error() {
        // ユーザー保存時のバリデーションエラー
        # Arrange
        invalid_user = User(name=// ", email=test        
        # Act
        await receiver.model.save_user(invalid_user)
        
        # Assert
        receiver.mock_listener.on_error.assert_called_once()
        receiver.mock_repository.save_user.assert_not_called()
```

## 実装パターン

### 1. 非同期MVP
```go
package main


type AsyncUserPresenter struct {
    // 非同期ユーザープレゼンター
    
    func (receiver) __init__(view: IUserView, model: IUserModel) {
        receiver.view = view
        receiver.model = model
        receiver.current_tasks: set = set()
        
        receiver.view.set_presenter(self)
        receiver.model.add_listener(self)
        
    async func (receiver) load_users() {
        // 非同期ユーザー読み込み
        await receiver._execute_async_operation(receiver.model.load_users())
        
    async func (receiver) save_user() {
        // 非同期ユーザー保存
        try:
            user_data = receiver.view.get_user_form_data()
            await receiver._execute_async_operation(receiver.model.save_user(user_data))
        except ValueError as e:
            receiver.view.show_error(str(e))
            
    async func (receiver) _execute_async_operation(operation: Awaitable) {
        // 非同期操作の実行管理
        receiver.view.show_loading()
        
        task = asyncio.create_task(operation)
        receiver.current_tasks.add(task)
        
        try:
            await task
        finally:
            receiver.current_tasks.discard(task)
            
    async func (receiver) cancel_all_operations() {
        // 全操作のキャンセル
        for task in receiver.current_tasks:
            task.cancel()
            
        # 完了を待つ
        if receiver.current_tasks:
            await asyncio.gather(*receiver.current_tasks, return_exceptions=true)
            
        receiver.view.hide_loading()

type ThreadSafeUserPresenter struct {
    // スレッドセーフなユーザープレゼンター
    
    func (receiver) __init__(view: IUserView, model: IUserModel) {
        super().__init__(view, model)
        receiver._lock = asyncio.Lock()
        
    async func (receiver) load_users() {
        // スレッドセーフなユーザー読み込み
        async with receiver._lock:
            await super().load_users()
            
    async func (receiver) save_user() {
        // スレッドセーフなユーザー保存
        async with receiver._lock:
            await super().save_user()
```

### 2. 状態管理
```go
package main


type ViewState struct {
    // ビューの状態
    IDLE = // idle
    LOADING = // loading
    ERROR = // error
    SUCCESS = // success

type UserViewState struct {
    // ユーザービューの状態
    state: ViewState = ViewState.IDLE
    users []User = nil
    error_message string = ""
    success_message string = ""
    form_data map[str]Any = nil
    
    func (receiver) __post_init__() {
        if receiver.users is nil:
            receiver.users = []
        if receiver.form_data is nil:
            receiver.form_data = {}

type StatefulUserPresenter struct {
    // 状態管理を行うユーザープレゼンター
    
    func (receiver) __init__(view: IUserView, model: IUserModel) {
        receiver.view = view
        receiver.model = model
        receiver.state = UserViewState()
        
        receiver.view.set_presenter(self)
        receiver.model.add_listener(self)
        
    async func (receiver) load_users() {
        // 状態管理付きユーザー読み込み
        receiver._update_state(ViewState.LOADING)
        await receiver.model.load_users()
        
    async func (receiver) save_user() {
        // 状態管理付きユーザー保存
        try:
            user_data = receiver.view.get_user_form_data()
            receiver.state.form_data = {
                'name': user_data.name,
                'email': user_data.email
            }
            
            receiver._update_state(ViewState.LOADING)
            await receiver.model.save_user(user_data)
            
        except ValueError as e:
            receiver._update_state(ViewState.ERROR, error_message=str(e))
            
    func (receiver) _update_state(new_state: ViewState, **kwargs) {
        // 状態更新
        receiver.state.state = new_state
        
        for key, value in kwargs.items():
            if hasattr(receiver.state, key):
                setattr(receiver.state, key, value)
                
        receiver._render_view()
        
    func (receiver) _render_view() {
        // 状態に基づくビューレンダリング
        if receiver.state.state == ViewState.LOADING:
            receiver.view.show_loading()
        elif receiver.state.state == ViewState.ERROR:
            receiver.view.hide_loading()
            receiver.view.show_error(receiver.state.error_message)
        elif receiver.state.state == ViewState.SUCCESS:
            receiver.view.hide_loading()
            receiver.view.show_success(receiver.state.success_message)
            receiver.view.show_users(receiver.state.users)
        elif receiver.state.state == ViewState.IDLE:
            receiver.view.hide_loading()
            receiver.view.show_users(receiver.state.users)
            
    # IUserModelListener の実装
    func (receiver) on_user_loaded(users []User) {
        // ユーザーロード完了
        receiver._update_state(ViewState.IDLE, users=users)
        
    func (receiver) on_user_saved(user: User) {
        // ユーザー保存完了
        receiver._update_state(ViewState.SUCCESS, 
                         success_message=// ユーザーを保存しました)
        # フォームデータクリア
        receiver.state.form_data = {}
        # 一覧再読み込み
        asyncio.create_task(receiver.model.load_users())
        
    func (receiver) on_error(error string) {
        // エラー発生
        receiver._update_state(ViewState.ERROR, error_message=error)
```

## メリット

### テスタビリティ
- **単体テスト**: 各コンポーネントを独立してテスト
- **モック化**: インターフェースによる簡単なモック作成
- **プレゼンターテスト**: ビジネスロジックの詳細テスト

### 保守性
- **関心の分離**: UI、ビジネスロジック、データアクセスの明確な分離
- **疎結合**: 変更の影響範囲を限定
- **再利用性**: プレゼンターとモデルの再利用

### 拡張性
- **ビュー交換**: 異なるUIフレームワークでの同一ロジック利用
- **プラットフォーム対応**: Web、デスクトップ、モバイルでの共通実装
- **機能追加**: 新機能の段階的追加

## デメリット

### 複雑性
- **コード量**: 単純な機能でも多くのクラス・インターフェース
- **学習コスト**: パターンの理解と適切な実装
- **オーバーエンジニアリング**: 小規模プロジェクトでの過度な設計

### パフォーマンス
- **間接参照**: インターフェース経由のオーバーヘッド
- **メモリ使用量**: 多数のオブジェクト生成
- **通信コスト**: コンポーネント間の頻繁な通信

## 適用場面

### 適している場合
- **複雑なUI**: 複雑なユーザーインターフェース
- **テスト重視**: 高いテストカバレッジが必要
- **マルチプラットフォーム**: 複数のUI実装
- **チーム開発**: UI開発者とロジック開発者の分離

### 適していない場合
- **シンプルなUI**: 基本的なCRUD操作のみ
- **小規模プロジェクト**: 実装コストがメリットを上回る
- **プロトタイプ**: 迅速な開発が優先される
- **静的コンテンツ**: インタラクションが少ない

## 関連パターン
- **MVC**: MVPの元となったパターン
- **MVVM**: データバインディングを活用したパターン
- **Observer Pattern**: モデル-プレゼンター間の通信
- **Command Pattern**: ユーザーアクションの実装
- **Strategy Pattern**: 異なるビジネスロジックの実装

## 参考資料
- [MVP Pattern - Martin Fowler](https://martinfowler.com/eaaDev/ModelViewPresenter.html)
- [GUI Architectures - Martin Fowler](https://martinfowler.com/eaaDev/uiArchs.html)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
- [MVP vs MVC - Microsoft](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff649571(v=pandp.10))
