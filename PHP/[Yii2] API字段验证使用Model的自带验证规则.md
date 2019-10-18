# [Yii2] API字段验证使用Model的自带验证规则

最近写API，想做一个可配置的接口验证，在Github上面看了一些验证器，大多都比较复杂，想着能不能用框架自带的验证器来做，尝试了后发现不是很好，但是还是可以满足大部分验证了

## Yii2的内置验证与Laravel的内置验证

你可能会有个疑问，难道Yii2框架没提供API请求验证吗，其实Yii2框架提供的验证是作用于Model层的，没有单独提供针对Request的验证器，下面用官方文档来比较一下

* Yii2
```
$model = new \app\models\ContactForm();

// 根据用户的输入填充到模型的属性中
$model->load(\Yii::$app->request->post());
// 等效于下面这样：
// $model->attributes = \Yii::$app->request->post('ContactForm');

if ($model->validate()) {
    // 所有输入通过验证
} else {
    // 验证失败: $errors 是一个包含错误信息的数组
    $errors = $model->errors;
}
```
在Yii2中，想要验证用户提交的数据，需要先建一个Model，然后在Model中的rules()方法中定义字段的验证规则，Model->validate()才会触发验证
```
public function rules()
{
    return [
        // name，email，subject 和 body 特性是 `require`（必填）的
        [['name', 'email', 'subject', 'body'], 'required'],

        // email 特性必须是一个有效的 email 地址
        ['email', 'email'],
    ];
}
```

* Laravel
```
/**
 * 保存一篇新的博客文章。
 *
 * @param  Request  $request
 * @return Response
 */
public function store(Request $request)
{
    $validatedData = $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ]);

    // 文章内容是符合规则的，存入数据库...
}
```
Laravel验证是在Request上面，只要是请求就可以验证其中的数据，跟Model没有直接的绑定关系

## 新建组件ApiModel

为了能使用Yii2的内置验证器，但是我并不想给每一个API请求都建一个Model，所以先建一个公共的ApiModel，当有请求过来的时候，自动把API的输入配置参数建一个Model

```
$apiConfig = [
    'member' => [
        'create' => [
            [['name'], 'string'],
            [['name', 'age', 'mobile', 'email'], 'required'],
            [['email'], 'email'],
            [['age', 'mobile'], 'number'],
        ],
        'list' => [
            // ...
        ],
    ],
    // ...
]
```

比如这个 member/create API, 我们把验证规则按Model中rules()方法的方式先写好，然后用name, age, mobile, email 建一个`ApiModel`，下面的ApiModel的代码

```
<?php

namespace frontend\components\api;

use yii;
use yii\base\Model;

class ApiModel extends Model
{
	public $apiAttributesName = [];
	
	public $apiAttributes = [];
	
	public $rules = [];
	
	public function __set($name, $value)
	{
		if (in_array($name, $this->apiAttributesName)) {
			$this->apiAttributes[$name] = $value;
		} else {
			parent::__set($name, $value);
		}
	}
	
	public function __get($name)
	{
		if (in_array($name, $this->apiAttributesName)) {
			return $this->apiAttributes[$name];
		} else {
			return null;
		}
	}
	
	public function attributes()
	{
		if (count($this->apiAttributesName) > 0) {
			return $this->apiAttributesName;
		} else {
			return parent::attributes();
		}
	}
	
	/**
	 * example: ['name', 'age', 'gender']
	 * @param array $names
	 */
	public function setApiAttributesName($names)
	{
		$this->apiAttributesName = $names;
	}
	
	/**
	 * example: [[['name', 'age'], 'required'], [['gender'], 'string']]
	 * @param array $rules
	 */
	public function setRules($rules)
	{
		if (self::validateRuleFormat($rules)) {
			$this->rules = $rules;
		}
	}
	
	public function rules()
	{
		return $this->rules;
	}
	
	public static function validateRuleFormat($rules)
	{
		// TODO
		return true;
	}
}
```

# 建立ApiBehavior使用ApiModel

我这边使用了一个`ApiBehavior`，这样的好处是不会影响到其他不是Api的`Controller`，当你用到这个Behavior的时候才会触发这些验证


```
<?php

namespace frontend\components\api;

use yii;
use ArrayObject;
use yii\base\ActionFilter;
use yii\base\InvalidConfigException;
use yii\validators\Validator;

class ApiBehavior extends ActionFilter
{	
	public $apiModel;
	
	public function beforeAction($action)
	{
		$post = Yii::$app->request->post();
		
		$this->apiModel = new ApiModel();
		
        // 拿到当前API配置的验证规则
		$apiRules = $this->getApiRules($action->controller->id, $action->id);
		if (empty($apiRules)) {
			return true;
		}
		// 把验证规则写入Model
		$this->apiModel->setRules($apiRules);

        // 从配置中取出当前API的所有字段
		$apiAttributes = $this->getApiAttributes($apiRules);
        // 设置Model的字段名称
		$this->apiModel->setApiAttributesName($apiAttributes);
        // 把Request中的post的值写入到对应的字段
		$this->apiModel->setAttributes($post, false);
        // 验证，不通过则返回错误
        // DataValidateException是封装好的错误返回
		if (!$this->apiModel->validate()) {
			throw new DataValidateException('the post params not validated', $this->apiModel->getErrors());
		}
		
		return true;
	}
	
	protected function getApiRules($controller, $action)
	{
        // 读取到上面的 $apiConfig
		$hashList = Yii::$app->params['api_hash'];
		
		if (!isset($hashList[$controller])) {
			return [];
		}
		
		if (!isset($hashList[$controller][$action])) {
			return [];
		}
		
		return $hashList[$controller][$action];
	}
	
	protected function getApiAttributes($rules)
	{
		$apiAttributes = [];
		
		foreach ($rules as $rule) {
			if (is_array($rule) && isset($rule[0], $rule[1])) {
				$attributes = (array) $rule[0];
				$apiAttributes = array_merge($apiAttributes, $attributes);
			}
		}
		
		$apiAttributes = array_unique($apiAttributes);
		sort($apiAttributes);
		
		return $apiAttributes;
	}

}
```

## 自定义错误返回

我自定义了一个`ApiExcption`作为基类，继承与Yii2的`HttpException`，并做了一些修改，验证不通过则会返回如下的结果
上面的`DataValidateException`是`ApiExcption`的子类

```
{
  "name": "OK (#200)",
  "statusCode": 200,
  "code": 40001,
  "message": "the post params not validated",
  "hint": {
    "name": [
      "Name不能为空。"
    ],
    "mobile": [
      "Mobile不能为空。"
    ],
    "email": [
      "Email不是有效的邮箱地址。"
    ],
    "age": [
      "Age必须是一个数字。"
    ]
  }
}
```

## 总结

Yii2的内置验证不够灵活，验证组件跟Model进行了硬绑定

灵活的验证组件应该能被多个数据接收方所使用