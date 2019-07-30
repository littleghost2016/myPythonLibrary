### Flask-SQLAlchemy
1.初始化和设置数据库配置信息
    * 使用flask_sqlalchemy中的SQLAlchemy进行初始化:

        from Flask-SQLAlchemy import SQLAlchemy
        app = Flask(__name__)
        db = SQLAlchemy(app)

2.设置配置信息：在`config.py`文件中添加如下配置信息

    # dialect+driver://username:password@host:port/database
    DIALECT = 'mysql'
    DRIVER = 'pymysql'
    USERNAME = 'root'
    PASSWORD = 'root'
    HOST = '127.0.0.1'
    PORT = '3306'
    DATABASE = 'db_demo1'
    
    SQLALCHEMY_DATABASE_URI = '{}+{}://{}:{}@{}:{}/{}?charset=utf8'.format(DIALECT, DRIVER, USERNAME, PASSWORD, HOST, PORT, DATABASE)
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    
    DEBUG = True

3.在主`app`文件中添加配置文件

    app = Flask(__name__)
    app.config.from_object(config)
    db = SQLAlchemy(app)

4.测试

    db.create_all()
    如果没有报错error，说明配置没有问题
    
    若在`windows`平台出现
    C:\ProgramData\Anaconda3\lib\site-packages\pymysql\cursors.py:166: Warning: (1366, "Incorrect string value: '\\xD6\\xD0\\xB9\\xFA\\xB1\\xEA...' for column 'VARIABLE_VALUE' at row 481")
      result = self._query(query)
    C:\ProgramData\Anaconda3\lib\site-packages\pymysql\cursors.py:166: Warning: (1287, "'@@tx_isolation' is deprecated and will be removed in a future release. Please use '@@transaction_isolation' instead")
      result = self._query(query)
    等warning，目前为找到根本解决办法，应该是Windows系统编码与Python、Mysql不同导致，没有重大影响，转至`Linux`平台warning消失。

### 使用Flask-SQLAlchemy创建模型与表的映射
1.模型需要继承自`db.Model`，然后需要映射到表中的属性，必须写成`db.Column`的数据类型。
2.数据类型：
    * `db.integer`代表整形
    * `db.String()`代表`varchar`，但要传入一个数作为最大长度
    * `db.Text`代表`text`
3.其他参数
    * `primary_key`：代表将这个字段设置为主键
    * `autoincrement`：代表这个主键是自增长的
    * `nullable`：代表这个字段是否可以为空，默认可以为空，可手动设置为`False`即在数据库中不可为空
4.最后使用`db.create_all()`将模型真正创建到数据库当中

### Flask-SQLAlchemy数据的增删改查
1.增加

    # 增加
    article1 = Article(title='aaa',content=time.ctime())
    db.session.add(article1)
    # 事务
    db.session.commit()

2.查找

    # 查找（有可能查出来是个列表，不能直接像对单个对象一样操作）
    result = Article.query.filter(Article.title == 'aaa').all()
    for i in result:
        print(i.id, i.title)

3.更改
    
    # 更改
    # 1.先把要更改的数据查找出来
    article1 = Article.query.filter(Article.title == 'aaa').first()
    # 2.修改想要修改的地方
    article1.title = 'new title'
    # 3.进行事务的提交
    db.session.commit()

4.删除

    # 删除
    # 1.先把要删除的数据查找出来
    article1 = Article.query.filter(Article.title == 'aaa').first()
    # 2.删除操作
    db.session.delete(article1)
    # 3.进行事务的提交
    db.session.commit()

### Flask-SQLAlchemy外键及其关系
1.外键

    class User(db.Model):
        __tablename__ = 'user'
        id = db.Column(db.Integer,primary_key=True,autoincrement=True)
        username = db.Column(db.String(100),nullable=False)
    
    class Article(db.Model):
        __tablename__ = 'article'
        id = db.Column(db.Integer, primary_key=True, autoincrement=True)
        title = db.Column(db.String(100), nullable=False)
        content = db.Column(db.Text, nullable=False)
        author_id = db.Column(db.Integer,db.ForeignKey('user.id'))
        author = db.relationship('User',backref=db.backref('articles'))

2.`author = db.relationship('User',backref=db.backref('articles'))`的解释
    * 给`Article`这个模型添加一个`author`属性，可以访问这篇文章的作者的数据，像访问普通模型一样
    * `backref`是定义反向引用，可以通过`User.articles`这个模型访问这个模型所写的所有文章


### Flask-Script的介绍与安装
1.作用：可以通过命令行的形式来操作Flask，例如通过一个命令跑一个开发版本的服务器、初始化数据库、数据库迁移
2.安装：`pip install flask-script`
3.如果直接在主`manage.py`中写命令，在终端只需要输入`python manage.py command_name`
4.如果把一些命令集中在一个文件中，则在终端就需要输入一个父命令，如`python manage.py db init`
5.
    子：
    
    # db_script.py
    from flask_script import Manager
    
    db_manager = Manager()
    
    @db_manager.command
    def init():
        print('初始化数据库完成')
    
    @db_manager.command
    def migrate():
        print('数据库迁移完成')
    
    父：
    
    # manage.py
    
    from flask_script import Manager
    from test import app
    from db_script import db_manager
    
    manage = Manager(app)
    
    @manage.command
    def runserver():
        print('1111111111')
    
    manage.add_command('db', db_manager)
    
    if __name__ == '__main__':
        manage.run()


### 分开`models`以及解决循环引用
1.分开`models`的目的：为了让代码更加方便地管理
2.解决循环引用：把`db`单独放入一个文件中

    # extension.py
    from flask_sqlalchemy import SQLAlchemy
    db = SQLAlchemy()
    
    # models.py
    from extension import db
    class Article(db.Model):
        ......
    
    # 主文件.py
    from extension import db
    from models import Article


    db.init_app(app)


### Flask-migrate
1.Flask独有的特点

    db.init_app(app)
    with app.app_context():
    db.create_all()

2.介绍：
    使用`db.create_all()`在后期修改字段时，不会自动映射到数据库，必须删除表并且运行`db.create_all()`才会重新映射到数据库，且原来表中的数据也被删除。Flask-migrate被设计来解决这个问题，可以在每次修改模型后，将修改的字段映射到数据库中。

3.使用`flask_migrate`必须借助`flask_script`这个包的`Migratecommend`中包含所有和数据库有关的命令。

4.`flask_migrate`相关命令：
    `manage.py`

    from db_script import db_manager
    from flask_migrate import Migrate, MigrateCommand
    from extension import db
    from models import Article, User
    
    manager = Manager(app)
    
    # 创建环境-(init)->模型-(migrate)->迁移文件-(upgrade)->表
    # 1.要使用flask_migrate，必须绑定app和db
    migrate = Migrate(app, db)
    # 2.把MigrateCommand命令添加到manager中
    manager.add_command('db', MigrateCommand)
    # 此时就可以删除主app文件中的
    # with app.app_context():
    #     db.create_all()
    # 3.
    # python manage.py db upgrade
    
    # 若后期修改字段，需要执行migrate和upgrade命令
    # init初始化一个迁移脚本的环境，只需执行一次。
    # migrate将模型生成迁移文件，只要模型更改就要执行一次这个文件。
    # upgrade将迁移文件真正的映射到数据库中，每次运行`migrate`命令后都要执行一次`upgrade`命令
    
    if __name__ == '__main__':
    manager.run()

5.注意点：需要将所想要映射到数据库中的模型，都要导入到`manage.py`文件中，如果没有导入就不会映射到数据库中。

### 使用session
1.session的操作方法
    * 使用`session`从`flask`中导入`session`，以后所有对session的操作都是通过这个变量来的
    * 使用`session`需要设置`SECRET_KEY`，用来作为加密用的，并且每次这个key在服务器启动后都变化的话，那么之前的session就不能通过当前的`SECRET_KEY`来操作了
    * session的操作和字典相同
    * sessi的增删改查中，删除有`pop()`方法，或者`clear()`清除所有键值对

### get请求和post请求
get请求：
    - 使用场景：如果只向服务器获取数据，并且没有对服务器产生任何影响
    * 传参：传参放在URL中，并且通过问号的形式来指定key和value
post请求：
        * 使用场景：如果要对服务器产生影响
            * 传参：通过`from data`的形式发送给服务器

### get和post请求获取参数
1.get请求通过`request.args.get()`，post请求通过`request.form.get()`
2.post请求在模板中要注意：
    - `input标签`中要写`name`标识这个`value`的key，方便后台获取
    - 写form的时候，要指定`method=post`，并且要指定`action`的`url_for()`
3.示例代码
    
    <form action="{{ url_for('login') }}" method="post">
        <table>
            <tbody>
                <tr>
                    <td>用户名</td>
                    <td><input type="text" placeholder="请输入用户名" name="username"></td>
                </tr>
                <tr>
                    <td>密码</td>
                    <td><input type="text" placeholder="请输入密码" name="password"></td>
                </tr>
                <tr>
                    <td></td>
                    <td><input type="submit" value="提交"></td>
                </tr>
            </tbody>
        </table>
    </form>

### 钩子函数
1.before_request函数
    * `before_request`只是一个装饰器，她可以把需要设置为钩子函数的代码放到请求、视图函数之前执行
2.