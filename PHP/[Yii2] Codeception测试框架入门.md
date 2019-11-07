# [Yii2] Codeception测试框架入门

## **debug sucks, test rocks!**  
在经历了很多项目的bug过后，大家都应该意识到一个问题：测试的重要性。

我们不能太信任自己的代码，谁也不能保证自己的代码不出问题  
你还在使用`echo`， `print_r`， `var_dump`， 一步步的调试代码吗，不知道自己的代码运行的结果是什么  

当你拿到一个需求或一个用户故事的时候，你就可以开始构建你的测试了  
测试不仅仅可以保证我们的代码的质量，或看是否按照预期的结果正确运行，更可以让代码水平提高

当你开始写测试代码的时候，更多的时候也是对你自己的代码做一次自我审查，很多时候你会发现自己写的代码是不具备可测试性的，比如封装的不够好，耦合度太高，抽象做的不够好  

赶紧开始测试，倒逼提升你的代码水平和代码质量

## 什么时候或项目需要测试
测试有的时候被当做一个麻烦的事情，好像花了很多时间做了一些回报不高的事  
其实我们是要灵活的来选择是否进行测试，或者不同的情景下，测试的重要性是不一样的

测试的重要性不是很高：
* 项目不大或很简单
* 新频率很低
* 或是一次性项目
* 出bug的没有什么代价

测试的重要性高：
* 项目已经很大且复杂。
* 项目需求开始变得复杂起来。项目不断发展。
* 项目历时很长。
* 失败的代价非常高。

> 参考：https://www.yiiframework.com/doc/guide/2.0/zh-cn/test-overview

## Codeception
Codeception本身是一非常强大的PHP测试框架，它不仅可以集成到各种项目当中，也被很多PHP框架所使用，比如 `Yii2`, `Laravel`, `Symfony`，功能很全，覆盖面很广，并且本身也在持续更新，不断增加新的功能

Yii 2 官方兼容 [`Codeception`](https://github.com/Codeception/Codeception) 测试框架，
你可以创建以下类型的测试：

## 测试类型
Codeception把测试分为3种类型，其中验收测试和功能测试很近视，区别是是否开启一个真实的浏览器

- [单元测试](test-unit.md) - 验证一个独立的代码单元是否按照期望的方式运行；
- [功能测试](test-functional.md) - 在浏览器模拟器中以用户视角来验证期望的场景是否发生
- [验收测试](test-acceptance.md) - 在真实的浏览器中以用户视角验证期望的场景是否发生。

## 安装
Yii 为包括 [`yii2-basic`](https://github.com/yiisoft/yii2/tree/master/apps/basic) 和
[`yii2-advanced`](https://github.com/yiisoft/yii2/tree/master/apps/advanced) 
在内的应用模板脚手架提供全部三种类型的即用测试套件。

Codeception 预装了基本和高级项目模板。
如果您没有使用这些模板中的一个，则可以安装 Codeception
通过输入以下控制台命令：

```
composer require codeception/codeception
composer require codeception/specify
composer require codeception/verify
```

> 我们一般安装了Yii后，codeception都是已经装好了的

## 配置文件
第一步是先配置，Codeception一共有4个配置文件，分别是：

* codeception.yml - 全局配置
* unit.suite.yml - 单元测试配置
* functional.suite.yml - 功能测试配置
* acceptance.suite.yml - 验收测试配置

关于配置的详细情况直接在官网查看：
https://codeception.com/docs/modules/Yii2

> Yii2 默认写好这些些配置文件，我们可以根据具体情况后续修改

## 测试数据库 
当进行测试的时候，有时我们需要对数据进行新增或修改，我们不能直接在生产环境或开发环境，需要建立一个新的测试数据库，保持和生产环境一样的数据结构

具体的数据你可以选择伪造或导入生产数据（你的生产数据没有隐私保护或允许的情况下）
下面我会讲如何用`Faker`和`Fixture`来构造数据

在basic版本中，`config/test_db.php` 直接改为测试数据库的配置， `config/console.php`默认用的是`db.php`，当用 fixture 时候我们可以改为用`test_db.php`

## Faker和Fixture
Yii2 的 Fixture 是框架自身提供的一个测试数据制造管理工具，这个工具的作用就是构造可复用的伪数据，来满足测试

这个工具的好处是可以通过一些简单的命令实现：
* 数据的自动装载 yii fixture/load <fixture_name>
* 数据的自动删除 yii fixture/unload <fixture_name>

> 官方文档 https://www.yiiframework.com/doc/guide/2.0/zh-cn/test-fixtures#shi-yong-yii-fixture-lai-guan-li-fixtures

其中，被装载的数据，你可以自己造（手敲），或用Faker来生成

Faker是一个开源的假数据制造包，我们可以用这个轻松的造各式各样的数据

https://github.com/fzaninotto/Faker  
https://github.com/yiisoft/yii2-faker/blob/master/docs/guide/basic-usage.md

Yii2框架是默认安装Faker的，直接用就行了，下面是官网给的模板文件示例
```
// users.php file under the template path (by default @tests/unit/templates/fixtures)
/**
 * @var $faker \Faker\Generator
 * @var $index integer
 */
return [
    'name' => $faker->firstName,
    'phone' => $faker->phoneNumber,
    'city' => $faker->city,
    'password' => Yii::$app->getSecurity()->generatePasswordHash('password_' . $index),
    'auth_key' => Yii::$app->getSecurity()->generateRandomString(),
    'intro' => $faker->sentence(7, true),  // generate a sentence with 7 words
];
```

> 上面是没有id的，但是有些时候你需要做一些关联关系，你可以用`$index`变量如
```
return [
    'id' => $faker->$index+1,
];
```

写好的faker的模板文件后，运行`yii fixture/generate users`就可以在你的fixture文件新建一个data文件夹，并在里面生成一个users.php的假数据数组文件

当有了这些假数据够，你就可以用fixture把数据加载到测试数据库并开始测试，测试完毕后在卸载即可

## 单元测试
Codeception的单元测试是基于PHPUnit做的，所以你如熟悉PHPUnit的话，这个很好上手，断言都是一样的

先建立测试文件
```
php vendor/bin/codecept generate:test unit Example
```
这个命令能直接生成一个`ExampleTest.php`的测试文件，位于`tests/unit`的地方
```
class ExampleTest extends \Codeception\Test\Unit
{
    /**
     * @var \UnitTester
     */
    protected $tester;

    // 在测试之前执行，比如加载一些类
    protected function _before()
    {
    }

    // 测试结束后执行
    protected function _after()
    {
    }

    // 测试方法，以test开头
    public function testMe()
    {
        // 写一些断言 assertXXX
    }
}
```
> PHPUnit 的文档 https://phpunit.de/manual/6.5/en/appendixes.assertions.html

当需要用数据库相关的方法比如`seeInDatabase()`，你需要先加载DB Module，在Yii2中，可以不加载DB，而是Yii2提供的ORM方法`seeRecored()`，在https://codeception.com/docs/modules/Yii2中可以看到

## 功能测试
功能测试是模拟网页用户使用的情况进行测试（并不会真的用到浏览器），是对一个功能的整体进行测试

先建立测试文件
```
php vendor/bin/codecept g:cest functional Login
```
这里会在`tests\functional`下面生成一个`LoginCest.php`文件
> Cest means Codeception test

然后编写你的测试代码  
定位到一个页面
```
$I->amOnPage('/login');
```
进行一些交互
```
$I->click('Logout');
// click link inside .nav element
$I->click('Logout', '.nav');
// click by CSS
$I->click('a.logout');
// click with strict locator
$I->click(['class' => 'logout']);
```
填表
```
$I->submitForm('form#login', ['name' => 'john', 'password' => '123456']);
// alternatively
$I->fillField('#login input[name=name]', 'john');
$I->fillField('#login input[name=password]', '123456');
$I->click('Submit', '#login');
```
断言
```
$I->see('Welcome, john');
$I->see('Logged in successfully', '.notice');
$I->seeCurrentUrlEquals('/profile/john');
```

Codeception采用语义化的表述方式，看代码基本就可以知道要看什么事情，这一部分的工作应该由测试人员编写，而不是开发人员编写

## 验收测试
验收测试其实就是功能测试的浏览器版本，就是把测试过程，真正的在浏览器上面运行，其测试文件写法和功能测试一样

需要用的一些额外的工具(Selenium)

## 总结
随着未来程序发展越来越复杂，体量越大，总有疏忽的地方，程序需要更专业的标准（甚至工业化严格标准），测试显的越来越重要，让我们的程序能更好、更快的运行。

所以对于一个程序员来说，懂得测试，会用测试基本是必备技能了。
