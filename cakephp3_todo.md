CakePHP3でTODOアプリを作ってみます  
作ったもの: https://github.com/kshiva1126/cakephp3_todo.git

## setup

Gitクローンからコンテナの立ち上げまで

```
$ git clone https://github.com/kshiva1126/cakephp3_docker.git
$ cd cakephp3_docker
$ docker-compose build
$ docker-compose up -d
$ docker-compose exec app composer install
```

Databaseの設定

config/app.php の下記4項目を修正します

``` php
    'Datasources' => [
        'default' => [
            ...
            'host' => 'db',
            'username' => 'user',
            'password' => 'password',
            'database' => 'cake_db',
            ...
        ]
    ]
```

http://localhost:3000 でページが確認できます


## migration

下記コマンドでマイグレーションファイルを生成します  
 `config/Migrations/` に出力されます

```
bin/cake bake migration CreateTasks name:string decription:text done:boolean created modified
```

ちなみに `bin/cake migrations create` でも作成できるらしいけど今回はスキップ

``` php
<?php
use Migrations\AbstractMigration;

class CreateTasks extends AbstractMigration
{
    public function change()
    {
        $table = $this->table('tasks');
        $table->addColumn('name', 'string', [
            'default' => null,
            'limit' => 255,
            'null' => false,
        ]);
        $table->addColumn('decription', 'text', [
            'default' => null,
            'null' => false,
        ]);
        $table->addColumn('done', 'boolean', [
            'default' => false, // nullからfalseに変更した
            'null' => false,
        ]);
        $table->addColumn('created', 'datetime', [
            'default' => null,
            'null' => false,
        ]);
        $table->addColumn('modified', 'datetime', [
            'default' => null,
            'null' => false,
        ]);
        $table->create();
    }
}
```

マイグレートします

```
bin/cake migrations migrate
```

```
mysql> desc tasks;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| name       | varchar(255) | NO   |     | NULL    |                |
| decription | text         | NO   |     | NULL    |                |
| done       | tinyint(1)   | NO   |     | 0       |                |
| created    | datetime     | NO   |     | NULL    |                |
| modified   | datetime     | NO   |     | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

マイグレーションファイルにはなかった `id` は自動的に追加されるようです

## bake

CakePHPといえば、 `bake` ですね！！！  
`bake` にはたくさんのサブコマンドがありますが、今回は `all` を利用してMVCすべてのスケルトンを生成します  
先程作成した `tasks` を指定します

```
bin/cake bake all tasks
```

ここで http://localhost:3000/tasks にアクセスしてみます

tasks_top.png

もうそれっぽい画面が表示されました

左カラムの `New Task` から追加画面にいけます

tasks_add.png

追加後です

tasks_top_added.png

## customize

このまま終了だとかなり味気ないので、ちょっとしたカスタマイズを加えます  
現状、`Done` を 更新するためには編集画面で行うしかありません

tasks_edit.png

これを、TOP画面からでも更新できるようにします

まずViewを編集します  
`Done` を Checkbox で表示させます  
参考: https://book.cakephp.org/3/ja/views/helpers/form.html#checkbox-radio-select-options

src/Template/Tasks/index.ctp

``` html
<tbody>
    <?php foreach ($tasks as $task): ?>
    <tr>
        <td><?= $this->Number->format($task->id) ?></td>
        <td><?= h($task->name) ?></td>
        <!-- <td><?= h($task->done) ?></td> -->
        <!-- Checkboxで表示させるように修正 -->
        <td><?= $this->Form->checkbox('done', [
            'value' => h($task->done),
            'checked' => h($task->done),
            'data-id' => $this->Number->format($task->id),
            'hiddenField' => false,
        ]) ?></td>
        <td><?= h($task->created) ?></td>
        <td><?= h($task->modified) ?></td>
        .. 省略 ..
</tbody>
```

jQueryのAjaxを使いたいので、CDNで読み込ませます

src/Template/Layout/default.ctp

``` html
<head>
    <?= $this->Html->charset() ?>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>
        <?= $cakeDescription ?>:
        <?= $this->fetch('title') ?>
    </title>
    .. 省略 ..

    <!-- jQuery読み込み -->
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
</head>
```

`Done` を更新するためにAjaxでPOSTする処理を追加します

src/Template/Tasks/index.ctp

``` html
<script>
$(function () {
    const csrfToken = <?= json_encode($this->request->getParam("_csrfToken")) ?>;
    $("input[name='done']").on("change", function (event) {
        const data = {
            id: $(this).data('id'),
            done: $(this).prop('checked') ? 1 : 0,
        };
        $.ajax({
            type: 'POST',
            dataType: "json",
            url: '/tasks/changeDone',
            headers: {
                'X-CSRF-Token': csrfToken
            },
            data,
        })
        .fail(function (err) {
            console.log(err)
        });
    });
});
</script>
```

Controllerに更新処理を追加します  
`TasksController` に新たに `changeDone()` メソッドを追加することで `/tasks/changeDone` でアクセスできるようになります

src/Controller/TasksController

``` php
public function changeDone()
{
    // postのみ許可する
    $this->request->allowMethod('post');
    $id = (int) $this->request->getData('id');
    $done = (int) $this->request->getData('done');

    // idに合致するTaskを取得
    $task = $this->Tasks->get($id);
    $task->done = $done;
    $this->Tasks->save($task);

    exit;
}
```

(CakePHP的にこの書き方で正しいのかはわからない... (特に最後の `exit;` ))

これでTOP画面から`Done` の更新が行えるようになりました
