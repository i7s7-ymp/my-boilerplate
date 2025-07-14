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
```python
from abc import ABC, abstractmethod
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime

@dataclass
class User:
    """ユーザーデータモデル"""
    id: Optional[int] = None
    name: str = ""
    email: str = ""
    created_at: Optional[datetime] = None
    is_active: bool = True

class IUserModelListener(ABC):
    """ユーザーモデルリスナーインターフェース"""
    
    @abstractmethod
    def on_user_loaded(self, users: List[User]) -> None:
        """ユーザーロード完了通知"""
        pass
        
    @abstractmethod
    def on_user_saved(self, user: User) -> None:
        """ユーザー保存完了通知"""
        pass
        
    @abstractmethod
    def on_error(self, error: str) -> None:
        """エラー通知"""
        pass

class IUserModel(ABC):
    """ユーザーモデルインターフェース"""
    
    @abstractmethod
    def add_listener(self, listener: IUserModelListener) -> None:
        """リスナー追加"""
        pass
        
    @abstractmethod
    def remove_listener(self, listener: IUserModelListener) -> None:
        """リスナー削除"""
        pass
        
    @abstractmethod
    async def load_users(self) -> None:
        """ユーザー一覧読み込み"""
        pass
        
    @abstractmethod
    async def save_user(self, user: User) -> None:
        """ユーザー保存"""
        pass
        
    @abstractmethod
    async def delete_user(self, user_id: int) -> None:
        """ユーザー削除"""
        pass

class UserModel(IUserModel):
    """ユーザーモデル実装"""
    
    def __init__(self, user_repository: 'UserRepository'):
        self.user_repository = user_repository
        self.listeners: List[IUserModelListener] = []
        self.users: List[User] = []
        
    def add_listener(self, listener: IUserModelListener) -> None:
        """リスナー追加"""
        if listener not in self.listeners:
            self.listeners.append(listener)
            
    def remove_listener(self, listener: IUserModelListener) -> None:
        """リスナー削除"""
        if listener in self.listeners:
            self.listeners.remove(listener)
            
    async def load_users(self) -> None:
        """ユーザー一覧読み込み"""
        try:
            self.users = await self.user_repository.get_all_users()
            self._notify_users_loaded(self.users)
        except Exception as e:
            self._notify_error(f"ユーザー読み込みエラー: {str(e)}")
            
    async def save_user(self, user: User) -> None:
        """ユーザー保存"""
        try:
            # バリデーション
            if not user.name.strip():
                raise ValueError("ユーザー名は必須です")
            if not user.email.strip():
                raise ValueError("メールアドレスは必須です")
                
            saved_user = await self.user_repository.save_user(user)
            self._notify_user_saved(saved_user)
            
        except Exception as e:
            self._notify_error(f"ユーザー保存エラー: {str(e)}")
            
    async def delete_user(self, user_id: int) -> None:
        """ユーザー削除"""
        try:
            await self.user_repository.delete_user(user_id)
            # ローカルリストからも削除
            self.users = [u for u in self.users if u.id != user_id]
            self._notify_users_loaded(self.users)
            
        except Exception as e:
            self._notify_error(f"ユーザー削除エラー: {str(e)}")
            
    def _notify_users_loaded(self, users: List[User]) -> None:
        """ユーザーロード通知"""
        for listener in self.listeners:
            listener.on_user_loaded(users)
            
    def _notify_user_saved(self, user: User) -> None:
        """ユーザー保存通知"""
        for listener in self.listeners:
            listener.on_user_saved(user)
            
    def _notify_error(self, error: str) -> None:
        """エラー通知"""
        for listener in self.listeners:
            listener.on_error(error)
```

### 2. View（ビュー）
```python
class IUserView(ABC):
    """ユーザービューインターフェース"""
    
    @abstractmethod
    def show_users(self, users: List[User]) -> None:
        """ユーザー一覧表示"""
        pass
        
    @abstractmethod
    def show_loading(self) -> None:
        """ローディング表示"""
        pass
        
    @abstractmethod
    def hide_loading(self) -> None:
        """ローディング非表示"""
        pass
        
    @abstractmethod
    def show_error(self, message: str) -> None:
        """エラーメッセージ表示"""
        pass
        
    @abstractmethod
    def show_success(self, message: str) -> None:
        """成功メッセージ表示"""
        pass
        
    @abstractmethod
    def clear_form(self) -> None:
        """フォームクリア"""
        pass
        
    @abstractmethod
    def get_user_form_data(self) -> User:
        """フォームデータ取得"""
        pass
        
    @abstractmethod
    def set_presenter(self, presenter: 'IUserPresenter') -> None:
        """プレゼンター設定"""
        pass

# Web実装例（Flask）
from flask import Flask, render_template, request, redirect, url_for, flash, jsonify

class WebUserView(IUserView):
    """Webベースのユーザービュー"""
    
    def __init__(self, app: Flask):
        self.app = app
        self.presenter: Optional['IUserPresenter'] = None
        self.users_data: List[User] = []
        self.is_loading = False
        self.setup_routes()
        
    def set_presenter(self, presenter: 'IUserPresenter') -> None:
        """プレゼンター設定"""
        self.presenter = presenter
        
    def setup_routes(self):
        """ルート設定"""
        @self.app.route('/users')
        def users_page():
            if self.presenter:
                asyncio.create_task(self.presenter.load_users())
            return render_template('users.html', 
                                 users=self.users_data, 
                                 loading=self.is_loading)
                                 
        @self.app.route('/users/new', methods=['GET', 'POST'])
        def new_user():
            if request.method == 'POST':
                if self.presenter:
                    asyncio.create_task(self.presenter.save_user())
                return redirect(url_for('users_page'))
            return render_template('user_form.html')
            
        @self.app.route('/users/<int:user_id>/delete', methods=['POST'])
        def delete_user(user_id):
            if self.presenter:
                asyncio.create_task(self.presenter.delete_user(user_id))
            return redirect(url_for('users_page'))
            
        @self.app.route('/api/users')
        def api_users():
            return jsonify([{
                'id': u.id,
                'name': u.name,
                'email': u.email,
                'created_at': u.created_at.isoformat() if u.created_at else None,
                'is_active': u.is_active
            } for u in self.users_data])
            
    def show_users(self, users: List[User]) -> None:
        """ユーザー一覧表示"""
        self.users_data = users
        self.hide_loading()
        
    def show_loading(self) -> None:
        """ローディング表示"""
        self.is_loading = True
        
    def hide_loading(self) -> None:
        """ローディング非表示"""
        self.is_loading = False
        
    def show_error(self, message: str) -> None:
        """エラーメッセージ表示"""
        flash(message, 'error')
        self.hide_loading()
        
    def show_success(self, message: str) -> None:
        """成功メッセージ表示"""
        flash(message, 'success')
        self.hide_loading()
        
    def clear_form(self) -> None:
        """フォームクリア"""
        # Webの場合はリダイレクトで処理
        pass
        
    def get_user_form_data(self) -> User:
        """フォームデータ取得"""
        return User(
            name=request.form.get('name', ''),
            email=request.form.get('email', ''),
            is_active=request.form.get('is_active') == 'on'
        )

# GUI実装例（tkinter）
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog

class TkinterUserView(IUserView):
    """tkinterベースのユーザービュー"""
    
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("ユーザー管理")
        self.root.geometry("800x600")
        
        self.presenter: Optional['IUserPresenter'] = None
        self.users_data: List[User] = []
        
        self.setup_ui()
        
    def set_presenter(self, presenter: 'IUserPresenter') -> None:
        """プレゼンター設定"""
        self.presenter = presenter
        
    def setup_ui(self):
        """UI設定"""
        # メインフレーム
        main_frame = ttk.Frame(self.root)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # ボタンフレーム
        button_frame = ttk.Frame(main_frame)
        button_frame.pack(fill=tk.X, pady=(0, 10))
        
        ttk.Button(button_frame, text="読み込み", 
                  command=self._load_users).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(button_frame, text="新規作成", 
                  command=self._create_user).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(button_frame, text="削除", 
                  command=self._delete_selected_user).pack(side=tk.LEFT)
                  
        # ユーザーリスト
        self.tree = ttk.Treeview(main_frame, columns=('ID', 'Name', 'Email', 'Active'), show='headings')
        self.tree.heading('#1', text='ID')
        self.tree.heading('#2', text='名前')
        self.tree.heading('#3', text='メール')
        self.tree.heading('#4', text='アクティブ')
        
        # スクロールバー
        scrollbar = ttk.Scrollbar(main_frame, orient=tk.VERTICAL, command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        
        self.tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # プログレスバー
        self.progress = ttk.Progressbar(main_frame, mode='indeterminate')
        
    def show_users(self, users: List[User]) -> None:
        """ユーザー一覧表示"""
        self.users_data = users
        
        # ツリーをクリア
        for item in self.tree.get_children():
            self.tree.delete(item)
            
        # ユーザーを追加
        for user in users:
            self.tree.insert('', tk.END, values=(
                user.id,
                user.name,
                user.email,
                'はい' if user.is_active else 'いいえ'
            ))
            
        self.hide_loading()
        
    def show_loading(self) -> None:
        """ローディング表示"""
        self.progress.pack(fill=tk.X, pady=5)
        self.progress.start()
        
    def hide_loading(self) -> None:
        """ローディング非表示"""
        self.progress.stop()
        self.progress.pack_forget()
        
    def show_error(self, message: str) -> None:
        """エラーメッセージ表示"""
        messagebox.showerror("エラー", message)
        self.hide_loading()
        
    def show_success(self, message: str) -> None:
        """成功メッセージ表示"""
        messagebox.showinfo("成功", message)
        self.hide_loading()
        
    def clear_form(self) -> None:
        """フォームクリア"""
        # 選択をクリア
        for item in self.tree.selection():
            self.tree.selection_remove(item)
            
    def get_user_form_data(self) -> User:
        """フォームデータ取得"""
        # ダイアログから入力を取得
        name = simpledialog.askstring("ユーザー作成", "名前を入力してください:")
        if not name:
            raise ValueError("名前が入力されていません")
            
        email = simpledialog.askstring("ユーザー作成", "メールアドレスを入力してください:")
        if not email:
            raise ValueError("メールアドレスが入力されていません")
            
        return User(name=name, email=email)
        
    def _load_users(self):
        """ユーザー読み込み"""
        if self.presenter:
            asyncio.create_task(self.presenter.load_users())
            
    def _create_user(self):
        """ユーザー作成"""
        if self.presenter:
            asyncio.create_task(self.presenter.save_user())
            
    def _delete_selected_user(self):
        """選択されたユーザーを削除"""
        selection = self.tree.selection()
        if not selection:
            messagebox.showwarning("警告", "削除するユーザーを選択してください")
            return
            
        item = selection[0]
        user_id = self.tree.item(item)['values'][0]
        
        if messagebox.askyesno("確認", "選択されたユーザーを削除しますか？"):
            if self.presenter:
                asyncio.create_task(self.presenter.delete_user(user_id))
                
    def run(self):
        """アプリケーション開始"""
        self.root.mainloop()
```

### 3. Presenter（プレゼンター）
```python
class IUserPresenter(ABC):
    """ユーザープレゼンターインターフェース"""
    
    @abstractmethod
    async def load_users(self) -> None:
        """ユーザー読み込み"""
        pass
        
    @abstractmethod
    async def save_user(self) -> None:
        """ユーザー保存"""
        pass
        
    @abstractmethod
    async def delete_user(self, user_id: int) -> None:
        """ユーザー削除"""
        pass

class UserPresenter(IUserPresenter, IUserModelListener):
    """ユーザープレゼンター実装"""
    
    def __init__(self, view: IUserView, model: IUserModel):
        self.view = view
        self.model = model
        
        # ビューにプレゼンターを設定
        self.view.set_presenter(self)
        
        # モデルのリスナーとして登録
        self.model.add_listener(self)
        
    async def load_users(self) -> None:
        """ユーザー読み込み"""
        self.view.show_loading()
        await self.model.load_users()
        
    async def save_user(self) -> None:
        """ユーザー保存"""
        try:
            user_data = self.view.get_user_form_data()
            self.view.show_loading()
            await self.model.save_user(user_data)
            
        except ValueError as e:
            self.view.show_error(str(e))
        except Exception as e:
            self.view.show_error(f"予期しないエラー: {str(e)}")
            
    async def delete_user(self, user_id: int) -> None:
        """ユーザー削除"""
        self.view.show_loading()
        await self.model.delete_user(user_id)
        
    # IUserModelListener の実装
    def on_user_loaded(self, users: List[User]) -> None:
        """ユーザーロード完了通知"""
        self.view.show_users(users)
        
    def on_user_saved(self, user: User) -> None:
        """ユーザー保存完了通知"""
        self.view.show_success("ユーザーを保存しました")
        self.view.clear_form()
        # 一覧を再読み込み
        asyncio.create_task(self.model.load_users())
        
    def on_error(self, error: str) -> None:
        """エラー通知"""
        self.view.show_error(error)

# 高度なプレゼンター実装例
class AdvancedUserPresenter(UserPresenter):
    """高度なユーザープレゼンター"""
    
    def __init__(self, view: IUserView, model: IUserModel, 
                 validator: 'UserValidator', logger: 'Logger'):
        super().__init__(view, model)
        self.validator = validator
        self.logger = logger
        self.operation_history: List[str] = []
        
    async def save_user(self) -> None:
        """バリデーション付きユーザー保存"""
        try:
            user_data = self.view.get_user_form_data()
            
            # バリデーション
            validation_errors = self.validator.validate_user(user_data)
            if validation_errors:
                error_message = "\\n".join(validation_errors)
                self.view.show_error(f"入力エラー:\\n{error_message}")
                return
                
            self.view.show_loading()
            
            # 操作ログ
            self.logger.info(f"User save attempt: {user_data.email}")
            self.operation_history.append(f"保存試行: {user_data.name}")
            
            await self.model.save_user(user_data)
            
        except ValueError as e:
            self.logger.warning(f"Validation error: {str(e)}")
            self.view.show_error(str(e))
        except Exception as e:
            self.logger.error(f"Unexpected error during user save: {str(e)}")
            self.view.show_error(f"予期しないエラー: {str(e)}")
            
    async def delete_user(self, user_id: int) -> None:
        """確認付きユーザー削除"""
        try:
            self.view.show_loading()
            
            # 操作ログ
            self.logger.info(f"User delete attempt: ID {user_id}")
            self.operation_history.append(f"削除試行: ID {user_id}")
            
            await self.model.delete_user(user_id)
            
        except Exception as e:
            self.logger.error(f"Error during user delete: {str(e)}")
            self.view.show_error(f"削除エラー: {str(e)}")
            
    def get_operation_history(self) -> List[str]:
        """操作履歴取得"""
        return self.operation_history.copy()

# バリデーター
class UserValidator:
    """ユーザーバリデーター"""
    
    def validate_user(self, user: User) -> List[str]:
        """ユーザーバリデーション"""
        errors = []
        
        # 名前チェック
        if not user.name or not user.name.strip():
            errors.append("名前は必須です")
        elif len(user.name.strip()) < 2:
            errors.append("名前は2文字以上で入力してください")
            
        # メールアドレスチェック
        if not user.email or not user.email.strip():
            errors.append("メールアドレスは必須です")
        elif not self._is_valid_email(user.email):
            errors.append("有効なメールアドレスを入力してください")
            
        return errors
        
    def _is_valid_email(self, email: str) -> bool:
        """メールアドレス形式チェック"""
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None
```

## MVPパターンの特徴

### 1. 完全な分離
```python
# ビューはモデルを直接知らない
class OrderView(IOrderView):
    def __init__(self):
        self.presenter: Optional['IOrderPresenter'] = None
        # モデルへの直接的な参照はない
        
    def set_presenter(self, presenter: 'IOrderPresenter') -> None:
        self.presenter = presenter
        
    def on_submit_button_clicked(self):
        # プレゼンターを通してのみ操作
        if self.presenter:
            self.presenter.submit_order()

# モデルもビューを直接知らない
class OrderModel(IOrderModel):
    def __init__(self):
        self.listeners: List[IOrderModelListener] = []
        # ビューへの直接的な参照はない
        
    def process_order(self, order: Order):
        # ビジネスロジック実行
        result = self._validate_and_process(order)
        
        # リスナー（プレゼンター）に通知
        if result.success:
            self._notify_order_processed(result.order)
        else:
            self._notify_error(result.error_message)
```

### 2. テスタビリティ
```python
import unittest
from unittest.mock import Mock, AsyncMock

class TestUserPresenter(unittest.TestCase):
    """ユーザープレゼンターのテスト"""
    
    def setUp(self):
        self.mock_view = Mock(spec=IUserView)
        self.mock_model = Mock(spec=IUserModel)
        self.presenter = UserPresenter(self.mock_view, self.mock_model)
        
    async def test_load_users_shows_loading(self):
        """ユーザー読み込み時にローディング表示"""
        # Arrange
        self.mock_model.load_users = AsyncMock()
        
        # Act
        await self.presenter.load_users()
        
        # Assert
        self.mock_view.show_loading.assert_called_once()
        self.mock_model.load_users.assert_called_once()
        
    async def test_save_user_with_valid_data(self):
        """有効なデータでのユーザー保存"""
        # Arrange
        test_user = User(name="Test User", email="test@example.com")
        self.mock_view.get_user_form_data.return_value = test_user
        self.mock_model.save_user = AsyncMock()
        
        # Act
        await self.presenter.save_user()
        
        # Assert
        self.mock_view.show_loading.assert_called_once()
        self.mock_model.save_user.assert_called_once_with(test_user)
        
    async def test_save_user_with_invalid_data(self):
        """無効なデータでのユーザー保存"""
        # Arrange
        self.mock_view.get_user_form_data.side_effect = ValueError("名前は必須です")
        
        # Act
        await self.presenter.save_user()
        
        # Assert
        self.mock_view.show_error.assert_called_once_with("名前は必須です")
        self.mock_model.save_user.assert_not_called()
        
    def test_on_user_loaded_updates_view(self):
        """ユーザーロード完了時のビュー更新"""
        # Arrange
        test_users = [
            User(id=1, name="User 1", email="user1@example.com"),
            User(id=2, name="User 2", email="user2@example.com")
        ]
        
        # Act
        self.presenter.on_user_loaded(test_users)
        
        # Assert
        self.mock_view.show_users.assert_called_once_with(test_users)
        
    def test_on_error_shows_error_message(self):
        """エラー時のエラーメッセージ表示"""
        # Arrange
        error_message = "データベース接続エラー"
        
        # Act
        self.presenter.on_error(error_message)
        
        # Assert
        self.mock_view.show_error.assert_called_once_with(error_message)

class TestUserModel(unittest.TestCase):
    """ユーザーモデルのテスト"""
    
    def setUp(self):
        self.mock_repository = Mock()
        self.mock_listener = Mock(spec=IUserModelListener)
        self.model = UserModel(self.mock_repository)
        self.model.add_listener(self.mock_listener)
        
    async def test_load_users_success(self):
        """ユーザー読み込み成功"""
        # Arrange
        test_users = [User(id=1, name="Test", email="test@example.com")]
        self.mock_repository.get_all_users = AsyncMock(return_value=test_users)
        
        # Act
        await self.model.load_users()
        
        # Assert
        self.mock_listener.on_user_loaded.assert_called_once_with(test_users)
        
    async def test_load_users_failure(self):
        """ユーザー読み込み失敗"""
        # Arrange
        self.mock_repository.get_all_users = AsyncMock(
            side_effect=Exception("DB Error")
        )
        
        # Act
        await self.model.load_users()
        
        # Assert
        self.mock_listener.on_error.assert_called_once()
        
    async def test_save_user_validation_error(self):
        """ユーザー保存時のバリデーションエラー"""
        # Arrange
        invalid_user = User(name="", email="test@example.com")
        
        # Act
        await self.model.save_user(invalid_user)
        
        # Assert
        self.mock_listener.on_error.assert_called_once()
        self.mock_repository.save_user.assert_not_called()
```

## 実装パターン

### 1. 非同期MVP
```python
import asyncio
from typing import Awaitable

class AsyncUserPresenter(IUserPresenter, IUserModelListener):
    """非同期ユーザープレゼンター"""
    
    def __init__(self, view: IUserView, model: IUserModel):
        self.view = view
        self.model = model
        self.current_tasks: set = set()
        
        self.view.set_presenter(self)
        self.model.add_listener(self)
        
    async def load_users(self) -> None:
        """非同期ユーザー読み込み"""
        await self._execute_async_operation(self.model.load_users())
        
    async def save_user(self) -> None:
        """非同期ユーザー保存"""
        try:
            user_data = self.view.get_user_form_data()
            await self._execute_async_operation(self.model.save_user(user_data))
        except ValueError as e:
            self.view.show_error(str(e))
            
    async def _execute_async_operation(self, operation: Awaitable) -> None:
        """非同期操作の実行管理"""
        self.view.show_loading()
        
        task = asyncio.create_task(operation)
        self.current_tasks.add(task)
        
        try:
            await task
        finally:
            self.current_tasks.discard(task)
            
    async def cancel_all_operations(self) -> None:
        """全操作のキャンセル"""
        for task in self.current_tasks:
            task.cancel()
            
        # 完了を待つ
        if self.current_tasks:
            await asyncio.gather(*self.current_tasks, return_exceptions=True)
            
        self.view.hide_loading()

class ThreadSafeUserPresenter(UserPresenter):
    """スレッドセーフなユーザープレゼンター"""
    
    def __init__(self, view: IUserView, model: IUserModel):
        super().__init__(view, model)
        self._lock = asyncio.Lock()
        
    async def load_users(self) -> None:
        """スレッドセーフなユーザー読み込み"""
        async with self._lock:
            await super().load_users()
            
    async def save_user(self) -> None:
        """スレッドセーフなユーザー保存"""
        async with self._lock:
            await super().save_user()
```

### 2. 状態管理
```python
from enum import Enum
from dataclasses import dataclass
from typing import Dict, Any

class ViewState(Enum):
    """ビューの状態"""
    IDLE = "idle"
    LOADING = "loading"
    ERROR = "error"
    SUCCESS = "success"

@dataclass
class UserViewState:
    """ユーザービューの状態"""
    state: ViewState = ViewState.IDLE
    users: List[User] = None
    error_message: str = ""
    success_message: str = ""
    form_data: Dict[str, Any] = None
    
    def __post_init__(self):
        if self.users is None:
            self.users = []
        if self.form_data is None:
            self.form_data = {}

class StatefulUserPresenter(IUserPresenter, IUserModelListener):
    """状態管理を行うユーザープレゼンター"""
    
    def __init__(self, view: IUserView, model: IUserModel):
        self.view = view
        self.model = model
        self.state = UserViewState()
        
        self.view.set_presenter(self)
        self.model.add_listener(self)
        
    async def load_users(self) -> None:
        """状態管理付きユーザー読み込み"""
        self._update_state(ViewState.LOADING)
        await self.model.load_users()
        
    async def save_user(self) -> None:
        """状態管理付きユーザー保存"""
        try:
            user_data = self.view.get_user_form_data()
            self.state.form_data = {
                'name': user_data.name,
                'email': user_data.email
            }
            
            self._update_state(ViewState.LOADING)
            await self.model.save_user(user_data)
            
        except ValueError as e:
            self._update_state(ViewState.ERROR, error_message=str(e))
            
    def _update_state(self, new_state: ViewState, **kwargs):
        """状態更新"""
        self.state.state = new_state
        
        for key, value in kwargs.items():
            if hasattr(self.state, key):
                setattr(self.state, key, value)
                
        self._render_view()
        
    def _render_view(self):
        """状態に基づくビューレンダリング"""
        if self.state.state == ViewState.LOADING:
            self.view.show_loading()
        elif self.state.state == ViewState.ERROR:
            self.view.hide_loading()
            self.view.show_error(self.state.error_message)
        elif self.state.state == ViewState.SUCCESS:
            self.view.hide_loading()
            self.view.show_success(self.state.success_message)
            self.view.show_users(self.state.users)
        elif self.state.state == ViewState.IDLE:
            self.view.hide_loading()
            self.view.show_users(self.state.users)
            
    # IUserModelListener の実装
    def on_user_loaded(self, users: List[User]) -> None:
        """ユーザーロード完了"""
        self._update_state(ViewState.IDLE, users=users)
        
    def on_user_saved(self, user: User) -> None:
        """ユーザー保存完了"""
        self._update_state(ViewState.SUCCESS, 
                         success_message="ユーザーを保存しました")
        # フォームデータクリア
        self.state.form_data = {}
        # 一覧再読み込み
        asyncio.create_task(self.model.load_users())
        
    def on_error(self, error: str) -> None:
        """エラー発生"""
        self._update_state(ViewState.ERROR, error_message=error)
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
