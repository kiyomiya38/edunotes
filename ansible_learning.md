# Ansible 学習ガイド（コード読解のための基礎＋実例）

Ansible のプレイブック/ロールを読むときに必要な概念と、よく使う書き方の実例をまとめました。初心者向けに、何を見れば流れが理解できるかをコード例付きで整理しています。

---

## 1. 基本構造を押さえる
- インベントリ: 接続先ホストやグループを定義。`hosts: web` のようにグループ名で指定。
- プレイブック: どのホストにどのロール/タスクを適用するかを記述する YAML。
- ロール: 再利用単位のまとまり。`roles/web/tasks/main.yml`, `templates/`, `files/`, `vars/`, `defaults/`, `handlers/` など。
- 変数の優先順位: コマンドライン > play vars > host_vars > group_vars > role vars > role defaults。

例（最小のプレイブック）:
```yaml
- hosts: web
  become: true
  roles:
    - web
```

---

## 2. 変数とテンプレート (Jinja2)
- 変数参照: `{{ var_name }}`。辞書なら `{{ item.key }}` / `{{ item.value }}`。
- フィルタ: `| default('value')`, `| to_nice_yaml`, `| json_query()` などで整形/抽出。
- 条件付き埋め込み: `{% if condition %} ... {% endif %}` を templates/*.j2 で使用。
- 変数の定義場所: `group_vars/all.yml`（共通）、`host_vars/<hostname>.yml`（個別）、ロールの `defaults/main.yml`（初期値）、`vars/main.yml`（強めの優先度）。

例（テンプレート置換）:
```jinja2
connection_string = postgresql://{{ db_user }}:{{ db_password }}@{{ db_host }}/{{ db_name }}
log_level = {{ app_log_level | default("info") }}
```

---

## 3. タスクとモジュール
- よく使うモジュール: `apt`/`yum`/`package`, `service`, `template`, `copy`, `unarchive`, `user`, `file`, `git`, `lineinfile`, `command`/`shell`（必要最小限で）。
- `state` パラメータ: `present`/`absent`/`started`/`stopped` などで「どうしたいか」を明示。
- ループ: `loop: ["a", "b"]` や `with_items`。`loop_control` で label を整える。
- 条件: `when:` で条件付き実行。例: `when: ansible_os_family == 'Debian'`。

例（パッケージとサービス）:
```yaml
- name: install nginx
  apt:
    name: nginx
    state: present
  when: ansible_os_family == 'Debian'

- name: enable and start nginx
  service:
    name: nginx
    state: started
    enabled: true
```

例（git pull + systemd 再起動を通知）:
```yaml
- name: fetch app code
  git:
    repo: https://example.com/app.git
    dest: /opt/app
    version: main
  notify: restart app
```

---

## 4. ハンドラーと通知
- 変更があったときだけ実行したい処理を `handlers` に置き、タスク側で `notify` する。
- 読解ポイント: `notify` がどこで呼ばれ、`handlers/main.yml` に何があるかをセットで追う。

例（handler 定義）:
```yaml
# roles/web/handlers/main.yml
- name: restart app
  service:
    name: app
    state: restarted
```

---

## 5. テンプレート/ファイル配布のパターン
- `template`: Jinja2 テンプレートを配置。`owner`/`group`/`mode` を明示。
- `copy`: そのままファイルを送る（バイナリ可）。
- `unarchive`: アーカイブを展開（`remote_src` でリモート展開かどうか指定）。

例（テンプレート配布）:
```yaml
- name: place nginx conf
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
  notify: restart app
```

---

## 6. ループ・条件・register の組み合わせ
- `register`: タスク結果を変数に入れる。`when` と組み合わせて分岐。
- `failed_when` / `changed_when`: 判定を上書きして無用な失敗・変更を避ける。

例（状態に応じてサービス起動）:
```yaml
- name: check service status
  command: systemctl is-active myapp
  register: status
  failed_when: false   # 失敗してもプレイを止めない

- name: start service if inactive
  service:
    name: myapp
    state: started
  when: status.rc != 0
```

例（loop でユーザー作成）:
```yaml
- name: create app users
  user:
    name: "{{ item }}"
    shell: /bin/bash
  loop:
    - alice
    - bob
```

---

## 7. 役割分担とベストプラクティス
- ロールで責務を分ける（web, db, monitoring など）。
- 変数は `defaults/main.yml` に初期値を置き、環境差分は `group_vars`/`host_vars` に寄せる。
- `become: true` が必要なタスクと不要なタスクを分ける。
- 冪等性を意識し、`command`/`shell` よりモジュールを優先。

例（冪等な設定変更: lineinfile）:
```yaml
- name: set timezone
  lineinfile:
    path: /etc/timezone
    line: "Asia/Tokyo"
```

---

## 8. 実行・デバッグの基本コマンド
- インベントリ確認: `ansible-inventory --list` / `--graph`
- 疎通確認: `ansible all -m ping`
- ドライラン: `ansible-playbook site.yml --check --diff`
- 対象絞り込み: `ansible-playbook site.yml -l web-1`
- 一時変数: `-e key=value` で上書き

---

## 9. 変数の解決とデバッグ
- 変数確認: `debug: var=変数名`
- 優先順位の高い場所で上書きされていないかを確認（group_vars/host_vars/vars/defaults）。

例（debug で値確認）:
```yaml
- name: show db host
  debug:
    var: db_host
```

---

## 10. よくある落とし穴
- テンプレートのインデント/スペースずれで構文エラー。
- 変数名のタイプミス（未定義エラー）。
- `become: true` の付け忘れ/付け過ぎで権限が想定外になる。
- `command`/`shell` の多用で冪等性が崩れる。必要なら `creates`/`removes` を指定。

---

## 11. 読むときの手順（おすすめ）
1. `inventory` と `group_vars`/`host_vars` を見て、対象ホストと主要変数を把握。
2. プレイブック（例: `site.yml`）で、どのロールがどのホストに適用されるか確認。
3. 各ロールの `tasks/main.yml` を上から追い、`notify` → `handlers` の流れも確認。
4. `templates/` と `defaults/vars` を見て、設定ファイルにどの変数が入るかを確認。
5. 不明点は `--check --diff` や `-l <host>` で小さく試す。

---

このガイドは「Ansible の読み方」に特化しています。実際のプレイブックやロールと照らし合わせて、分からない表現が出たら公式ドキュメントや `ansible-playbook --check` で小さく検証しながら進めてください。
