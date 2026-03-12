# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 13)：Users 機能 CRUD

---

## Users 機能の残りを実装

### 1️⃣ ルーティングに `edit` / `update` / `destroy` を追加

```ruby
resources :users, only: %i[new create edit update destroy]
```

---

### コントローラーに `edit` / `update` / `destroy` を追加

```ruby
class UsersController < ApplicationController
  skip_before_action :require_login, only: [ :new, :create ]
  before_action :set_user, only: %i[edit update destroy]

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      redirect_to root_path, notice: "登録が完了しました"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @user.update(user_params)
      redirect_to root_path, notice: "ユーザー情報を更新しました"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @user.destroy
    redirect_to root_path, notice: "ユーザーを削除しました"
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
  end

  def set_user
    @user = current_user
  end
end
```

---

## DB の保存先を Neon に移行

Render の PostgreSQL の無料枠には期限があるため、Neon に移行した。

### 1️⃣ Neon の `Connection string` を取得

- GitHub アカウントでサインイン
- 初期設定（PostgreSQL のバージョンは最新を指定）
- Project dashboard から `Connection string` を開き、文字列をコピー

### 2️⃣ Render の `DATABASE_URL` に文字列をペースト

- Render の **Dashboard > 当該 Project > MANAGE > Environment** を開く
- **Environment Variables** の `DATABASE_URL` を編集し、Neon で取得した `Connection string` をペースト

⬇️

「Save」すると自動でデプロイが始まるので、エラーにならなければ成功

⚠️ *新しい DB を使うため既存データは引き継がれないので注意*
